---
layout: post
title: 推荐系统入门之Crab源码阅读(一)
date: 2015-12-03 23:30:00
categories: 算法
tags: 推荐系统 Python Crab
---
# 推荐系统入门之Crab源码阅读(一)

## 前言
最近需要搞一下推荐系统, 正好一直以来都对推荐算法比较感兴趣。那么怎么入门呢, 从阅读开源的推荐系统开始是个不错的方法.
Crab是一个轻量级的推荐算法库, 使用python写成, 代码比较简洁易读, 正好通过学习它来对推荐系统有个初步的了解。Crab 实现了item和user的协同过滤, 项目地址https://github.com/muricoca/crab, 不过目前该项目已经不再更新维护。

* Recommender Algorithms: User-Based Filtering and Item-Based Filtering
* Work in progress: Slope One, SVD, Evaluation of Recommenders.
* Planed: Sparse Matrices, REST API’s.

关于Crab的源码学习将分为两篇:

* 第一篇简单介绍推荐算法和基于Item-Based的协同过滤算法
* 第二篇将介绍基于User-Based的协同过滤算法以及源码的其他内容。

## 推荐系统简介

### 搜索系统与推荐系统的区别

* 搜索系统解决的场景是用户对自己需求相对明确的时候通过关键字很快搜索到自己需要的信息
* 推荐系统解决的场景是用户在并不明确自己的需求时候或者他们的需求很难用关键字表达时候通过他们的习惯性来猜测用户需要的

### 推荐系统分类

现在主流的推荐系统是以模型的为主, 根据模型的建立方式可以分为以下:

* 基于物品和用户本身的，这种推荐引擎将每个用户和每个物品都当作独立的实体，预测每个用户对于每个物品的喜好程度，这些信息往往是用一个二维矩阵描述的。由于用户感兴趣的物品远远小于总物品的数目，这样的模型导致大量的数据空置，即我们得到的二维矩阵往往是一个很大的稀疏矩阵。同时为了减小计算量，我们可以对物品和用户进行聚类， 然后记录和计算一类用户对一类物品的喜好程度，但这样的模型又会在推荐的准确性上有损失。
* 基于关联规则的推荐（Rule-based Recommendation）：关联规则的挖掘已经是数据挖掘中的一个经典的问题，主要是挖掘一些数据的依赖关系，典型的场景就是“购物篮问题”，通过关联规则的挖掘，我们可以找到哪些物品经常被同时购买，或者用户购买了一些物品后通常会购买哪些其他的物品，当我们挖掘出这些关联规则之后，我们可以基于这些规则给用户进行推荐。
* 基于模型的推荐（Model-based Recommendation）：这是一个典型的机器学习的问题，可以将已有的用户喜好信息作为训练样本，训练出一个预测用户喜好的模型，这样以后用户在进入系统，可以基于此模型计算推荐。这种方法的问题在于如何将用户实时或者近期的喜好信息反馈给训练好的模型，从而提高推荐的准确度。

本文主要介绍基于物品和用户本身的推荐系统即协同过滤.

### 基于用户的推荐（User-based Recommendation）
基于用户对物品的偏好找到相邻邻居用户，然后将邻居用户喜欢的推荐给当前用户。计算上，就是将一个用户对所有物品的偏好作为一个向量来计算用户之间的相似度，找到 K 邻居后，根据邻居的相似度权重以及他们对物品的偏好，预测当前用户没有偏好的未涉及物品，计算得到一个排序的物品列表作为推荐。图1 给出了一个例子，对于用户 A，根据用户的历史偏好，这里只计算得到一个邻居 - 用户 C，然后将用户 C 喜欢的物品 D 推荐给用户 A。

![img](../image/usercf.gif)

### 基于项目的推荐（Item-based Recommendation）
类似，只是在计算邻居时采用物品本身，而不是从用户的角度，即基于用户对物品的偏好找到相似的物品，然后根据用户的历史偏好，推荐相似的物品给他。从计算的角度看，就是将所有用户对某个物品的偏好作为一个向量来计算物品之间的相似度，得到物品的相似物品后，根据用户历史的偏好预测当前用户还没有表示偏好的物品，计算得到一个排序的物品列表作为推荐。图2 给出了一个例子，对于物品 A，根据所有用户的历史偏好，喜欢物品 A 的用户都喜欢物品 C，得出物品 A 和物品 C 比较相似，而用户 C 喜欢物品 A，那么可以推断出用户 C 可能也喜欢物品 C。
![img](../image/itemcf.gif)

## 协同过滤实践

要实现一个协同过滤的推荐系统需要以下几步:

* 收集用户偏好
* 找到相似的用户或物品
* 计算推荐

以下是使用Crab构建简单的协同过滤的代码, 测试数据来自于电源数据http://archive.ics.uci.edu/ml 的10万次评论。

### 推荐代码示例
以Item-based CF 为例
{% highlight python linenos %}
#简单的基于movie Data 的item cf
from pandas import DataFrame
from xmlife_statistics.utils.crab.models.datamodel import DictDataModel
from xmlife_statistics.utils.crab.neighborhood.itemstrategies import PreferredItemsNeighborhoodStrategy
from xmlife_statistics.utils.crab.recommender.recommender import ItemRecommender
from xmlife_statistics.utils.crab.similarities.similarity import ItemSimilarity
from xmlife_statistics.utils.crab.similarities.similarity_distance import sim_euclidian
#将u.data转成DataFrame
datapath="/Users/admin/Workspace/jupyter/data/ml-100k/u.data"
with open(datapath,'r') as f:
    user_items_rate = map(lambda x:x.strip().split('\t')[0:3],f.readlines())
    df_user_item_rate = DataFrame(user_items_rate,columns=['user_id','item_id','rate'])
#将数据转成 {userID: {itemID:preference, itemID2:preference2},userID2:{itemID:preference3, itemID4:preference5}}
item_data={}
for item, group in df_user_item_rate.groupby('user_id'):
    item_data[item] = dict(zip(group['item_id'],map(lambda x:int(x),group['rate'])))
#收集用户偏好
model = DictDataModel(item_data)
#设置相似计算算法
simillarity = ItemSimilarity(model, sim_euclidian)
#设置选邻居策略
strategy = PreferredItemsNeighborhoodStrategy()
#计算推荐
recSys = ItemRecommender(model, simillarity, strategy, True)
#获取最相似的top 10
items = recSys.mostSimilarItems(['1'], 10)
print(items)
{% endhighlight python %}

我们通过这一例子为入口来学习下Crab源码以及学习下基于Item-based 协同过滤算法
### 收集用户偏好
Crab使用了DictDataModel来存放收集的用户数据, 数据格式为
{userID: {itemID:preference, itemID2:preference2},
userID2:{itemID:preference3, itemID4:preference5}}
在这个例子中就是:
{用户A:{电影A:打分3, 电影B:打分5},
用户B:{电影B:打分4, 电影C:打分5, 电影D:打分3}}

DictDataModel 最主要的方法有以下几个
#### 1. buildMode
{% highlight python linenos %}
def buildModel(self):
    ''' Build the model '''
    self.userIDs = self.dataU.keys()
    self.userIDs.sort()

    #这些用户所涉及到的item, 建立一个完整的item向量
    self.itemIDs = []
    for userID in self.userIDs:
        items = self.dataU[userID]
        self.itemIDs.extend(items.keys())
    self.itemIDs = list(set(self.itemIDs))
    self.itemIDs.sort()

    #最大和最小的打分
    self.maxPref = -100000000
    self.minPref = 100000000

    self.dataI = {}
    for user in self.dataU:
        for item in self.dataU[user]:
            self.dataI.setdefault(item, {})
            #将以user为key的dict转换成以item为key的dict,如{电影A:{用户A:打分3}, #电影B:{用户A:打分5,用户B:打分4},...}
            self.dataI[item][user] = self.dataU[user][item]
            if self.dataU[user][item] > self.maxPref:
                self.maxPref = self.dataU[user][item]
            if  self.dataU[user][item] < self.minPref:
                self.minPref = self.dataU[user][item]
{% endhighlight python %}

#### 2. PreferencesForItem
{% highlight python linenos %}
def PreferencesForItem(self, itemID, orderByID=True):
    """
    根据itemId获取它的user向量
    """
    itemPrefs = self.dataI.get(itemID, None)

    if not itemPrefs:
        raise ValueError(
                'User not found. Change for a suitable exception here!')

    itemPrefs = itemPrefs.items()

    if not orderByID:
        itemPrefs.sort(key=lambda itemPref: itemPref[1], reverse=True)
    else:
        itemPrefs.sort(key=lambda itemPref: itemPref[0])

    return itemPrefs
{% endhighlight python %}

#### 3. PreferencesFromUser
{% highlight python linenos %}
def PreferencesFromUser(self, userID, orderByID=True):
        """
        根据userId获取它的itemId向量
        """
        userPrefs = self.dataU.get(userID, None)

        if userPrefs is None:
            raise ValueError(
                    'User not found. Change for a suitable exception here!')

        userPrefs = userPrefs.items()

        if not orderByID:
            userPrefs.sort(key=lambda userPref: userPref[1], reverse=True)
        else:
            userPrefs.sort(key=lambda userPref: userPref[0])

        return userPrefs
{% endhighlight python %}

由此可见, DictDataModel主要负责构建基于user的Item向量 和 基于Item的user向量, 分别为User-Based和Item-based计算用户和物品的相似度。
### 找到相似的用户或物品
找到相似的用户或物品分为两步:

* 计算相似度: simillarity = ItemSimilarity(model, sim_euclidian)
* 获取可能邻居策略: strategy = PreferredItemsNeighborhoodStrategy()

#### 计算相似度
Item-based CF的similarity是由ItemSimilarity类实现的, 他主要用到两个方法。
{% highlight python linenos %}
def getSimilarity(self, vec1, vec2):
    """
    根据item的user向量,计算两个item的相似度
    """
    item1Prefs = dict(self.model.PreferencesForItem(vec1))
    item2Prefs = dict(self.model.PreferencesForItem(vec2))

    # Evaluate the similarity between the two users vectors.
    return self.distance(item1Prefs, item2Prefs)
def getSimilarities(self, vec):
    """
    根据item的user向量, 以此计算item与其他的item的相似度
    """
    return [(other, self.getSimilarity(vec, other))
            for other in self.model.ItemIDs()]
{% endhighlight python %}

* 需要注意的是, 通过PreferencesForItem获取的user向量, 由于只存放有打分过的user(不打分的user缺省), 所以多个item之间的向量长度以及user对应的值是不同的, 因此计算时只取都存在两个向量内的user。
如sum_of_squares = sum([pow(vector1[item] - vector2[item], 2.0) for item in vector1 if item in vector2])

相似度的求法很多

* 欧几里德距离 - Euclidean-distance
* 皮尔逊相关系数 － Pearson Correlation Coefficient
* Cosine 相似度 -Cosine Similarity
* Tanimoto 系数 - Tanimoto Coefficient
* Jaccard系数 - Jaccard similarity coefficient
* manhattan系数 - Manhattan distance
* Sørensen’s similarity coefficient
* log-likelihood ratio score

这里只介绍几种常用的, 其他的实现就省略了, 详见源码。

##### 欧几里德距离（Euclidean Distance）
最初用于计算欧几里德空间中两个点的距离，假设 x，y 是 n 维空间的两个点，它们之间的欧几里德距离是：
![img](../image/Euclidean-Distance.gif)
可以看出，当 n=2 时，欧几里德距离就是平面上两个点的距离。
当用欧几里德距离表示相似度，一般采用以下公式进行转换：距离越小，相似度越大
![img](../image/Euclidean-Distance2.gif)
##### 皮尔逊相关系数（Pearson Correlation Coefficient）
皮尔逊相关系数一般用于计算两个定距变量间联系的紧密程度，它的取值在 [-1，+1] 之间。
![img](../image/Pearson.gif)
sx, sy是 x 和 y 的样品标准偏差。
##### Cosine 相似度（Cosine Similarity）
Cosine 相似度被广泛应用于计算文档数据的相似度：
![img](../image/Cosine.gif)
夹角越小越相似
##### Jaccard 系数
![img](../image/Jaccard.gif)

#### 获取潜在邻居策略
{% highlight python linenos %}
class PreferredItemsNeighborhoodStrategy(CandidateItemsStrategy):
    '''
    Returns all items that have not been rated by the user and that were
    preferred by another user that has preferred at least one item that the
    current user has preferred too
    '''

    def candidateItems(self, userID, model):
        possibleItemIDs = []
        itemIDs = model.ItemIDsFromUser(userID)
        for itemID in itemIDs:
            prefs2 = model.PreferencesForItem(itemID)
            for otherUserID, pref in prefs2:
                possibleItemIDs.extend(model.ItemIDsFromUser(otherUserID))

        possibleItemIDs = list(set(possibleItemIDs))

        return [itemID for itemID in possibleItemIDs if itemID not in itemIDs]
{% endhighlight python %}
方法很简单, 它假设:

* 如果User-B对应的Items包含了User-A的某个Item, 那么User-B就是User-A的邻居, 那么User-B有而User-A没有的item就有可能是User-A的item的相似item。这样就减少了User-A推荐范围, 不需要对所有item计算相似度。
* 我们反过来想, 之前Item-Based CF的协同过滤算法中提到, 计算item的相似度只要计算item对应的User向量的相似度就行, 如果两个item间没有user重合, 那么肯定不可能是相似的.

### 计算推荐
在Crab内有三种推荐方式分别是:ItemRecommender, UserRecommender, SlopeOneRecommender, 本文只介绍ItemRecommender, 其他两种将在下文介绍.

ItemRecommender 有两种用法, 1: 推荐user相似item, 2: 推荐item的相似items

#### 推荐user相似item
该场景是根据user A的历史习惯, 推荐user A没有的其他item:
{% highlight python linenos %}
def recommend(self, userID, howMany, rescorer=None):
    if self.numPreferences(userID) == 0:
        return []

    # 获取user其他的可能相似的item
    possibleItemIDs = self.allOtherItems(userID)

    # 计算user的历史item 与 可能相似的item逐一进行相似计算并获取最相似的几个
    rec_items = topItems(userID, possibleItemIDs, howMany,
            self.estimatePreference, self.similarity, rescorer)

    return rec_items

def allOtherItems(self, userID):
    return self.strategy.candidateItems(userID, self.model)
{% endhighlight python %}

#### 推荐item的相似items
该场景是根据一个或几个item, 计算与它/它们相似的items
{% highlight python linenos %}
def mostSimilarItems(self, itemIDs, howMany, rescorer=None):
    possibleItemIDs = []

    # 根据item对应的user,找出这些item潜在的相似items
    for itemID in itemIDs:
        prefs = self.model.PreferencesForItem(itemID)
        for userID, pref in prefs:
            possibleItemIDs.extend(self.model.ItemIDsFromUser(userID))
    # 去重
    possibleItemIDs = list(set(possibleItemIDs))

    pItems = [itemID for itemID in possibleItemIDs
              if itemID not in itemIDs]

    return topItems(itemIDs, pItems, howMany,
            self.estimateMultiItemsPreference, self.similarity, rescorer)
{% endhighlight python %}

多个item与某个item计算相似度
{% highlight python linenos %}
def estimateMultiItemsPreference(self, **args):
    toItemIDs = args.get('thingID', None)
    itemID = args.get('itemID', None)
    similarity = args.get('similarity', self.similarity)
    rescorer = args.get('rescorer', None)

    sum = 0.0
    total = 0

    for toItemID in toItemIDs:
        preference = similarity.getSimilarity(itemID, toItemID)

        rescoredPref = rescorer.rescore((itemID, toItemID), preference) \
                            if rescorer else preference

        sum += rescoredPref
        total += 1

    return sum / total
{% endhighlight python %}

### 测试验证

运行以上demo, 找出item '1' 最相似的 10个 item 为 ['1627', '1628', '1507', '1504', '1616', '1549', '1554', '1123', '1238', '1484']

运行一下代码查看电影名
{% highlight python linenos %}
datapath="/Users/admin/Workspace/jupyter/data/ml-100k/u.item"
with open(datapath,'r') as f:
    item_info = map(lambda x:x.strip().split('|')[0:2],f.readlines())
    df_items = DataFrame(item_info,columns=['movie_id','movie_title'])
print(df_items[df_items['movie_id']=='1'])
df_items[df_items['movie_id'].isin(['1627', '1628', '1507', '1504', '1616', '1549', '1554', '1123', '1238', '1484'])]

movie_id       movie_title
0        1  Toy Story (1995)
movie_id	movie_title
1122	1123	Last Time I Saw Paris, The (1954)
1237	1238	Full Speed (1996)
1483	1484	Jerky Boys, The (1994)
1503	1504	Bewegte Mann, Der (1994)
1506	1507	Three Lives and Only One Death (1996)
1548	1549	Dream Man (1995)
1553	1554	Safe Passage (1994)
1615	1616	Desert Winds (1995)
1626	1627	Wife, The (1995)
1627	1628	Lamerica (1994)
{% endhighlight python %}

## 总结
本文主要简单介绍了推荐系统的一些概念, 以及结合Crab源码简单介绍了基于Item-Based的协同过滤实现。

引用:

http://www.ibm.com/developerworks/cn/web/1103_zhaoct_recommstudy1/
http://www.ibm.com/developerworks/cn/web/1103_zhaoct_recommstudy2/

本文完
