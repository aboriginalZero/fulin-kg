1. python 模块说明 

    * `__pycache__`：里面包含解释器编译的字节码，以 pyc 或者 pyo 后缀结尾。如果下次再次运行该模块的时候检查发现模块没有任何改变，就直接跑字节码，否则就重新编译。
    * `__init__`：拥有`__init__.py`文件的目录，在被 import 的时候会自动执行该文件的内容，通过`__all__`变量传入要导入文件名/方法的 list

2. 打印输出到文件中

    ```python
    import os
    import sys
    import errno
    
    def mkdir_if_missing(dir_path):
        try:
            os.makedirs(dir_path)
        except OSError as e:
            if e.errno != errno.EEXIST:
                raise
    
    class Logger():
        
        def __init__(self, fpath=None):
            # 构造函数
            self.console = sys.stdout
            self.file = None
            if fpath is not None:
                mkdir_if_missing(os.path.dirname(fpath))
                self.file = open(fpath, 'w')
    
        def __del__(self):
            # 析构函数（不论是手动 del 或者 GC 回收都会调用）
            self.close()
    
        def __enter__(self):
            # 当 with 语句在开始运行时，会在上下文管理器对象上调用此方法
            pass
    
        def __exit__(self, *args):
            # 当 with 语句在运行结束时，会在上下文管理器对象上调用此方法
            self.close()
    
        def write(self, msg):
            # 在屏幕和日志各写一份
            self.console.write(msg)
            if self.file is not None:
                self.file.write(msg)
    
        def flush(self):
            # 将文件缓冲区中的数据立刻写入文件，同时清空缓冲区
            self.console.flush()
            if self.file is not None:
                self.file.flush()
                # 确保 self.file 在内存上的数据都写入了磁盘
                os.fsync(self.file.fileno())
    
        def close(self):
            self.console.close()
            if self.file is not None:
                self.file.close()
    
    sys.stdout = Logger(os.path.join("/log/model_name/train_timestamp.txt"))
    ```

3. 拿到文件夹中所有图片的路径，并以 list 返回

    ```python
    import glob
     
    ret_list = glob.glob('/home/caiyiwu/dataset/msmt17/bounding_box_train/*.jpg')
    ```

4. 代码开头会加上`from __future__ import *`这样的语句。这样的做法的作用就是将新版本的特性引进当前版本中，比如用 python2 解释器却用 python3 的语法

    ```python
    from __future__ import absolute_import
    from __future__ import division
    from __future__ import print_function
    from __future__ import unicode_literals
    ```

5. 通过 fire 自动生成命令行

    一个简单的示例如下

    ```python
    # demo.py
    import fire
    
    def train(data_path):
      pass
    
    if __name__ == '__main__':
      fire.Fire(train)
     
    # 在命令行上执行 python demo.py --data_path /home/caiyiwu/dataset/ 就可以执行函数了
    ```

    * 还可以给 Fire 传入一个类，那么所有的类成员方法都可以通过命令行直接调用
    * fire 默认使用 - 作为参数分隔符，所以如果你要在命令行传入类似 2017-04-22 的参数时，那么程序接收到的参数就肯定不是 2017-04-22 了，你需要使用 --separator 来改变分隔符，参考 [Changing the Separator](https://github.com/google/python-fire/blob/master/doc/using-cli.md#separator-flag)
    * fire 会自动区分你在命令行传入的参数的类型，例如 20170422 会自动识别成 int，hello 会自动识别成 str，'(1,2)' 会自动识别成 tuple，'{"name": "Alan Lee"}' 会自动识别成 dict。但是你如果想要传入一个字符串类型的 20170422 怎么办？那就需要这样写：'"20170422"' 或者 "'20170422'" 或者 \"20170422\"，总之呢，就是加一个转义，因为命令行默认会吃掉你的引号。参考 [Argument Parsing](https://github.com/google/python-fire/blob/master/doc/guide.md#user-content-argument-parsing)

6. 使用 GPU 训练

    按照 PCI_BUS_ID 顺序从 0 开始排列 GPU 设备 ，优先使用 1 号卡

    ```shell
    CUDA_DEVICE_ORDER=PCI_BUS_ID CUDA_VISIBLE_DEVICES='1,0' python train.py
    ```

7. Anaconda 用法

    ```shell
    conda env list 							  # 查看所有的 conda 环境
    conda create --name cbn python=3.6		  # 创建一个指定版本的名为 cbn 的环境
    conda create -n cbn --clone idm			  # 复制一个环境
    conda activate cbn					      # 切换到 cbn 环境
    conda list 				   				  # 查看当前环境下已安装的包  
    pip3 install -r requirements.txt	      # 根据文件下载包
    conda install request			    	  # 安装 requests 包
    conda remove request 					  # 卸载 requests 包
    conda remove -n cbn --all 				  # 删除 cbn 环境及环境下的所有的包
    ```
    
    

