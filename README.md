# 实体Embedding

### 任务
给定知识图谱的全部三元组，对其中的实体进行embedding
### 输入
整个知识图谱
### 输出
每一个实体一个向量。要求：
* 应可用于相似实体寻找
* 用于NLP任务的预训练

---
## 了解数据
格式上整个数据都是`(h,r,t)`的格式，用`\t`分割，每一个三元组一行。

数据量较大，统计结果如下：

* head——1,857,754个
* relation——135,818个
* tails——12,718,355个

分别取三者的top-10，结果为：
```
* head
  * ('中华人民共和国合同法（法律）', 380),
  * ('中华人民共和国证券法［已被修正］（法律）', 232),
  * ('杭州工联大厦股份有限公司', 216),
  * ('海南经济特区股份有限公司条例［修正］（法律）', 186),
  * ('深圳经济特区股份有限公司条例［已被修正］（法律）', 179),
  * ('中国证券交易系统有限公司业务规则（法律）', 175),
  * ('市政府关于颁发《常州市行政执法程序暂行规定》的通知 （法律）', 175),
  * ('中华人民共和国食品安全法(2015全文）（法律）', 170),
  * ('上海市股份有限公司暂行规定［失效］（法律）', 161),
  * ('保险代理机构管理规定（法律）', 160)
* relation
  * ('CATEGORY_ZH', 3270304),
  * ('中文名', 1089688),
  * ('DESC', 968843),
  * ('国籍', 255581),
  * ('歌手', 248855),
  * ('长度', 243576),
  * ('专辑', 243184),
  * ('出生日期', 233904),
  * ('周围景观', 226947),
  * ('职业', 200549)
* tail
  * ('人物', 335246),
  * ('<a>中国</a>', 197326),
  * ('地点', 169037),
  * ('组织机构', 145678),
  * ('<a>汉族</a>', 85977),
  * ('字词', 83198),
  * ('<a>男</a>', 77419),
  * ('中国其他行政区划', 69635),
  * ('游戏', 64987),
  * ('村庄', 62190)
```

可见出现频率最高的10个实体/关系中数量差距也是十分巨大的，因此可以想象出现频率较低的实体/关系可能得不到有效的训练。

在观察数据的时候还发现很多问题，以下一一列举：

### 1. 停用词
上面的tail中已经出现了很多`<a>`和`</a>`的HTML标签，这种标签的存在会影响训练出来的embedding在NLP中的作用，应该剔除。

### 2. DESC
关系“DESC”对应的tail是对head的描述。比如：

    '塘溪源村', 'DESC', '临安市湍口镇塘溪源村\xa0位于湍口镇西北部，村域面积14.913平方公里，于2007年11月由塘溪、岔口两村合并而成，有8个村民小组，驻地岔口。全村共285户，1035人。山多地少,现有耕地374.4亩，山林面积22544亩，其中用材林16398亩，经济 林6146亩。2008年村级集体经济总收入为49万元，农民人均收入为6882元。'

显然这样的tail不适合训练，因为其不是简单的一个实体，对其训练没有任何意义。并且描述性语句几乎不可能相同，所以出现频率极低，无法训练出合适的语义。因此不妨用一个特殊的符号代替冗杂的描述，比如用`<DESC>`表示。__或者干脆忽略relation为`DESC`的记录。__

> 以上两个操作在`preprocess.py/convert_kg`里实现，经过初步处理，KG文件大小从986 MB降到了639 MB。

### 3. 度量信息
实体中存在大量度量信息，也包含很多数字，这些度量形式多样，用简单的替换策略需要先找到所有形式，效率不高。这类信息有`长度`、`出生日期`等：

    'Give Her to Me（RebeccaMader演唱歌曲）', '长度', '1分18秒'
    'Stick to Your Guns（Dokken演唱歌曲）', '长度', '3分23秒'
    '佳能FAX-JX510P', '复印速度', '4.5cpm*4'
    '佳能FAX-JX510P', '复印分辨率', '600dpi×600dpi'
    '德格印经院藏族雕版印刷技艺', '时间', '2006年'
    '余其平', '出生日期', '1962年11月'
    '乔治·穆耶姆', '出生日期', '1971年04月15日'
    '后岔湾村', '人口', '726人'

和DESC相同，这些数字相同的概率低，且这个任务的目的是训练Embedding，而不要求构建一个精确的KG，用算法来区分这些数字不是主要目的，因此思路是先筛选出tail中存在数字的relation，然后对这些数据进行统一处理。
> 类似方法：聚类

具体来讲该方法就是维护一个dict，从头开始遍历KG，当遇到一个三元组中tail中含有数字的时候，就将这个relation记录到dict中（relation作为键，出现次数作为值）。最后将dict中的键根据值的大小降序排序，排在前几位的几乎可以确定就是“度量关系”了。

有了“度量关系”，就可以将包含“度量关系”的tail替换成为统一的标记。这样做就能大大提高实体出现的频率，对网络训练应该是起正向作用。

---
## 数据清洗
根据上面的观察结果，对数据可以做以下预处理：
1. 剔除类似`<a>`或`</a>`的停用词
2. relation为`DESC`的tail替换为`<DESC>`。或者干脆去除该三元组
3. 时间数据、地点数据的处理

当然这些数据还有一些问题，比如下面的数据：
```
'中国科学院鹤山丘陵综合试验站', '中文名', '<a>中国科学院鹤山丘陵综合试验站</a>'
'鄂州市鄂钢第二小学', '中文名', '<a>鄂州市鄂钢第二小学</a>'
```
剔除`<a>`和`</a>`之后head和tail是相同的，那么此时的`中文名`就是0向量。这会不会影响到Embedding在NLP中的作用？


---
## 网络权值的初始化
权值初始化是十分重要的。

