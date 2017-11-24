---
layout: post
title: elasticsearch 入门及中文分词ik
---

Elasticsearch是款全文搜索引擎,也是目前全文搜索引擎的首选。

很多年前有用过，但是之前没理解，也没记录，再次使用又得重头开头。刚好这次机会有需要用到elasticsearch和中文分词。就做个入门blog。

Elasticsearch的底层是开源库Lucence, Elasticsearch提供RestAPI, 方便使用。

这篇文章先从最简单的单机elasticsearch开始搭建:

一、Install

  前提条件:
      1. java8: 在这不细说,例如ubuntu环境下可以直接:apt-get install default-jre
      
  安装Elasticsearch:[官方文档](https://www.elastic.co/guide/en/elasticsearch/reference/current/zip-targz.html)
      
      $ wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.4.zip
      $ unzip elasticsearch-5.6.4.zip
      $ cd elasticsearch-5.6.4/ 

  启动elasticsearch:
      
      $ ./bin/elasticsearch start
      
  如果启动正常,elasticsearch默认启动端口9200,此时可访问:
  
      $ curl http://127.0.0.1:9200
        {
          "name" : "master",
          "cluster_name" : "logger",
          "cluster_uuid" : "lUTEPbfdRBupX7DcWT8gBw",
          "version" : {
            "number" : "5.6.4",
            "build_hash" : "8bbedf5",
            "build_date" : "2017-10-31T18:55:38.105Z",
            "build_snapshot" : false,
            "lucene_version" : "6.6.1"
          },
          "tagline" : "You Know, for Search"
        }
 
二、安装ik中文分词:

  elasticsearch自带的中文索引不太好用，分词不准确. 在网上找了个ik,测试了下分词很准确,[ik 分词器的github地址](https://github.com/medcl/elasticsearch-analysis-ik)
  
  ik文档里已经有了安装方法，方法一:
  
    ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.0.0/elasticsearch-analysis-ik-6.0.0.zip
    
  方法二: 
  
    1. 从  https://github.com/medcl/elasticsearch-analysis-ik/releases 这下载安装包
    2. 解压到 es-path/plugins
    
  我用的版本是5.6.4, 采用方法一时,在plugins/analysis-ik里没有config目录，启动后无法用ik分词器.后采用方法二成功可用。
    
三、基本语法:
  
  以上就已经安装好,下面介绍一些常用的语法:
  
  查看当前节点的所有索引:
  
     $ curl -X GET 'http://localhost:9200/_cat/indices?v'

  创建索引:
  
     $ curl -XPUT http://localhost:9200/indexname
     
  删除索引:
  
     $ curl -X DELETE localhost:9200/indexname
     
  创建map:
   
     $ curl -XPOST http://localhost:9200/indexname/fulltext/_mapping -d'
      {
              "properties": {
                  "content": {
                      "type": "text",
                      "analyzer": "ik_max_word",
                      "search_analyzer": "ik_max_word"
                  }
              }
          
      }'

   这个map指明了: 在indexname下，有个type: fulltext。 对字段 content 采用ik_max_word 的分析器
   
   创建数据:
   
      $ curl -XPOST http://localhost:9200/indexname/fulltext/1 -d'
        {"content":"美国留给伊拉克的是个烂摊子吗"}
        '
        
      $ curl -XPOST http://localhost:9200/indexname/fulltext/2 -d'
      {"content":"公安部：各地校车将享最高路权"}
      '
      
      $ curl -XPOST http://localhost:9200/indexname/fulltext/3 -d'
      {"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}
      '
      
      $ curl -XPOST http://localhost:9200/indexname/fulltext/4 -d'
      {"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}
      '

   查询数据:
   
      $ curl -XPOST http://localhost:9200/index/fulltext/_search  -d'
      {
          "query" : { "match" : { "content" : "中国" }},
          "highlight" : {
              "pre_tags" : ["<tag1>", "<tag2>"],
              "post_tags" : ["</tag1>", "</tag2>"],
              "fields" : {
                  "content" : {}
              }
          }
      }'

   返回结果:
   
      {"took":5,"timed_out":false,"_shards":{"total":5,"successful":5,"skipped":0,"failed":0},"hits":{"total":2,"max_score":0.6099695,"hits":[{"_index":"indexname1","_type":"fulltext","_id":"4","_score":0.6099695,"_source":
      {"content":"中国驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"}
      ,"highlight":{"content":["<tag1>中国</tag1>驻洛杉矶领事馆遭亚裔男子枪击 嫌犯已自首"]}},{"_index":"indexname1","_type":"fulltext","_id":"3","_score":0.27179778,"_source":
      {"content":"中韩渔警冲突调查：韩警平均每天扣1艘中国渔船"}
      ,"highlight":{"content":["中韩渔警冲突调查：韩警平均每天扣1艘<tag1>中国</tag1>渔船"]}}]}}


   查看以上结果，是能正确匹配到 "中国" 字段的, 如果没有采用ik中文分词器,elasticsearch默认的结果会返回匹配 "中" 或者 "国" 的结果
   
   查看某条具体的数据记录:
   
       $ curl 'localhost:9200/indexname/fulltext/1?pretty=true'
       
   删除记录:
   
       $ curl -X DELETE 'localhost:9200/indexname/fulltext/1'
       
   列出所有记录:
   
       $ curl 'localhost:9200/indexname/fulltext/_search'
       
   带条件的查询:
   
       $ curl 'localhost:9200/indexname/fulltext/_search'  -d '
      {
        "query" : { "match" : { "desc" : "中国" }},
        "from": 0,
        "size": 1
      }'
