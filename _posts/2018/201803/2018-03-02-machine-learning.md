---
layout: post
title:  "机器学习相关算法"
date:   2018-03-02 15:13:55 +0800
categories: 其他
tags: python
---

* TOC
{:toc}

# 朴素贝叶斯
1. 将要处理的数据转换为数字
2. 分别计算待测试的文档中的每个单词，出现在不同类别的概率。
对于一个包含字符串的数组：`test_list=['problems', 'buying', 'him', 'has', 'quit', 'how', 'steak', 'help', 'please', 'flea', 'mr', 'ate', 'garbage', 'worthless', 'love', 'maybe', 'I', 'posting', 'licks', 'stupid', 'not', 'food', 'to', 'dog', 'so', 'is', 'cute', 'dalmation', 'park', 'my', 'take', 'stop']`
假定有两个类别：0和1.要判断该一个字符串数组属于类别0，会给出字符串数组。
~~~
    sample_list = [['my', 'dog', 'has', 'flea', 'problems', 'help', 'please'],
                    ['maybe', 'not', 'take', 'him', 'to', 'dog', 'park', 'stupid'],
                    ['my', 'dalmation', 'is', 'so', 'cute', 'I', 'love', 'him'],
                    ['stop', 'posting', 'stupid', 'worthless', 'garbage'],
                    ['mr', 'licks', 'ate', 'my', 'steak', 'how', 'to', 'stop', 'him'],
                    ['quit', 'buying', 'worthless', 'dog', 'food', 'stupid']]
    class_vec = [0, 1, 0, 1, 0, 1]
~~~
对给定的字符串数组test_list,需要判定的是其中的每个单词分别出现在类别0和1中的概率。
具体怎么计算呢？在第1步中，test_list和sample_list做比对后，获得一个矩阵：

~~~
matrix=
[[1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, 1, 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0], 
[0, 1, 0, 0, 0, 0, 1, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0], 
[0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 1], 
[0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0], 
[0, 0, 1, 0, 1, 0, 0, 0, 0, 1, 0, 1, 0, 1, 0, 1, 0, 0, 0, 0, 0, 0, 1, 0, 0, 1, 0, 0, 0, 0, 1, 0], 
[0, 0, 0, 0, 0, 1, 1, 1, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 0, 0, 0]]
~~~
python：

~~~
def set_of_words2_vec(vocab_list, input_set):
    return_vec = [0] * len(vocab_list)
    for word in input_set:
        if word in vocab_list:
            return_vec[vocab_list.index(word)] = 1
        else:
            print("the wojrd:%s is not in my vocabulary", word)
    return return_vec

train_mat = []
for postin_doc in sample_list:
    train_mat.append(set_of_words2_vec(test_list, postin_doc))
~~~

这个矩阵和数组class_vec中的类别对应，以matrix[0]为例，matrix[0]对应class_vec[0],即matrix[0]中，被标记为1的字符串被判定为在类别0中出现。
接下来计算针对类别0，计算test_list中的每个单词出现在类别0中的概率。
取出matrix[0],matrix[2],matrix[4],计算其中的1出现的总次数`p0_num=24`，
`matrix[0]+matrix[2]+matrix[4]`获得每个单词在不同的类别0样本中出现的次数和`p0_vec=[0. 0. 1. 0. 1. 1. 0. 1. 0. 1. 0. 1. 1. 3. 1. 0. 1. 1. 1. 0. 1. 1. 1. 1. 0. 2. 1. 0. 1. 1. 0. 1.]`
那么要计算其中的每个单词属于类别0的概率就是`p0_vec/p0_num`,即：

~~~
[0.05263158 0.05263158 0.         0.05263158 0.05263158 0.
 0.05263158 0.05263158 0.05263158 0.05263158 0.05263158 0.05263158
 0.         0.05263158 0.05263158 0.         0.         0.05263158
 0.05263158 0.05263158 0.         0.05263158 0.15789474 0.
 0.05263158 0.         0.05263158 0.         0.10526316 0.
 0.05263158 0.        ]
~~~
同样的方式计算test_list中每个单词属于类别1的概率：

~~~
[0.         0.         0.05263158 0.05263158 0.         0.05263158
 0.         0.10526316 0.         0.         0.         0.
 0.05263158 0.         0.         0.05263158 0.05263158 0.
 0.         0.05263158 0.10526316 0.         0.         0.15789474
 0.         0.05263158 0.         0.05263158 0.05263158 0.05263158
 0.         0.05263158]
~~~

python:
~~~
def train_nb_0(train_mat, class_vec):
    num_train_docs = len(train_mat)
    num_words = len(train_mat[0])
    p_abusive = sum(class_vec) / float(num_train_docs)  # 代表划分为类别1的样本占总样本的比例（0.5）
    p0_num = zeros(num_words)
    p1_num = zeros(num_words)
    p0_denom = 0.0
    p1_denom = 0.0
    for i in range(num_train_docs):
        if class_vec[i] == 1:
            p1_num += train_mat[i]
            p1_denom += sum(train_mat[i])
        else:
            p0_num += train_mat[i]
            p0_denom += sum(train_mat[i])
    print(p0_num)
    print(p1_num)
    p1_vect = p1_num / p1_denom
    p0_vect = p0_num / p0_denom
    print(p0_vect)
    print(p1_vect)
    print(p_abusive)
    return p0_vect, p1_vect, p_abusive
~~~

## 算法修正
1. 为了计算概率的乘积，消除概率为0的情况
2. 为了避免过小的浮点数做乘积运算时丧失运算精度，将乘积换做对数的相加。即 a*b<=>loga+logb

使用bayes算法做分类器
原理：在上一个算法中，根据给定的实验样本，计算出了每个单词属于不同类别的概率。在对一个文档做分类的时候，也就知道了这个文档中的单词属于不同类别的概率，通过对这些概率做相乘（对数相加），获得这篇文档中所有单词分别属于类别0和1的概率，比较这两个概率大小来对文档做分类。
首先，对于给定的一篇文档，`test_entry = ['love', 'my', 'dalmation']`，将其转换为数组为：`test_vec=[0 0 0 0 0 0 0 0 0 1 0 0 0 0 0 0 1 0 0 0 0 0 0 0 0 1 0 0 0 0 0 0]`，这个test_vec数组的长度等于实验样本中包含的不重复的单词的数量，其中的1代表测试文档中的单词在实验样本中出现的次数。
通过上一个步骤，我们已经知道了对应于每个单词分别属于类别0和1的概率（p0_vec和p1_vec），所以p0_vec*test_vec可以用来衡量测试文档属于类别0的概率，sum（p0_vec*test_vec）也就是属于类别0的总概率。

python：

~~~
def train_nb_0(train_matrix, train_category):
    num_train_docs = len(train_matrix)
    num_words = len(train_matrix[0])
    p_abusive = sum(train_category) / float(num_train_docs)  # 代表划分为类别1的样本占总样本的比例（0.5）
    p0_num = ones(num_words)   #数组用1来填充
    p1_num = ones(num_words)
    p0_denom = 2.0
    p1_denom = 2.0
    for i in range(num_train_docs):
        if train_category[i] == 1:
            p1_num += train_matrix[i]
            p1_denom += sum(train_matrix[i])
        else:
            p0_num += train_matrix[i]
            p0_denom += sum(train_matrix[i])
    p1_vect = log(p1_num / p1_denom)    #对概率值取对数
    p0_vect = log(p0_num / p0_denom)
    return p0_vect, p1_vect, p_abusive

# p0_vec对应各个单词属于类别0的概率；vec2_classify对应实验样本转换为数字后的数组。p0和p1分别对应各个单词位于类别0和1的概率之和
def classify_nb(vec2_classify, p0_vec, p1_vec, p_class1):
    p1 = sum(vec2_classify * p1_vec) + log(p_class1)   以概率对数的相加替换概率的乘积
    p0 = sum(vec2_classify * p0_vec) + log(1.0 - p_class1)
    if p1 > p0:
        return 1
    else:
        return 0
~~~

# 逻辑回归






