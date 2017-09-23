---
layout: post
title:  "Keras 入门教程(1) Raw"
date:   "2016-03-10 17:50:00"
categories: ML
keywords: keras
---

## 常用层

**Dense 层：**

Dense 层就是全连接层

![2DenseLayer](/images/ml/2denseLayer.png)

Dense就是常用的全连接层，所实现的运算是 

```
output = activation(dot(input, kernel)+bias)
```

其中activation是逐元素计算的激活函数，kernel是本层的权值矩阵，bias为偏置向量，只有当use_bias=True才会添加。

具体的算法可以参考下图:

![](/images/ml/nn_pricipal.png)

函数签名和每层的意思

```
keras.layers.core.Dense(units, activation=None, use_bias=True, 
    kernel_initializer='glorot_uniform', bias_initializer='zeros', 
    kernel_regularizer=None, bias_regularizer=None, activity_regularizer=None, 
    kernel_constraint=None, bias_constraint=None)
```

- units：大于0的整数，代表该层的输出维度。
- activation：激活函数，为预定义的激活函数名（参考激活函数），或逐元素（element-wise）的Theano函数。如果不指定该参数，将不会使用任何激活函数（即使用线性激活函数：a(x)=x）
- use_bias: 布尔值，是否使用偏置项
- kernel_initializer：权值初始化方法，为预定义初始化方法名的字符串，或用于初始化权重的初始化器。参考initializers
- bias_initializer：权值初始化方法，为预定义初始化方法名的字符串，或用于初始化权重的初始化器。参考initializers

上面都是常用的参数，还有一些正则相关的知识点没有写出来

**第一层，要输入 input 维度**

**Dropout 层**

```
keras.layers.core.Dropout(rate, noise_shape=None, seed=None)
```

为输入数据施加Dropout。Dropout将在训练过程中每次更新参数时按一定概率（rate）随机断开输入神经元，Dropout层用于防止过拟合。

- rate：0~1的浮点数，控制需要断开的神经元的比例

**Flatten层**

```
model = Sequential()
model.add(Convolution2D(64, 3, 3,
            border_mode='same',
            input_shape=(3, 32, 32)))
# now: model.output_shape == (None, 64, 32, 32)

model.add(Flatten())
# now: model.output_shape == (None, 65536)
```

Flatten层用来将输入“压平”，即把多维的输入一维化，常用在从卷积层到全连接层的过渡。Flatten不影响batch的大小。

border_mode 默认值可能就是 padding, 这样也蛮好的，保证了维度的一致性

**Reshape层**

```
# as first layer in a Sequential model
model = Sequential()
model.add(Reshape((3, 4), input_shape=(12,)))
# now: model.output_shape == (None, 3, 4)
# note: `None` is the batch dimension

# as intermediate layer in a Sequential model
model.add(Reshape((6, 2)))
# now: model.output_shape == (None, 6, 2)

# also supports shape inference using `-1` as dimension
model.add(Reshape((-1, 2, 2)))
# now: model.output_shape == (None, 3, 2, 2)
```

-1 表示让系统自己去算这个值是多少

**RepeatVector层**

RepeatVector层将输入重复n次

```
model = Sequential()
model.add(Dense(32, input_dim=32))
# now: model.output_shape == (None, 32)
# note: `None` is the batch dimension

model.add(RepeatVector(3))
# now: model.output_shape == (None, 3, 32)
```

**ActivityRegularizer层**

```
keras.layers.core.ActivityRegularization(l1=0.0, l2=0.0)
```


经过本层的数据不会有任何变化，但会基于其激活值更新损失函数值

## 卷积层

**Conv2D**

```
keras.layers.convolutional.Conv2D(filters, kernel_size, strides=(1, 1), 
    padding='valid', data_format=None, dilation_rate=(1, 1), 
    activation=None, use_bias=True, kernel_initializer='glorot_uniform', 
    bias_initializer='zeros', kernel_regularizer=None, bias_regularizer=None, 
    activity_regularizer=None, kernel_constraint=None, bias_constraint=None)
```

- 卷积核的数目（即输出的维度）
- 单个整数或由两个整数构成的list/tuple，卷积核的宽度和长度


## 池化层

```
keras.layers.pooling.MaxPooling2D(pool_size=(2, 2), strides=None, padding='valid', data_format=None)
```

- pool_size：整数或长为2的整数tuple，代表在两个方向（竖直，水平）上的下采样因子，如取（2，2）将使图片在两个维度上均变为原长的一半。为整数意为各个维度值相同且为该数字。
- strides：整数或长为2的整数tuple，或者None，步长值。

输入输出都是 4D 张量

(samples，channels, rows，cols)

## 循环层

循环层有 LSTM, GRU 层之分

**LSTM**

```
keras.layers.recurrent.LSTM(units, activation='tanh', recurrent_activation='hard_sigmoid', 
    use_bias=True, kernel_initializer='glorot_uniform', recurrent_initializer='orthogonal', 
    bias_initializer='zeros', unit_forget_bias=True, kernel_regularizer=None, 
    recurrent_regularizer=None, bias_regularizer=None, activity_regularizer=None, 
    kernel_constraint=None, recurrent_constraint=None, bias_constraint=None, dropout=0.0, 
    recurrent_dropout=0.0)
```

- units：输出维度
- activation：激活函数，为预定义的激活函数名（参考激活函数）
- recurrent_activation: 为循环步施加的激活函数（参考激活函数）
- dropout：0~1之间的浮点数，控制输入线性变换的神经元断开比例
- input_dim：输入维度，当使用该层为模型首层时，应指定该值（或等价的指定input_shape)
- input_length：当输入序列的长度固定时，该参数为输入序列的长度。当需要在该层后连接Flatten层，然后又要连接Dense层时，
  需要指定该参数，否则全连接的输出无法计算出来。注意，如果循环层不是网络的第一层，你需要在网络的第一层中指
  定序列的长度（通过input_shape指定）。


输入:

形如（samples，timesteps，input_dim）的3D张量

输出

如果return_sequences=True：返回形如（samples，timesteps，output_dim）的3D张量

否则，返回形如（samples，output_dim）的2D张量

```
# as the first layer in a Sequential model
model = Sequential()
model.add(LSTM(32, input_shape=(10, 64)))
# now model.output_shape == (None, 32)
# note: `None` is the batch dimension.

# the following is identical:
model = Sequential()
model.add(LSTM(32, input_dim=64, input_length=10))

# for subsequent layers, no need to specify the input size:
         model.add(LSTM(16))

# to stack recurrent layers, you must use return_sequences=True
# on any recurrent layer that feeds into another recurrent layer.
# note that you only need to specify the input size on the first layer.
model = Sequential()
model.add(LSTM(64, input_dim=64, input_length=10, return_sequences=True))
model.add(LSTM(32, return_sequences=True))
model.add(LSTM(10))
```


## 嵌入层 Embedding

```
keras.layers.embeddings.Embedding(input_dim, output_dim, embeddings_initializer='uniform', 
    embeddings_regularizer=None, activity_regularizer=None, embeddings_constraint=None, 
    mask_zero=False, input_length=None)
```


嵌入层将正整数（下标）转换为具有固定大小的向量，如[[4],[20]]->[[0.25,0.1],[0.6,-0.2]] (这句话没看懂)

Embedding层只能作为模型的第一层

- input_dim：大或等于0的整数，字典长度(大小)，即输入数据最大下标+1
- output_dim：大于0的整数，代表全连接嵌入的维度
- input_length：当输入序列的长度固定时，该值为其长度。如果要在该层后接Flatten层，然后接Dense层，则必须指定该参数，
  否则Dense层的输出维度无法自动推断。

注：一般在用这个层之前需要先把 input_dim 也就是 vocabulary size 定下来

**输入shape**

形如（samples，sequence_length）的2D张量

**输出shape**

形如(samples, sequence_length, output_dim)的3D张量

```
model = Sequential()
model.add(Embedding(1000, 64, input_length=10))
# the model will take as input an integer matrix of size (batch, input_length).
# the largest integer (i.e. word index) in the input should be no larger than 999 (vocabulary size).
# now model.output_shape == (None, 10, 64), where None is the batch dimension.

# 这行返回的是 32 * 10 的矩阵，矩阵中的每个值都小于 1000？
input_array = np.random.randint(1000, size=(32, 10))

model.compile('rmsprop', 'mse')
output_array = model.predict(input_array)
assert output_array.shape == (32, 10, 64)
```

## 序列预处理

**填充序列pad_sequences**

```
keras.preprocessing.sequence.pad_sequences(sequences, maxlen=None, dtype='int32',
    padding='pre', truncating='pre', value=0.)
```

没遇到过，待处理

**跳字 skipgrams**

```
keras.preprocessing.sequence.skipgrams(sequence, vocabulary_size,
    window_size=4, negative_samples=1., shuffle=True,
    categorical=False, sampling_table=None)
```

skipgrams将一个词向量下标的序列转化为下面的一对tuple：

- 对于正样本，转化为（word，word in the same window）
- 对于负样本，转化为（word，random word from the vocabulary）

【Tips】根据维基百科，n-gram代表在给定序列中产生连续的n项，当序列句子时，每项就是单词，此时n-gram也称为shingles。而skip-gram的推广，skip-gram产生的n项子序列中，各个项在原序列中不连续，而是跳了k个字。例如，对于句子：

“the rain in Spain falls mainly on the plain”

其 2-grams为子序列集合：

the rain，rain in，in Spain，Spain falls，falls mainly，mainly on，on the，the plain

其 1-skip-2-grams为子序列集合：

the in, rain Spain, in falls, Spain mainly, falls on, mainly the, on plain.

## 文本预处理
   

**句子分割text_to_word_sequence**

```
keras.preprocessing.text.text_to_word_sequence(text,
                                               filters='!"#$%&()*+,-./:;<=>?@[\]^_`{|}~\t\n',
                                               lower=True,
                                               split=" ")
```

**one-hot编码**

```
keras.preprocessing.text.one_hot(text,
                                 n,
                                 filters='!"#$%&()*+,-./:;<=>?@[\]^_`{|}~\t\n',
                                 lower=True,
                                 split=" ")
```