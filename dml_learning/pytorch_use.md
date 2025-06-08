## 个人Pytorch使用

### 下载安装

关掉小飞机，下载指定的老版本的 torch，可以从这里面下载，然后本地 pip install

```shell
pip install torch==1.4.0 torchvision==0.5.0 -f https://download.pytorch.org/whl/torch_stable.html
```

去 pytorch 官网下载还不好使

### 数据创建

由列表、numpy创建或tensor自定义的随机整数、正态分布等

```python
import torch
import numpy as np


# *****tensor创建*****
device = torch.device('cuda:0')
a = np.array([1, 2, 3])
t1 = torch.from_numpy(a)  # 原地创建tensor, t1与t1.numpy()共享内存
t2 = torch.as_tensor(a)  # 同一设备的话，原地创建tensor
t3 = torch.as_tensor(a, device=device)  # 先注释，为了后面运行快
a[0] = -1
t4 = torch.full_like(t1, 1)
t5 = torch.arange(1, 2.5, 0.5)
t6 = torch.linspace(start=-10, end=10, steps=5)  # 从-10到10分5步取完
t7 = torch.normal(mean=2, std=3, size=(2, 2, 3))  # 当mean=0,std=1时，等价于torch.randn(2,2,3)
t8 = t7.to(device) # 跟t7尺度数值一样的张量，但是在不同设备上
t9 = torch.rand(1, requires_grad=True, device=device)  # 生成一个[1]的浮点数随机矩阵，且计算梯度
t10 = t9.item()
for i in range(1, 11):
    print('-----t' + str(i) + '-----')
    eval('print(t' + str(i) + ')')
```

### 基本数学运算

加、减、除、哈达玛积、矩阵乘、幂、开方、指数、对数、绝对值、sigmoid函数、近似值、裁剪

```python
import torch
import numpy as np

# a.shape = [2,2,3]
a = torch.tensor([[[4, 2, 3], [3, 2, 1]], [[1, 2, 3], [3, 2, 3]]], dtype=torch.float32)
# d.shape = [2,3,2]
d = torch.tensor(np.array([[[2, 1], [3, 2], [4, 3]], [[1, 4], [2, 3], [2, 3]]]), dtype=torch.float32)
# b.shape = [3] 相当于[1,3]
b = torch.tensor([1, -2, 3], dtype=torch.float32)

# ******基本数学运算*****
c1 = a + b #开新内存，用a.di
c2 = a - b
c3 = a / b
c4 = torch.mul(a, b)  # 逐个相乘
c5 = torch.matmul(a, d)  # 矩阵乘，要保证维度一致
# 对位相乘用torch.mul，二维矩阵乘法用torch.mm，批量二维矩阵用torch.bmm
# batch、广播用torch.matmul
c6 = b.pow(2)
c7 = b.sqrt()
c8 = b.rsqrt() #开方再取倒数
c9 = torch.exp(b)
c10 = torch.log(c9)  # 以e为底，还有torch.log1p(),即torch.log(1+input)
c11 = torch.abs(b)  # c11 = torch.abs_(b) 带下划线意为在原内存执行操作
c12 = torch.sigmoid(b)
for i in range(1, 13):
    print('-----c' + str(i) + '-----')
    eval('print(c' + str(i) + ')')

e = torch.tensor([3.14, 4.98, 5.5])
print('-----取下,取上,取整数,取小数,四舍五入,值裁剪-----')
print(e.floor(), e.ceil(), e.trunc(), e.frac(), e.round(), e.clamp(4, 5))
```

### 常用矩阵运算

求最值及其索引、求p-norm、求累加指数的对数、求均值方差、累加、累乘、求（连续）不同值、排序、求前n大、判断是否相等、构造对角矩阵、求迹、SVD

```python
import torch
import numpy as np

# a.shape = [2,2,3]
a = torch.tensor([[[4, 2, 3], [3, 2, 1]], [[1, 2, 3], [3, 2, 3]]], dtype=torch.float32)
# d.shape = [2,3,2]
d = torch.tensor(np.array([[[2, 1], [3, 2], [4, 3]], [[1, 4], [2, 3], [2, 3]]]), dtype=torch.float32)
# b.shape = [3] 相当于[1,3]
b = torch.tensor([1, -2, 3], dtype=torch.float32)
aa = torch.rand(2,3) #[2,3] 0-1的均匀分布，等概率落在0-1范围
bb = torch.randn(2,3)#[2,3] 0-1的正态分布，均值为0，方差为1
aa_bb_diff = (aa-bb).norm().item() < 1e-6 # 判断aa,bb是否相等
# *****常用矩阵运算*****
d1 = torch.argmax(a, dim=0)  # [2,3] a[0]跟a[1]之间的比较，torch.argmax是对应的索引
d2 = torch.argmax(a, dim=1)  # [2,3] a[:,0]跟a[:,1]之间的比较
d3 = torch.argmax(a, dim=2)  # [2,2] a[:,:,0]跟a[:,:,1]跟a[:,:,2]之间的比较
d4 = torch.dist(a, b, p=2)  # p-norm距离(p=2为欧氏距离)
d5 = torch.logsumexp(a, dim=1)  # 求1维度上log（exp的累加）
d6 = torch.mean(a, 1)  # 求1维度上的平均数，可以用d6,d7 = torch.std_mean(a,1)来求均值和标准差
d7 = torch.std(a, 1)  # 求1维度上的标准差,可以用d6,d7 = torch.var_mean(a,1)来求均值和方差
d8 = torch.sum(a, dim=1)  # 求1维度上的累加
d9 = torch.prod(a, dim=1)  # 求1维度上的累乘
d10 = torch.unique(a, return_inverse=True, return_counts=True)  # 求a摊平之后的不同元素及其个数
d11 = torch.unique_consecutive(a, return_inverse=True, return_counts=True)  # 求a摊平之后的连续不同元素及其个数
d12 = torch.argsort(a, dim=2)  # 求升序排序之后的数值在原矩阵中对应的索引
d13 = torch.topk(a, k=2, dim=2)  # 求第2维里面前k大的数
d14 = torch.eq(a, b)  # [2,2,3] 各元素对比
d15 = torch.equal(a, b)  # True or False, 要求两句矩阵尺寸、数值完全一样
d16 = torch.diag(b, 0)  # 以一维张量b作为第二行对角线元素构建矩阵
d17 = torch.trace(d16)  # 必须得是2维张量
d18 = torch.svd(d16)  # 必须得是2维张量
for i in range(1, 19):
    print('-----d' + str(i) + '-----')
    eval('print(d' + str(i) + ')')
```

### Tensor 维度操作

拼接、叠加、转置、升维、降维、掩码取值、查值、取值、摊平、取张量不同维度视角

```python
import torch
import numpy as np

# a.shape = [2,2,3]
a = torch.tensor([[[4, 2, 3], [3, 2, 1]], [[1, 2, 3], [3, 2, 3]]], dtype=torch.float32)
# d.shape = [2,3,2]
d = torch.tensor(np.array([[[2, 1], [3, 2], [4, 3]], [[1, 4], [2, 3], [2, 3]]]), dtype=torch.float32)
# b.shape = [3] 相当于[1,3]
b = torch.tensor([1, -2, 3], dtype=torch.float32)

#*****Tensor 维度操作*****
f1 = torch.cat((a, a), 0)  # [4,2,3]
f2 = torch.cat((a, a, a), 2)  # [2,2,9]
f3 = torch.stack((a, a), 0)  # [2,2,2,3]
f4 = torch.stack((a, a), 2)  # [2,2,2,3]
f5 = torch.stack((a, a), 3)  # [2,2,3,2]
f6 = a.transpose(1, 2)  # [2,3,2]
f7 = a.permute(1, 0, 2)  # [2,2,3]
f8 = a.unsqueeze(2)  # [2,2,1,3]
f9 = f7.unsqueeze(0)  # [1,2,2,3]
f10 = f8.squeeze()  # [2,2,3] 不指定参数，则去掉所有多余的维度
f11 = torch.masked_select(a, mask=a.gt(3))  # [6] 通过掩码选取a中大于3的数(6个),a.le(3)表示选择小于等于3的
f12 = torch.where(a > 3, a, torch.zeros_like(a))  # [2,2,3] 选取a中大于3的数，其他的用0表示
f13 = torch.narrow(a, 2, 1, 2)  # [2,2,2] 取第2维的[1:1+2]，可以通过torch.index_select，或者通过切片a[:, :, 1:3]的方式
f14 = torch.index_select(a, 2, index=torch.tensor([1, 2]))
f15 = torch.flatten(a, start_dim=1)  # [2,6] 从第一维开始摊平
f16 = torch.reshape(a, (-1, 2 * 3))  # [2,6] 可能改变源数据形状，相当于tensor.contiguous().view()
f17 = a.view(-1, 2 * 3)
f18 = a.flip(1, 2)  # [2,2,3] 翻转张量的第1，2维
for i in range(1, 19):
    print('-----f' + str(i) + '-----')
    eval('print(f' + str(i) + '.shape)')
    eval('print(f' + str(i) + ')')
```

### 神经网络

先给包重命名

```python
import torch
import torch.nn.functional as F
import torch.nn as nn
import torch.optim as optim
import torch.nn.utils.rnn as rn
from torch.utils.tensorboard import SummaryWriter
```

#### 构造网络

继承`nn.Module`构造网络模型

```python
class MyModel(nn.Module):
    def __init__(self):
        super(MyModel,self).__init__()
        pass
    def forward(self,input):
        pass
        return output
```

通过`nn.Sequential()`函数构造网络模型

```python
model = nn.Sequential(nn.Conv1d(),
                    nn.MaxPool1d(), # nn.AvgPool1d()
                    nn.ReLU(), #nn.Tanh() or nn.LeakyReLU()
                    nn.Dropout(),
                    nn.Linear())

model = nn.Sequential(OrderDict([
    ('conv1', nn.Conv1d()),
    ('pool1', nn.MaxPool1d()),
    ('relu', nn.ReLU()),
    ('drouput',nn.Dropout()),
    ('linear',nn.Linear())
]))
```

#### 加载模型和参数

```python
# 模型结构和参数
torch.save(model,'save_path')
model = torch.load('save_path')
# 模型参数
torch.save(model.state_dict(),'save_path')
model.load_state_dict(torch.load('save_path'))
# 加载部分模型参数
pre_model = pre_model(pretrained = True)
pre_dict, model_dict = pre_model.state_dict(), model.state_dict()
pre_dict = {k:v for k,v in pre_model.items() if k in model_dict}
model_dict.update(pre_dict)
model.load_state_dict(model_dict)
```

#### 微调（fine-tune）

先使用`Module.children()`方法查看网络的直接子模块，将不需要调整的模块中的参数设置为`param.requires_grad = False`，同时用一个list收集需要调整的模块中的参数。具体代码为：

```python
for i,k in enumerate(model.children()):
    if i < 6: # 具体数值取决于上游网络的层数
        for param in k.parameters():
            param.requires_grad = False
# 只有True的才训练
optimizer = optim.SGD(filter(lambda p: p.requires_grad, model.parameters()), lr = 1e-3)
```

或者也可以在创建模型的时候直接指定某些模块不更新梯度

```python
class Net(nn.Module):
    def __init__(self):
        super(Net, self).__init__()
        self.conv1 = nn.Conv2d(1, 6, 5)
        self.conv2 = nn.Conv2d(6, 16, 5)
		# 前面的模块是False，后面的不变
        for p in self.parameters():
            p.requires_grad=False

        self.fc1 = nn.Linear(16 * 5 * 5, 120)
        self.fc2 = nn.Linear(120, 84)
        self.fc3 = nn.Linear(84, 10)
```

#### 规范化处理

```python
nn.BatchNorm1d() # 全连接层
nn.BatchNorm2d() # 卷积层
nn.LayerNorm()
nn.InstanceNorm1d()
```

#### 参数初始化

* 指定网络中的某一层

  ```python
  self.conv1 = nn.Conv2d(3, 64, kernel_size=7, stride=2, padding=3)
  init.xavier_uniform(self.conv1.weight)
  init.constant(self.conv1.bias, 0.1)
  ```

* 对整个网络不同层分别初始化

  ```python
  for m in model.modules():
      #卷积层参数初始化
      if isinstance(m,nn.Conv2d):
          nn.init.normal(m.weight.data)
          nn.init.xavier_normal(m.weight.data)
          nn.init.kaiming_normal(m.weight.data)
          m.bias.data.fill_(0)
      #全连接层参数初始化
      elif isinstance(m,nn.Linear):
          m.weight.data.normal_()
  ```

#### 优化器&损失函数

```python
optimizer = optim.SGD([input],lr=0.01,momentum=0.9)
optimizer = optim.Adam(net.parameters(),lr=0.0001)
loss = nn.MSELoss()
loss = nn.KLDivLoss()
loss = nn.NLLLoss()
loss = nn.CrossEntropyLoss()
input = torch.randn(2, 3, requires_grad=True)
target = torch.empty(2, dtype=torch.long).random_(3)
output = loss(input,target)
# 每个batch都要对优化器清零，或者通过多个batch更新参数缓解显存不足的问题
optimizer.zero_grad()
output.backward()
optimizer.step()
```

#### 动态调整学习率

根据损失或者准确度变化调整学习率（一般情景）

```python
optimizer = torch.optim.SGD(model.parameters(), lr=0.1, momentum=0.9)
optim.lr_scheduler.ReduceLROnPlateau(optimizer, 'max', patience=3)
for epoch in range(10):
    train(...)
    val_acc = validate(...)
    # 降低学习率需要在给出 val_acc 之后
    scheduler.step(val_acc)
```

根据不同层设置不同的学习率（fine-tune情景）

```python
model = BERT(pretrained=True)
optimizer = optim.Adam([
        {'params': [param for name, param in model.named_parameters() if name[-4:] == 'bias'], 'lr': 2 * args['lr']},
        {'params': [param for name, param in model.named_parameters() if name[-4:] != 'bias'], 'lr': args['lr'], 'weight_decay': args['weight_decay']}
    ], betas=(args['momentum'], 0.9))
```

#### 词嵌入

```python
# one-hot形式
F.one_hot(torch.arange(0, 5) % 3, num_classes=5)
# 加载预训练权重
embedding = nn.Embedding.from_pretrained(weight)
# 按0-1正态分布随机生成
embedding = nn.Embedding(10, 3)
# 按索引lookup
input = torch.LongTensor([[1, 2, 4, 5], [4, 3, 2, 9]])
output = embedding(input) # shape = [2,4,3]
```

#### 循环神经网络

```python
input = rn.pad_sequence([input]) # 补全操作
rnn = nn.RNN(input_size=10, hidden_size=20, num_layers=2) # nn.LSTM() or nn.GRU()
output,hn = rnn(input,h0)
```

多头注意力机制

```python
multi_attn = nn.MultiheadAttention(embed_dim,num_heads)
attn_output,attn_output_weights = multi_attn(query,key,value)
```

Transformer

```python
tm = nn.Transformer(nhead=16,num_encoder_layers = 12)
src,tgt = torch.rand((10, 32, 512)), torch.rand((20, 32, 512))
out = tm(src,tgt) # out.shape = tgt.shape
```

Transformer-encoder

```python
encoder_layer = nn.TransformerEncoderLayer(d_model=512, nhead=8)
tm_encoder = nn.TransformerEncoder(encoder_layer, num_layers=6)
src = torch.randn(10, 32, 512)
out = tm_encoder(src) #out.shape = src.shape
```

#### 度量函数

```python
# 余弦距离
dis = nn.CosineSimilarity()(torch.randn(2,3),torch.randn(2,3))
# 明氏距离（默认为欧氏距离）
dis = nn.PairwiseDistance()(torch.randn(2,3),torch.randn(2,3))
```

#### 固定随机种子

固定随机种子以复现网络结果

```python
import os
import random
import numpy as np
import torch
seed = 0
random.seed(seed)
np.random.seed(seed)
os.environ['PYTHONHASHSEED'] = str(seed) # 为了禁止hash随机化，使得实验可复现。
torch.manual_seed(seed)            # 为CPU设置随机种子
torch.cuda.manual_seed(seed)       # 为当前GPU设置随机种子
torch.cuda.manual_seed_all(seed)   # 为所有GPU设置随机种子
```

如果Dataloader使用了多线程（num_workers > 1）：

```python
GLOBAL_SEED = 1
 
def set_seed(seed):
    random.seed(seed)
    np.random.seed(seed)
    torch.manual_seed(seed)
    torch.cuda.manual_seed(seed)
    torch.cuda.manual_seed_all(seed)
 
GLOBAL_WORKER_ID = None
def worker_init_fn(worker_id):
    global GLOBAL_WORKER_ID
    GLOBAL_WORKER_ID = worker_id
    set_seed(GLOBAL_SEED + worker_id)
 
dataloader = DataLoader(dataset, batch_size=16, shuffle=True, num_workers=2, worker_init_fn=worker_init_fn)
```

#### Tensorboard 使用

```python
writer = SummaryWriter('logs')
writer.add_scalar('random_val', torch.tensor([1]).item(), global_step=i)
writer.add_embedding(torch.randn(100, 3),i)
writer.add_text('lstm','This is an lstm',i)
# pr曲线
writer.add_pr_curve('pr_curve', groud_truth, predictions, i)
```

### 概念说明

1. PyTorch中Tensor计算得来的标量都在CPU上，只有Tensor才分GPU版和CPU版本，GPU上的Tensor需要先转换成CPU版才能转成numpy。

2. nn.NLLLoss()与nn.log_softmax()配合使用，效果等价为CrossEntropyLoss()

3. optimizer.zero_grad()和model.zero_grad()的区别：前者是只将优化器对应的参数，梯度进行清零，后者是将整个网络的梯度进行清零

4. 假设一个函数的输入为k维向量，输出为标量，那么它的海森矩阵（Hessian matrix）有k个特征值。当函数的海森矩阵在梯度为零的位置上的特征值：

   - 全为正时，该函数得到局部最小值。
   - 全为负时，该函数得到局部最大值。
   - 有正有负时，该函数得到鞍点。

   随机矩阵理论告诉我们，对于一个大的高斯随机矩阵来说，任一特征值是正或者是负的概率都是0.5。那么，以上第一种情况的概率为 $0.5^{k}$。由于深度学习模型参数通常都是高维的（k很大），目标函数的鞍点通常比局部最小值更常见。

5. Pytorch 与 Tensorflow 1.x 对比

   - 命令式编程（Pytorch）操作简单容易调试。当我们在Python里使用命令式编程时，大部分代码编写起来都很直观。同时可以很方便地获取并打印所有的中间变量值，或者使用Python的调试工具。
   - 符号式编程（Tensorflow）更高效并更容易移植。一方面，在编译的时候系统容易做更多优化；另一方面，符号式编程可以将程序变成一个与Python无关的格式，从而可以使程序在非Python环境下运行，以避开Python解释器的性能问题。

### 神经网络设计

 * LeNet

   2卷积池化+3全连接

   在每个卷积、全连接之后加batch_norm，效果提升很多。

 * AlexNet

   2卷积池化+2卷积+1卷积池化+3全连接

   网络由浅入深，开始适用ReLu和Dropout应对深层网络过拟合的可能。

* VGG

  5个VGG块+3全连接

  引入VGG块，即多个小卷积层+1池化层，它可以指定卷积层的数量和输入输出通道数。其中，卷积层保持输入的高和宽不变，而池化层则对其减半。

  用多个小卷积层来代替大卷积层，保证具有相同感知野的条件下，提升了网络的深度。

* NiN

  3NiN最大池化+1NiN平均池化

  引入NiN块，即1卷积层+2“全连接”层（1x1卷积）。

  去掉所有的全连接层，在NiN块之后接最大池化层，在最后一个NiN块之后接全局平均池化层

* GoogLeNet

  引入Inception块，即4条并行的线路。前3条线路使用窗口大小分别是1x1，3x3和5x5的卷积层来抽取不同空间尺寸下的信息，其中中间2个线路会对输入先做1x1卷积来减少输入通道数，以降低模型复杂度。第四条线路则使用3x3最大池化层，后接1×1卷积层来改变通道数。

* ResNet

  引入residual块，即ReLU（2卷积、批量归一化、ReLU()+初始输入）

* DenseNet

  引入dense块，不同于residual块将输入与输出相加，DenseNet在通道维上连结输入与输出

  引入过渡层，通过1x1卷积层减小通道数，并使用步幅为2的平均池化层减半高和宽，从而减少dense块带来的通道数增加

### 常规训练/测试

1. 最外层是epoch循环
2. 里面跟一个dataloader
3. 取出feature_batch and label_batch
4. 模型前向推理
5. pred和label算loss
6. 优化器把梯度置为0
7. 损失函数算反向传播的梯度
8. 优化器进行梯度更新
9. 其他操作，比如tensorfboard 你用以上我总说的和下面的代码一一对

```python
for epoch in range(35):
    
    # training process
    train_loss = 0.
    train_acc = 0.
    model.train()
    for i, (batch_x, batch_y) in enumerate(train_loader):
        # batch_x = batch_x.cuda()
        # batch_y = batch_y.cuda()
        out = model(batch_x)
        loss = loss_func(out, batch_y)
        train_loss += loss.item()
        pred = torch.max(out, 1)[1]
        train_correct = (pred == batch_y).sum()
        train_acc += train_correct.item()
        optimizer.zero_grad()
        loss.backward()
        optimizer.step()
        
    # eval process
    model.eval()
    eval_loss = 0.
    eval_acc = 0.
    for j,(batch_x, batch_y) in enumerate(test_loader):
        # batch_x, batch_y = batch_x.cuda(), batch_y.cuda()
        out = model(batch_x)
        loss = loss_func(out, batch_y)
        eval_loss += loss.item()
        pred = torch.max(out, 1)[1]
        num_correct = (pred == batch_y).sum()
        eval_acc += num_correct.item()
    
```

windows下查看显卡使用情况，在文件夹C:\Program Files\NVIDIA Corporation\NVSMI里找到文件nvidia-smi.exe
