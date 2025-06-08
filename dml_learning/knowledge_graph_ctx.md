## 知识图谱

#### 概念

以结构化的形式描述客观世界中概念、实体、关系，将互联网的信息表达成更接近人类认知世界的形式。简言之，知识图谱就是去发现世间万物之间的联系，在技术上就是将数据以一个一个的<subject,relation,object>的三元组形式存储起来。

#### 应用场景

* 搜索引擎、推荐系统
* 司法、金融等各领域的知识图谱，用于智能问答、辅助决策、风险规避等

#### 构建过程

* 确定目标业务
* 实体抽取，实体链接（两个实体同一个含义需要规整）。目前最主流的算法就是CNN+LSTM+CRF进行实体识别。

* 实体间关系抽取，拿到知识图谱最小单元三元组。比较经典算法的就是Piece-Wise-CNN和 LSTM+ Attention 。

* 知识存储。一般采用图数据库（neo4j等）

## 关系抽取

#### 概念

关系定义为两个或多个实体之间的某种联系，实体关系学习就是自动从文本那种检测和识别出实体之间具有的某种语义关系，也称为关系抽取。关系抽取的输出通常是一个三元组(实体 1，关系，实体 2)。例如，句子“北京是中国的首都、 政治中心和文化中心”中表述的关系可以表示为（中国，首都，北京），（中国， 政治中心，北京）和（中国，文化中心，北京）。

#### 问题难点

* 自然语言表达的多样性

  同一种关系可以有多种表达方式，例如“总部位置”这个语义关系可以用“X的总部位于Y”，“X总部坐落于Y”，“作为X的总部所在地，Y…”等等不同的文本表达方式。

* 关系表达的隐含性

  关系有时候在文本中找不到任何明确的标识，关系隐含在文本中。例如：蒂姆·库克与中国移动董事长奚国华会面商谈“合作事宜”，透露出了他将带领苹果公司进一步开拓中国市场的讯号。在这段文本中，并没有给出蒂姆·库克和苹果公司的关系，但是从“带领苹果公司”的表达，我们可以推断出蒂姆·库克是苹果公司的首席执行官(CEO)。

* 实体关系的复杂性

  同一个实体之间可能有多个关系，而且有的关系可以同时存在，而有的关系是具有时间特性的。如果两个人本来是夫妻关系，后来离婚了，他们就不是夫妻关系了，是前妻或者前夫的关系，这个类关系具有时空性，不能单独存在。

#### 领域内容

关系抽取系统处理各种非结构化/半结构化的文本输入（如新闻网页、商品页面、微博、论坛页面等），使用多种技术（如规则方法、统计方法、知识挖掘方法），识别和发现各种预定义类别的开放类别的关系。

* 根据类别是否预定义

  - 限定域关系抽取

    利用有监督或弱监督的方法抽取预定义的实体关系知识。

    - 有监督：挖掘更多能表征相应语义关系的特征
    - 弱监督：降低自动生成语料中的噪声

  - 开放域关系抽取

    用无监督的方法自动抽取关系三元组

* 根据关系抽取的方法

  * 基于规则的方法
  * 基于机器学习

* 根据对监督知识的依赖

  * 无监督

    无监督关系抽取方法主要基于**分布假设**，其核心思想是如果两个实体对具有相似的语境，那么这两个实体对倾向于具有相同的语义关系。无监督关系抽取将两个实体的上下文作为表征语义关系的特征。

  * 有监督

    看成是一个多分类问题。

    * 基于特征向量

    * 基于核方法

    * 基于深度学习

      * Pipeline

        把实体识别和关系分类作为两个完全独立的过程，不会相互影响，关系的识别依赖于实体识别的效果。Att-BLSTM

      * Joint Model

        实体识别和关系分类的过程共同优化。LSTM-RNNs

  * 弱监督

    * 使用回标的思想，利用现有知识库中的关系三元组，自动回标三元组中实体所在的文本作为训练数据。

    * 使用半监督学习，利用少量的标注信息进行学习

      * 基于Bootstrap的方法

        利用少量实例作为初始种子(seed tuples)的集合，然后利用 pattern 学习方法进行学习，通过不断迭代从非结构化数据中抽取实例，然后从新学到的实例中学习新的 pattern 并扩充 pattern 集合，寻找和发现新的潜在关系三元组。Snowball

      * 远程监督方法

        基本假设：两个实体如果在知识库中存在某种关系，则包含该两个实体的非结构化句子均能表示出这种关系。

        通过对知识库与非结构化文本对齐来自动构建大量训练数据，减少模型对人工标注数据的依赖，增强模型跨领域适应能力。PCNN+Attention

    * 使用主动学习等技术以尽可能少的代价提升抽取效果。

## nlp常用标签

#### 标签列表

- B，即Begin，表示开始
- I，即Intermediate，表示中间
- E，即End，表示结尾
- S，即Single，表示单个字符
- O，即Other，表示其他，用于标记无关字符

#### 常用标签方案

- IOB1: 标签I用于文本块中的字符，标签O用于文本块之外的字符，标签B用于在该文本块前面接续则一个同类型的文本块情况下的第一个字符。
- IOB2: 每个文本块都以标签B开始，除此之外，跟IOB1一样。
- IOE1: 标签I用于独立文本块中，标签E仅用于同类型文本块连续的情况，假如有两个同类型的文本块，那么标签E会被打在第一个文本块的最后一个字符。
- IOE2: 每个文本块都以标签E结尾，无论该文本块有多少个字符，除此之外，跟IOE1一样。
- START/END （也叫SBEIO、IOBES）: 包含了全部的5种标签，文本块由单个字符组成的时候，使用S标签来表示，由一个以上的字符组成时，首字符总是使用B标签，尾字符总是使用E标签，中间的字符使用I标签。

## 结巴中文分词

>参考[jieba官方](https://github.com/fxsjy/jieba)Readme

1. 分词

   ```python
   # sentence：待分词字符串
   # cut_all：全模式开启
   # HMM：使用HMM，会多发现一些新词
   # 返回值是迭代器
   cut(self, sentence, cut_all=False, HMM=True) 
   # 分词结果用列表返回
   lcut(self, sentence, cut_all=False, HMM=True) 
   # 搜索引擎模式分词，会把keyword都列出来
   cut_for_search(self, sentence, HMM=True) 
   # 使用
   seg_list = jieba.cut("我在看维达", cut_all=False)
   ```

2. 添加自定义词典

   1. 静态载入

      ```python
      jieba.load_userdict("dict_test.txt")
      ```

      `dict_test.txt` 中数据格式如下（UTF-8编码，编辑时用Win自带的编辑器会有BOM）：

      ```
      词语 词频（可略） 词性（可略），用空格隔开，顺序不可颠倒，每个词一行,如下：
      创新办 3 i
      云计算 5
      ```

   2. 动态载入

      ```python
      add_word(word, freq=None, tag=None)
      # 示例
      jieba.add_word(‘男孩子’, tag='BOY')
      del_word(word)
      suggest_freq(segment, tune=True)
      # 示例
      jieba.suggest_freq(('中', '将'), True)
      jieba.suggest_freq('台中', True)
      ```

   3. 设置主词典

      在 0.28 之前的版本是不能指定主词典的路径的，有了延迟加载机制后，你可以改变主词典的路径

      ```python
      jieba.set_dictionary('data/dict.txt.big')
      ```

3. 词语位置

   ```python
   result = jieba.tokenize(u'永和服装永和有限公司', mode='default')
   for tk in result:
       print("word %s\t\t start: %d \t\t end:%d" % (tk[0],tk[1],tk[2]))
   --------
   word 永和                start: 0                end:2
   word 服装                start: 2                end:4
   word 永和                start: 4                end:6
   word 有限公司            start: 6                end:10
   ```

   输入参数只接受 unicode，可选择搜索模式（model = 'search'）

4. 词性标注

   ```python
   import jieba.posseg as pseg
   words = pseg.cut(line)
   	for word, flag in words:
   		print('%s %s' % (word, flag))
   ------
   我 r
   爱 v
   北京 ns
   ```

   `jieba.posseg.dt` 为默认词性标注分词器，标注句子分词后每个词的词性，采用和 `ictclas` 兼容的标记法。

   | 标签 | 含义     | 标签 | 含义     | 标签 | 含义     | 标签 | 含义     |
   | ---- | -------- | ---- | -------- | ---- | -------- | ---- | -------- |
   | n    | 普通名词 | f    | 方位名词 | s    | 处所名词 | t    | 时间     |
   | nr   | 人名     | ns   | 地名     | nt   | 机构名   | nw   | 作品名   |
   | nz   | 其他专名 | v    | 普通动词 | vd   | 动副词   | vn   | 名动词   |
   | a    | 形容词   | ad   | 副形词   | an   | 名形词   | d    | 副词     |
   | m    | 数量词   | q    | 量词     | r    | 代词     | p    | 介词     |
   | c    | 连词     | u    | 助词     | xc   | 其他虚词 | w    | 标点符号 |
   | PER  | 人名     | LOC  | 地名     | ORG  | 机构名   | TIME | 时间     |

   可以通过`jieba.posseg.POSTokenizer(tokenizer=None)` 新建自定义分词器，`tokenizer` 参数可指定内部使用的 `jieba.Tokenizer` 分词器。

5. 并行分词

   ```python
   # 参数为并行进程数
   jieba.enable_parallel(4)
   # 关闭并行分词模式
   jieba.disable_parallel()
   ```

   并行分词仅支持默认分词器 `jieba.dt` 和 `jieba.posseg.dt`，以及不支持Windows

6. 关键词抽取

   - 基于TF-IDF算法
   - 基于TextRank算法

7. 自构建中文搜索引擎

   ChineseAnalyzer for Whoosh,参考[博客](https://blog.csdn.net/qq_1290259791/article/details/84104799)。

## 生成词向量

词向量技术将自然语言中的词转化为稠密的向量，相似的词会有相似的向量表示，这样的转化方便挖掘文字中词语和句子之间的特征。经典的语言模型方法：词袋模型、词向量模型，`word2vec`、`glove`、`ElMo`、`BERT`，[详细介绍](https://blog.csdn.net/xiayto/article/details/84730009)。

#### word2vec

通过词的上下文得到词的向量化表示。有两种方法：CBOW（通过附近词预测中心词）、Skip-gram（通过中心词预测附近的词）

1. 训练模型

   ```python
   # 该文件每一行是每一句分析后的词组，语料库不能大小，不然会报错
   sentences = LineSentence('jieba分析后的.txt')
   # sg=1 是skip-gram算法，对低频词敏感；默认sg=0为CBOW算法
   # size 是输出词向量的维数，值太小会导致词映射因为冲突而影响结果，值太大则会耗内存并使算法计算变慢，一般值取为100到200之间。
   # window 是句子中当前词与目标词之间的最大距离，3表示在目标词前看3-b个词，后面看b个词（b在0-3之间随机）
   # min_count是对词进行过滤，频率小于min-count的单词则会被忽视，默认值为5
   # workers控制训练的并行，此参数只有在安装了Cpython后才有效，否则只能使用单核
   # hs=1表示层级softmax将会被使用，默认hs=0且negative不为0，则负采样将会被选择使用
   model = Word2Vec(sentences, sg=1, hs=1, min_count=1, window=3, size=10, workers=4)
   ```

2. 保存/加载模型

   ```python
   #保存下来可以在之后追加训练
   model.save('path+filename')
   model = Word2Vec.load('path+filename')
   ```

3. 模型使用

   ```python
   # 如果使用的模型后续不再更新,转换成键值对模型
   use_model = model.wv
   del model
   
   model.most_similar(positive=['woman', 'king'], negative=['man'],topn=10)  
   #输出跟positive接近，跟negtive不接近的10个词 [('queen', 0.50882536), ...]  
      
   model.doesnt_match("breakfast angle dinner lunch".split())  
   #输出跟其他单词最不像的词 'angle'  
   
   model.similarity('woman', 'man')  
   #输出两个词之间的相似度 0.73723527  
      
   print(model['computer'])  # raw numpy vector of a word  
   #输出array([-0.00449447, -0.00310097,  0.02421786, ...], dtype=float32)
   ```

#### BERT

使用BERT获取词向量

```python
import torch
from transformers import BertModel, BertTokenizer, BertConfig, BertPreTrainedModel

# 进行模型的配置，变量为空即使用默认参数
config = BertConfig()
# 使用自定义配置实例化 Bert 模型
model = BertModel(config)
model = model.to(torch.device('cuda:0'))
# 查看模型参数
config = model.config
# 加载预训练模型（支持本地模型路径）
tokenizer = BertTokenizer.from_pretrained('pretrained/wmm')
# 将句子映射为词表里的id、token类型及掩盖的三个列表
input_ids = tokenizer.encode_plus('')
# 分词，token list
tokens = tokenizer.tokenize('NER系统成功的关键在很大程度上取决于其输入表示。')
# 将分词后的token映射为词表里的id
input_ids = tokenizer.convert_tokens_to_ids(tokens)
# 将id映射回token
ori_tokens = tokenizer.convert_ids_to_tokens(input_ids)
# 将id映射回带特殊符号的句子
ori_sent = tokenizer.decode(tokens)
print(tokens)
print(ori_tokens)
# BERT变体通过继承BertPreTrainedModel实现
```



## 命名实体识别（NER）

#### BiLSTM-CRF

BiLSTM-CRF 是使用深度学习的NER最常见的体系结构。NER系统成功的关键在很大程度上取决于其输入表示。在输入中同时包含单词级和字符级嵌入是必不可少的。常用步骤：

1. 分词，建立字符、标签映射表

2. 生成词向量（word2vec, glove, bert）

3. 提取句子特征，制作数据集（每个句子中的字符、标签、分词后的词语与位置的关系）

4. 制作batchManager（排序并补齐，yield的运用）

5. 创建模型

   1. 根据char_inputs和seg_inputs得到词嵌入向量

      ```python
      inputs = tf.placeholder(shape=(batch_size, num_steps), dtype=tf.int32, name='inputs')
      
      embedding = tf.Variable(tf.random_uniform([vocab_size, embedding_size], -1.0, 1.0), dtype=tf.float32)
      
      #inputs_embeded = [batch_size, num_steps,embedding_size]
      inputs_embeded = tf.nn.embedding_lookup(embedding, inputs)
      ```

   2. 搭建双向LSTM，得到最终输出和首尾状态

      ```python
      lstm_cell = tf.nn.rnn_cell.BasicLSTMCell(hidden_units)
      
      # outputs_fw, outputs_bw = [batch_size,num_step,lstm_dim]
      ((outputs_fw, outputs_bw), (outputs_state_fw, outputs_state_bw)) = tf.nn.bidirectional_dynamic_rnn(lstm_cell, lstm_cell, inputs_embeded, sequence_length=max_time)
      
      # outputs = [batch_size,num_step,lstm_dim*2]
      outputs = tf.concat((outputs_fw, outputs_bw), 2)
      ```

   3. 添加全连接层，得到logits_slot

      ```python
      # 自构建的全连接层，将2*lstm_dim变为1*lstm_dim
      with tf.variable_scope("hidden"):
          W = tf.get_variable("W", shape=[self.lstm_dim * 2, self.lstm_dim],dtype=tf.float32, initializer=self.initializer)
          b = tf.get_variable("b", shape=[self.lstm_dim], dtype=tf.float32,
                              initializer=tf.zeros_initializer())
          # lstm_outputs = [batch_size, num_steps, 2*lstm_dim]
          output = tf.reshape(lstm_outputs, shape=[-1, self.lstm_dim * 2])
          hidden = tf.tanh(tf.nn.xw_plus_b(output, W, b))
      
      # 自构建的类似于softmax的分类器
      with tf.variable_scope("logits_slot"):
          W = tf.get_variable("W", shape=[self.lstm_dim, self.num_tags],dtype=tf.float32,  initializer=self.initializer)
          b = tf.get_variable("b", shape=[self.num_tags], dtype=tf.float32, initializer=tf.zeros_initializer())
          pred_slot = tf.nn.xw_plus_b(hidden, W, b)
          # pred_slot = [batch_size,num_steps,num_tags] 
          pred_slot = tf.reshape(pred_slot, [-1, self.num_steps, self.num_tags])
      ```

   4. 定义crf损失函数

      ```python
      with tf.variable_scope("crf_loss" if not name else name):
          small = -1000.0
          # start_logits.shape (batch_size, 1, num_tags+1)
          start_logits = tf.concat(
              [small * tf.ones(shape=[self.batch_size, 1, self.num_tags]), tf.zeros(shape=[self.batch_size, 1, 1])],axis=-1)
          # pad_logits.shape (batch_size, num_steps, 1)    
          pad_logits = tf.cast(small * tf.ones([self.batch_size, self.num_steps, 1]), tf.float32)
          # [batch_size,num_steps, num_tags+1]
          logits = tf.concat([project_logits, pad_logits], axis=-1)
          # [batch_size, num_steps+1, num_tags+1]
          logits = tf.concat([start_logits, logits], axis=1)
          # 输入语句的真实标签，即groud truth, [batch_size, num_steps+1]
          targets = tf.concat(
              [tf.cast(self.num_tags * tf.ones([self.batch_size, 1]), tf.int32), self.targets], axis=-1)
          # 状态转移矩阵
          self.trans = tf.get_variable(
              "transitions", shape=[self.num_tags + 1, self.num_tags + 1], initializer=self.initializer)
          log_likelihood, self.trans = crf_log_likelihood(inputs=logits, tag_indices=targets,transition_params=self.trans, sequence_lengths=lengths + 1)
          # 最终的损失函数
          loss = tf.reduce_mean(-log_likelihood)
      ```

   5. 定义优化器，最小化损失函数

      ```python
      optimizer = tf.train.AdamOptimizer(lr)
      train_op = optimizer.minimize(loss)
      ```

   6. 计算准确率

      根据转移方程用维特比算法求得最佳路径来解码

6. 训练、测试

## Other

远程监督：假设两个实体在训练数据集中的关系与在基础知识库（如freebase）中的关系保持一致。

训练集由远程监督生成，仅在测试集和验证集上手动纠错

包的概念：包含同样实体对的所有句子

分词工具：[pkuseg](https://github.com/lancopku/PKUSeg-python),jieba,THULAC

在大量未标记数据上经过分词拿到词向量模型

bert远胜于elmo,skip-gram,glove，使用bert的输出作为词向量或者加上一个全连接层做分类，为避免过拟合情况，把人名改成英文格式，”刘伟明“（Liu Weiming）

特征工程，挖掘更多实体之间的信息，比如，[通过人名判断性别关系](https://pypi.org/project/ngender)，姓氏帮助判断血缘关系（父子，爷孙等）

考虑到数据分布不平均的问题，可以果断舍弃部分数据不用于训练，或者通过将中文翻译成英文再翻译回中文的手段增强指定类别的数据

用Parser控制参数超参数，将模型用类形式组织

[生成模型和判别模型的区别](https://www.jianshu.com/p/4ef549eb0ad4)

[从似然和期望的角度理解交叉熵](https://zhuanlan.zhihu.com/p/104580221)

从txt或其他文件读取1000*1000的数据，直接读取需要1s，而转成.npy后，读取只需要约0.01s，快100倍。多次读取相同的数据文件提升效果十分明显。

## NLP

目前，人们主要通过两种思路来进行自然语言处理，一种是基于规则的理性主义，另外一种是基于统计的经验主义。

RNN应用到机器学习，减轻了对特征工程的依赖。

![1583307130451](C:\Users\Hasee\AppData\Roaming\Typora\typora-user-images\1583307130451.png)

#### NLP问题分类

- 类别（对象）到序列：文本生成、图像描述生成
- 序列到类别：情感分类、文本分类
- 同步的序列到序列模式：中文分词、信息抽取、词性标注、语义角色标注
- 异步的序列到序列模式：机器翻译、自动摘要、对话系统

#### 文本与图像信息的差异

|      | 输入量             | 信息量                             | 关系                 | 底层特征                                         |
| ---- | ------------------ | ---------------------------------- | -------------------- | ------------------------------------------------ |
| 图像 | 二维像素集         | 黑白：128-256 彩色：3（128-256）   | 欧式空间             | 纹理，形状，彩色                                 |
| 文本 | 一维离散词符号序列 | 共250k（英文词类），一般用几千个词 | 语法、句法、语义关系 | 句子长度，句子在段落中的位置，段落在文章中的位置 |

#### 相关书籍

- 《知识图谱》赵军 高等教育出版社

- 《统计自然语言处理》 宗成庆 清华大学出版社

- 《神经网络与深度学习》 邱锡鹏 机械工业出版社 [网站](https://nndl.github.io/)

- 《Python核心编程》、《Effective Python: 编写高质量Python代码的59个有效方法》

- 《Python 设计模式》、《Python高级编程》、《Python性能分析与优化》

#### 会议

ACL、NAACL、EMNLP和COLING

#### 期刊

Computational Linguistics、TACL、ACM Transactions on Speech and Language Processing、ACM Transactions on Asian Language Information Processing、Journal of Quantitative Linguistics

#### 文献检索

Google Scholar，按标题出现关键词搜索：allintitle:"latent dirichlet allocation"，可以搜索在标题出现某些关键词的论文。搜索引擎常用的and、or和""均支持，其中""表示按引号中的字符串完整搜索。

为了了解某个课题，在维基百科等权威在线百科全书中查询该主题的科普综述介绍。在此基础上，可以在中文知网（CNKI）中搜索"课题名称+综述"或在Google Scholar中搜索“课题名称 + survey / review / tutorial / 综述”来查找。

比较新的研究方向可以通过查找该方向发表的最新论文，阅读它们的“相关工作”章节，顺着列出的参考文献，了解相关研究脉络。当然，还可以去各大学术会议或暑期学校上找Tutorial报告。

论文题目越短其创新价值更高的概率会更大。

#### 文献阅读顺序

-> title -> abstract ：大致可以判断这篇论文与自己研究课题的相关性

-> introduction -> experiments & results -> method ：精读导论和实验结果判断学术价值，是否阅读本文工作了解方法细节

-> related work -> conclusion ：了解相关工作和未来工作

#### 寻找idea

首先要有区分研究想法好与不好的能力，对学科文献的全面掌握。其次，

**实践法**。即在研究任务上实现已有最好的算法，通过分析实验结果，例如发现这些算法计算复杂度特别高、训练收敛特别慢，或者发现该算法的错误样例呈现明显的规律，都可以启发你改进已有算法的思路。现在很多自然语言处理任务的Leaderboard上的最新算法，就是通过分析错误样例来有针对性改进算法的。

**类比法**。即将研究问题与其他任务建立类比联系，调研其他相似任务上最新的有效思想、算法或工具，通过合理的转换迁移，运用到当前的研究问题上来。例如，当初注意力机制在神经网络机器翻译中大获成功，当时主要是在词级别建立注意力，后来我们课题组的林衍凯和沈世奇提出建立句子级别的注意力解决关系抽取的远程监督训练数据的标注噪音问题，这就是一种类比的做法。

**组合法**。即将新的研究问题分解为若干已被较好解决的子问题，通过有机地组合这些子问题上的最好做法，建立对新的研究问题的解决方案。例如，我们提出的融合知识图谱的预训练语言模型，就是将BERT和TransE等已有算法融合起来建立的新模型。

#### 论文典型结构

绝大部分论文由以下六大部分构成：摘要（Abstract）、介绍（Introduction）、相关工作（Related Work）、方法（Method）、实验（Experiment）、结论（Conclusion）。

- 摘要：用100-200词简介研究任务与挑战、解决思路与方法、实验效果与结论。
- 介绍：用1页左右篇幅，比摘要更详细地介绍研究任务、已有方法、主要挑战、解决思路、具体方法、实验结果。
- 相关工作：用0.5-1页左右篇幅介绍研究任务的相关工作，说明本文工作与已有工作的异同。
- 方法：用2-3页篇幅介绍本文提出的方法模型细节。
- 实验：用2-3页篇幅介绍验证本文方法有效性的实验设置、数据集合、实验结果、分析讨论等。
- 结论：简单总结本文主要工作，展望未来研究方向。

章节层面，Introduction提到已有方法面临的几个挑战，就要对应本文提出的几个创新思路，对应Method中的几个具体算法，对应Experiment中的几个实验验证。

段落和句子层面，段间要注意照应，是并列、递进、转折还是总分关系，需要谋划妥当，要有相应句子或副词衔接。段内各句，有总有分，中心思想句和围绕论述句分工协作。

除了整体结构上的建议外，每个部分也各有定式，下面按各部分提供一些写作建议，同时用我们最近发表的一篇ACL 2018论文 [2] 作为例子。

### Issues

1. 分词时，针对特定领域加入特定字典
2. 标注数据，有UI软件的人工标注；针对格式统一文件的正则匹配；爬取特定数据，如通过人名对去爬取句子，寻找“王健林”(PER)，“万达集团”(ORG)同时出现的句子作为“老板”这个关系的语料，特定领域的话，要去查找该领域的词库
3. 字向量，对中文来讲更加自然，不会把分词的错误带进来，现在也有对中文拆分做偏旁部首的embedding，或者直接对字向量做bigram的研究。
4. 多种实体关系，把A_B, A_C, B_C三种实体的feature分别输入，即把一句话变成三句仅仅是实体feature不同其它一样的句子来尝试。
5. 用one-hot形式的位置编码来代替随机初始化的向量
6. fixlen是指的句子的长度，而max_len是指记录位置向量的范围，这个显然是要小于句子长度的。60+5*2=70，5 是位置向量的size。
7. num_step是序列的长度，和sentence_length长度一致

### 项目问题

1. tensorboard 不能写入太多的数据，否则会出错（亲身经历）

2. train loss 不断下降，test loss不断下降，说明网络仍在学习;（最好的）

   train loss 不断下降，test loss趋于不变，说明网络过拟合;（max pool或者正则化）

   train loss 趋于不变，test loss不断下降，说明数据集100%有问题;（检查dataset）

   train loss 趋于不变，test loss趋于不变，说明学习遇到瓶颈，需要减小学习率或批量数目；（减少学习率）

   train loss 不断上升，test loss不断上升，说明网络结构设计不当，训练超参数设置不当，数据集经过清洗等问题。（最不好的情况）

3. 