## 个人 Tensorflow 使用

#### 图结构分析

- 构建图阶段：数据（Tensor）+ 操作（Operation）
  - 操作函数：`tf.constant()`、`tf.Variable()`
  - 操作对象: `Const`、`Tensor`对象等
- 执行图阶段：使定义好的数据和操作运行起来

#### 图相关操作

- 默认图

  ```
  tf.get_default_graph()
  # 查看会话的所在图属性
  graph=sess.graph
  ```

- 创建图

  ````python
  g = tf.Graph()
  with g.as_default():
  ````

#### 会话

* 实时显示张量实际值

  ```
  tf.InteractiveSession()
  ```

  适合用于`ipython`，配合`eval()`实时显示张量实际值。

* 打印设备使用情况，如：哪部分操作在第几块 GPU / CPU上运行

  ```
  with tf.Session(graph=g, config=tf.ConfigProto(allow_soft_placement=True, log_device_placement=True)) as sess:
  ```

* `op_value,loss_value = sess.run([op,loss],feed_dict={})`

  搭配占位符使用`tf.placeholder(tf.float32)`

#### 张量

* 修改类型

  `ndarray.astype(type)`

  `tf.cast(tensor,dtype)`

  `ndarray.tostring()`：转成二进制形式

* 修改形状

  `ndarray.reshape(shape)`

  `ndarray.resize(shape)`

  静态形状：初始创建张量时的形状，只有在形状没有完全固定下来的情况下才能更新静态形状，`tensor.set_shape(shape)`，优先选择这种方法。

  动态形状：不会改变原始的张量，返回改变后的张量，`new_tensor = tf.reshape(tensor,shape)`，元素个数必须匹配。

#### 变量

* 创建变量，`a = tf.Variable(initial_value=50)`

  变量需要显式初始化，才能运行值，`sess.run(tf.global_variables_initializer())`

* 使用`tf.variable_scope()`修改变量的命名空间，可以使得图结构更加清晰

#### 数据可视化

1. 创建事件文件

   ```
   tf.summary.FileWriter(path,graph=sess.graph)
   ```

2. 收集变量

   ```
   # 收集损失函数和准确率等单值变量
   tf.summary.scalar('在tensorboard中显示的名称', tensor)
   # 收集高维度的变量参数
   tf.summary.histogram('bias', b)
   # 收集输入的图片张量能显示图片
   tf.summary.image(name='',tensor)
   ```

3. 指定迭代次数合并变量

   ```
   merged = tf.summary.merge_all()
   summary = sess.run(merged)
   ```

4. 指定迭代次数将summary对象写入事件文件

   ```
   file_writer.add_summary(summary, step)
   ```

5. 会话运行结束，在指定目录下生成events文件，进入`tensorboard.exe`所在目录，命令行运行（**注意避免路径中多余的斜杠**）

   ```
   tensorboard --logdir=project/summary
   ```

#### 模型参数保存与读取

* 保存

  ```
  saver = tf.train.Saver()
  # 会话内
  saver.save(sess, './model/linear.ckpt')
  ```

* 读取

  ```
  #先定义与训练阶段一样的图
  xxxxxx
  
  with tf.Session(graph=graph) as sess:
  	if os.path.exists('./model/checkpoint'):
  		saver.restore(sess, './model/linear.ckpt')
  		# 之后就可以直接sess.run拿数据了
  ```

#### 解析命令行参数

```python
tf.app.flags.DEFINE_integer('max_step', 100, 'iterations of train')
tf.app.flags.DEFINE_string('model_dir', 'xxx', 'dir of model')
FlAGS = tf.app.flags.FLAGS

def get_command_params():
    print(FlAGS.max_step,FlAGS.model_dir)
```

#### 数据IO操作

1. 构造文件名队列

   ```
   file_queue = tf.train.string_input_producer([路径+文件名])
   file_queue = tf.train.string_input_producer([os.path.join(data_path, file) for file in os.listdir(data_path) if file[-3:] == '.bin'])
   ```

2. 读取与解码

   应保证读取之后所有的样本的**形状**和**类型**统一

   * 文本：一行一个样本

     ```
     读取器：tf.TextLineReader()
     解码：tf.decode_csv()
     ```

   * 图片：一张图片一个样本

     ```
     读取器：tf.WholeFileReader()
     解码：tf.image.decode_jpeg(contents)
     	 tf.image.decode_png(contents)
     	 	return uint8张量 [height.width,channels]
     ```

   * 二进制文件：设定固定字节一个样本

     ```
     读取器：tf.FixedLengthRecordReader(record_bytes)
     解码：tf.decode_raw(value,tf.uint8)
     
     reader = tf.FixedLengthRecordReader(1024)
     key, value = reader.read(file_queue)
     decoded = tf.decode_raw(value, tf.uint8)
     ```

   * TFRecords 文件

     ```
     读取器：tf.TFRecordReader()
     解码：tf.decode_raw(value,tf.unint8)
     
     reader = tf.TFRecordReader()
     key, value = reader.read(file_queue)
     feature = tf.parse_single_example(value, features={
     	'image': tf.FixedLenFeature([], tf.string),
     	'label': tf.FixedLenFeature([], tf.int64)
     })
     image = feature['image']
     label = feature['label']
     image_decoded = tf.decode_raw(image, tf.uint8)
     ```

     >如何制作TFRecords数据集？

     ```python
     with tf.python_io.TFRecordWriter('save_path + cifar10.tfrecords') as writer:
             for i in range(batch_size):
             	# 转成二进制形式
                 image = image_batch[i].tostring()
                 #仅支持 int64 或者 byteslist 或者 float32三种形式
                 label = label_batch[i][0]
                 # 构造TFRecords的基本数据结构（example）
                 example = tf.train.Example(
                 features=tf.train.Features(feature={
                     'image': tf.train.Feature(bytes_list=tf.train.BytesList(value=[image])),
                     'label': tf.train.Feature(int64_list=tf.train.Int64List(value=[label]))
                 }))
                 #序列化便于写入
                 writer.write(example.SerializeToString())
     ```

3. 批处理

   ```
   tf.train.batch([tensor],batch_size=100,num_threads=4,capacity=100)
   ```

4. 线程操作

   ```
   with tf.Session() as sess:
   	coord = tf.train.Coordinator()
   	threads = tf.train.start_queue_runner(sess,coord)
   	image_batch_value, label_batch_value = sess.run(image_batch, label_batch)
   	xxx
   	xxx
   	coord.request_stop()
   	coord.join(threads)
   ```

#### 图片数据

* 数据格式

  * 存储：`uint8` （节省空间）

  * 矩阵计算：`float32` （提高精精度）

  * 处理尺寸

    ```
    image = tf.image.decode_png(value)
    # 仅改变size，没有指定通道数
    image = tf.image.resize_images(images,[32,32])
    image.set_shape(shape=[200, 200, 3])
    ```

#### 常用API

* c = tf.matmul(a,b)

* tf.slice(input_, begin, size, name=None)，对tensor进行切片操作

  ```
  # 'input' is [[[1, 1, 1], [2, 2, 2]],
  #             [[3, 3, 3], [4, 4, 4]],
  #             [[5, 5, 5], [6, 6, 6]]]
  tf.slice(input, [1, 0, 0], [1, 1, 3]) ==> [[[3, 3, 3]]]
  tf.slice(input, [1, 0, 0], [1, 2, 3]) ==> [[[3, 3, 3],
  											[4, 4, 4]]]
  tf.slice(input, [1, 0, 0], [2, 1, 3]) ==> [[[3, 3, 3]],
  											[[5, 5, 5]]]
  ```

* tf.split(value, num_or_size_splits, axis=0, num=None, name="split")，沿着某一维度将tensor分离为num_split tensors

  ```
  # 'value' is a tensor with shape [5, 30]
  # Split 'value' into 3 tensors with sizes [4, 15, 11] along dimension 1
  split0, split1, split2 = tf.split(value, [4, 15, 11], 1)
  tf.shape(split0) ==> [5, 4]
  tf.shape(split1) ==> [5, 15]
  tf.shape(split2) ==> [5, 11]
  # Split 'value' into 3 tensors along dimension 1
  split0, split1, split2 = tf.split(value, num_or_size_splits=3, axis=1)
  tf.shape(split0) ==> [5, 10]
  ```

* tf.concat(values, axis, name="concat")

  ```
  t1 = [[1, 2, 3], [4, 5, 6]]
  t2 = [[7, 8, 9], [10, 11, 12]]
  tf.concat([t1, t2], 0) ==> [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10, 11, 12]]
  tf.concat([t1, t2], 1) ==> [[1, 2, 3, 7, 8, 9], [4, 5, 6, 10, 11, 12]]
  ```

* tf.transpose(a, perm=None, name="transpose")

  ```
  # 'x' is [[1 2 3]
  #         [4 5 6]]
  tf.transpose(x) ==> [[1 4]
  					 [2 5]
  					 [3 6]]
  ------------------------------------               
    # 'x' is   [[[1  2  3]
    #            [4  5  6]]
    #           [[7  8  9]
    #            [10 11 12]]]
    # Take the transpose of the matrices in dimension-0
    tf.transpose(x, perm=[0, 2, 1]) ==> [[[1  4]
                                          [2  5]
                                          [3  6]]
  
                                         [[7 10]
                                          [8 11]
                                          [9 12]]]
  ```

* tf.one_hot(indices, depth, on_value=None, off_value=None,axis=None, dtype=None, name=None):

  ```
  indices = [[0, 2], [1, -1]]
  depth = 3
  on_value = 1.0
  off_value = 0.0
  axis = -1
  # then output is  [2*2*3]
  output =[
  		 [1.0, 0.0, 0.0]  // one_hot(0)
  		 [0.0, 0.0, 1.0]  // one_hot(2)
  		][
  		 [0.0, 1.0, 0.0]  // one_hot(1)
  		 [0.0, 0.0, 0.0]  // one_hot(-1)
  		]
  ```

* 规约计算

  * tf.reduce_sum(input_tensor,axis=None,keep_dims=False)

    ```
    # 'x' is [[1, 1, 1]
    #         [1, 1, 1]]
    tf.reduce_sum(x) ==> 6
    tf.reduce_sum(x, 0) ==> [2, 2, 2]
    tf.reduce_sum(x, 1) ==> [3, 3]
    tf.reduce_sum(x, 1, keep_dims=True) ==> [[3], [3]]
    tf.reduce_sum(x, [0, 1]) ==> 6
    ```

  * tf.reduce_all(input_sensor,axis=None,keep_dims=False)

    ```
    # 'x' is [[True,  True]
    #         [False, False]]
    tf.reduce_all(x) ==> False
    tf.reduce_all(x, 0) ==> [False, False]
    tf.reduce_all(x, 1) ==> [True, False]
    ```

  * tf.reduce_any()

  * tf.reduce_mean()

* 索引提取

  * tf.argmin(input, axis=None, name=None, dimension=None)，返回input最大值的索引index。

  * tf.unique(x, out_idx=None, name=None)

    ```
    # tensor 'x' is [1, 1, 2, 4, 4, 4, 7, 8, 8]
    y, idx = unique(x)
    y ==> [1, 2, 4, 7, 8]
    idx ==> [0, 0, 1, 2, 2, 2, 3, 4, 4]
    ```

更多用法[在此](https://www.cnblogs.com/wuzhitj/p/6431381.html)

#### 搭建一般神经网络

以下通过伪代码的形式给出

```python
1. 导入数据
2. 搭建网络
   1. 输入
      给 x , y 声明占位符
   2. 进入隐藏层
      设置 weights , biases 和激励函数 tf.nn.xxx，得到输出 output
   3. 计算误差
      设置 loss 的计算方法
   4. 反向传播、梯度下降
      设置 train 的优化器
   5. 计算正确率
      根据 output 和 y 计算 accuracy
   6. 保存网络参数
      saver = tf.train.Saver()
3. 创建会话训练
   sess.run(tf.global_variables_initializer())
   1. 迭代次数
   		for step range(iterations):
   2. 批处理大小
   			for batch in range(batch_size):
   				sess.run()
4. 训练结果可视化
   1. 准确率曲线
   2. 误差柱状图
   3. 混淆矩阵
   4. Tensorboard 网络结构图 
```

#### 交叉验证

1. 常用的是**简单交叉验证**。8-2分，然后用训练集来训练模型，在测试集上验证模型及参数，接着，我们再把样本打乱，重新选择训练集和测试集，继续训练数据和检验模型。最后我们选择损失函数评估最优的模型和参数。

   参考Tensorflow 中的`train_test_split()`函数。

   ```
   x_train,x_test,y_train,y_test = train_test_split(x, y, random_state=1, test_size = 0.2)
   ```

2. **S 折交叉验证**（S-Folder Cross Validation）。S 折交叉验证会把样本数据随机的分成 S 份，每次随机的选择 S - 1 份作为训练集，剩下的 1 份做测试集。当这一轮完成后，重新随机选择 S - 1 份来训练数据。若干轮（小于 S ）之后，选择损失函数评估最优的模型和参数。

   参考Tensorflow 中的`cross_val_score()`函数。

   ```
   score = cross_val_score(model, x_data, y_data, cv = 10, scoring='accuracy')
   scores.append(score.mean())
   ```

   S 的数值大小可以通过实验得出。


#### 数据预处理部分

参考 sklearn 包：

* 标准化：把数据调整（scaling）为标准正态分布（standard normal）

* 规范化：提升模型收敛速度和模型精度
* 归一化：

图像处理部分：

* 图像灰度化

  * 最大值法
  * 平均值法
  * 加权法

  选取标准：亮度适中、边缘区分度明显

* 噪声去除

  * 均值滤波（效果差）
  * 中值滤波
  * 高斯滤波
  * 



待补充 ......