# 农业知识图谱
github地址：https://github.com/mattzheng/Agriculture_KnowledgeGraph
github开源中的PPT也是极好的，可以帮助了解关键核心以及大致框架，感谢作者的开源。

整个知识图谱的结构图：
![此处输入图片的描述](https://github.com/mattzheng/Agriculture_KnowledgeGraph/blob/master/matt/%E7%9F%A5%E8%AF%86%E5%9B%BE%E8%B0%B1-V1.0.bmp)
笔者比较关注'实体词识别-知识获取-关系抽取-关系归纳'，这个过程之中。其中的知识获取相当于无监督的关系抽取过程，而关系抽取的PCNNs是在知识获取的基础上的无监督一个步骤。
以上的两个都是归纳'实体-实体'之间的关联，关系归纳，则是归纳'实体-类别'之间的联系。
图数据库存储基本上来看，都会放在图数据库之中，导入csv的规范可见如下。

## 一、如何使用neo4j的docker版本
修改内存，先行打开neo4j-docker：

    sudo docker run \
    --publish=7474:7474 --publish=7687:7687 \
    --volume=/data1/neo4j:/var/lib/neo4j/import \
    --rm -ti neo4j\
    neo4j
把csv放到/data1/neo4j就可以直接导入csv之中了。
在`/neo4j/conf/neo4j.conf`之中修改以下参数：

    dbms.memory.heap.initial_size = 1024G 
    dbms.memory.heap.max_size= 1024G
    dbms.memory.pagecache.size = 10240M # 缓存，可以调制到一些

然后启动neo4j

    ./neo4j start

打开之后需要等待一段时间的启动。然后在浏览器输入：`http://localhost:7474/browser/`


----------


## 一、图数据库

总结一下一般可以定位为以下三个表格形式：节点表（节点 + 节点属性）、节点关系/属性表

 - 节点表（节点 + 节点属性）

node | attr  
: | :-: 
节点1 | 节点属性

可以用neo4j：

    LOAD CSV WITH HEADERS  FROM "file:///node.csv" AS line  
    CREATE (p:nodeItem{title:line.node,image:line.attr})  

 - 节点关系表

node1 | relation | node2
: | :-: | -: 
节点1 | 关系/属性 | 节点2 

可以用neo4j：

    LOAD CSV WITH HEADERS FROM "file:///attributes.csv" AS line
    MATCH (entity1:nodeItem{title:line.node1}), (entity2:nodeItem{title:line.node2})
    CREATE (entity1)-[:RELATION { type: line.relation }]->(node2);


----------


### 1.1 数据结构
**实体词解释库：hudong_pedia.csv + hudong_pedia2.csv**

    "title","url","image","openTypeList","detail","baseInfoKeyList","baseInfoValueList"
    "冀香糯1号","http://www.baike.com/wiki/冀香糯1号","http://a4.att.hudong.com/87/06/20300543526108144636060022320_s.gif","农业##植物",
    "冀香糯1号，株高111cm，分蘖力强，穗长18.6cm，每穗粒数150.4个，结实率83%，千粒重29.9 g。生育期169天左右。2008年农业部谷物品质监督检验测试
    中心检测，籽粒粗蛋白9.83%，糙米率82.5%，精米率74.3%，整精米率38%，胶稠度99mm，碱消值6，长宽比2.0。建议在河北省秦皇岛、
    唐山市稻区作糯性专用品种种植。","",""

neo4j的创建语句为：

    // 将hudong_pedia.csv 导入
    LOAD CSV WITH HEADERS  FROM "file:///hudong_pedia.csv" AS line  
    CREATE (p:HudongItem{title:line.title,image:line.image,detail:line.detail,url:line.url,openTypeList:line.openTypeList,baseInfoKeyList:line.baseInfoKeyList,baseInfoValueList:line.baseInfoValueList})  

词条名称title、词条互动百科URL，词条代表图片image，词条类型openTypeList,词条解释detail，还有两个："baseInfoKeyList","baseInfoValueList"

**导入新节点new_node.csv**
title,lable
药物治疗,newNode
新实体词名称title，新实体词标签label

neo4j的查询语句为：

    // 导入新的节点
    LOAD CSV WITH HEADERS FROM "file:///new_node.csv" AS line
    CREATE (:NewNode { title: line.title })
    
    //添加索引
    CREATE CONSTRAINT ON (c:NewNode)
    ASSERT c.title IS UNIQUE


**wikidata_relation2.csv  hudongItem和新加入节点之间的关系**

    HudongItem1,relation,HudongItem2
    菊糖,instance of,化合物
    菊糖,instance of,多糖

HudongItem1，实体词1；HudongItem2，实体词2l;relation是两个词的关系

    MATCH (entity1:HudongItem{title:line.HudongItem}) , (entity2:NewNode{title:line.NewNode})
    CREATE (entity1)-[:RELATION { type: line.relation }]->(entity2)

**attributes.csv 导入实体属性(数据来源: 互动百科)**

    Entity,AttributeName,Attribute
    密度板,别名,纤维板

    LOAD CSV WITH HEADERS FROM "file:///attributes.csv" AS line
    MATCH (entity1:HudongItem{title:line.Entity}), (entity2:HudongItem{title:line.Attribute})
    CREATE (entity1)-[:RELATION { type: line.AttributeName }]->(entity2);

Entity,AttributeName,Attribute分别代表，实体词x，属性名称，实体词y
跑了四段就是为了建立HudongItem-NewNode两两之间的节点，而且是由顺序的。



### 1.2 图数据库查询code

    from py2neo import Graph,Node,Relationship
    
    class Neo4j():
    	graph = None
    	def __init__(self):
    		print("create neo4j class ...")
    
    	def connectDB(self):
    		self.graph = Graph("http://localhost:7474", username="neo4j", password="qwer@1234")
    
    	def matchItembyTitle(self,value):
    		answer = self.graph.find_one(label="Item",property_key="title",property_value=value)
    		return answer
    
    	# 根据title值返回互动百科item
    	def matchHudongItembyTitle(self,value):
    		answer = self.graph.find_one(label="HudongItem",property_key="title",property_value=value)
    		return answer
    
    	# 根据entity的名称返回关系
    	def getEntityRelationbyEntity(self,value):
    		answer = self.graph.data("MATCH (entity1) - [rel] -> (entity2)  WHERE entity1.title = \"" +value +"\" RETURN rel,entity2")
    		return answer

这边从neo4j中主要查询的有三块内容：
matchItembyTitle：定位到实体value,getEntityRelationbyEntity，找到该实体的链接关系。


## 二、命名实体识别
利用thulac，来进行命名实体识别，并根据词性进行一定的筛选。

    import json
    import thulac
    import time
    # n
    # all+n  (all 不包含 vm)
    # a+n
    # np 人名
    # ns 地名
    # ni 机构名
    # nz 其它专名
    # t和r 时间和代词该步不用加，但是在命名实体识别时需要考虑（这里做个备注）
    # i 习语
    # j 简称
    # x 其它
    # 不能含有标点w
    	
    def nowok(s): #当前词的词性筛选
    	
    	if s=='n' or s=='np' or s=='ns' or s=='ni' or s=='nz':
    		return True
    	if s=='i' or s=='j' or s=='x' or s=='id' or s=='g' or s=='t':
    		return True
    	if s=='t' or s=='m':
    		return True
    	return False	
    
    def judge(s):  #含有非中文和英文，数字的词丢弃
    	num_count = 0
    	for ch in s:
    		if u'\u4e00' <= ch <= u'\u9fff':
    			pass
    		elif '0' <= ch <= '9':
    			num_count += 1
    			pass
    		elif 'a' <= ch <= 'z':
    			pass
    		elif 'A' <= ch <= 'Z':
    			pass
    		else:
    			return False
    	if num_count == len(s):  ##如果是纯数字，丢弃
    		return False
    	return True
    
    # 给定分词结果，提取NER
    def createWordList(x):
    	i = 0
    	n = len(x)
    	L = []
    	while i < n:
    		if judge(x[i][0]) == False :
    			i += 1
    			continue;
    		if nowok(x[i][1]):  
    			L.append(x[i][0])
    			
    		i += 1
    		
    	return L
    			
    
    def thulac_ner(text):
        return createWordList(thu.cut(text))

其中通过使用可以获得：

    thu = thulac.thulac()
    thulac_ner('利用工具进行分词，词性标注')
    >>> Model loaded succeed
    >>> ['工具', '词性']

该部分是从项目中剥离出来的，可以见得通过thulac的辨识，可以简单对相关实体进行提取，有点像jieba中的关键词抽取。

## 三、wiki知识抽取
Agriculture_KnowledgeGraph-master/wikidataSpider/wikiextractor/extracted/zh_wiki.py
里面保存着，台湾-简体字，一些个别字词的替换


## 延伸一：neo4j的启动

在启动整个程序的时候，会出现:

    You have 14 unapplied migration(s). Your project may not work properly until you apply the migrations for app(s): admin, auth, contenttypes, sessions.
    Run 'python manage.py migrate' to apply them.

启动之后报错。

解决：

    sudo python3 manage.py migrate
