// 4 Elasticsearch analyzer

// 4.3.1 索引时使用analyzer

// ① ES默认使用的standard
POST _analyze
{
  "analyzer":"standard",
  "text":"Hello, I am Tommy and living in China."
}

// ② 使用simple
POST _analyze
{
  "analyzer":"simple",
  "text":"Hello, I am Tommy and living in China."
}

// ③ 使用whitespace
POST _analyze
{
  "analyzer":"whitespace",
  "text":"Hello, I am Tommy and living in China."
}


// ④ 方式1：在settings参数中指定在索引的所有文本字段中使用standard进行索引构建

PUT /case1
{
  "settings":{
    "analysis":{
      "analyzer":{
        "default":{
          "type":"standard"
        }
      }
    }
  },
  "mappings":{
    "properties":{
      "name":{  // 属性1
        "type":"text"
      },
      "address":{ // 属性2
        "type":"text"
      }
    }
  }
}


// ⑤ 方式2：在mappings参数中指定在索引的name字段中使用whitespace, address字段使用simple进行索引构建

PUT /case2
{
  "mappings":{
    "properties":{
      "name":{
        "type":"text",
        "analyzer":"whitespace"  // 分词器1
      },
      "address":{
        "type":"text",
        "analyzer":"simple"  // 分词器2
      }
    }
  }
}



// 4.3.4 搜索时使用analyzer
PUT /case3
{
  "mappings":{
    "properties":{
      "name":{
        "type":"text",
        "analyzer":"whitespace",  // 索引时,
        "search_analyzer":"simple"   // 搜索时
      }
    }
  }
}


// 4.3.5 自定义analyzer

PUT /case4
{
  "settings":{
    "analysis":{
      "analyzer":{
        "rebuild_analyzer":{ // 自定义的analyzer名称
          "type":"custom",
          "tokenizer":"standard",
          "filter":["lowercase"]
        }
      }
    }
  }
}

// 对指定内容根据如上自定义的分词规则进行分词
POST /case4/_analyze
{
  "text":"Hello, I am Tommy and living in China."
}



// 4.4 中文analyzer

// 4.4.3 IK分词器提供了两个子分词器:
// ik_smart 和 ik_max_word

// ① ik_smart
POST _analyze
{
  "analyzer":"ik_smart",  // 粗粒度
  "text":"今天天气很好，我们去郊游。"
}

// ② ik_max_word
POST _analyze
{
  "analyzer":"ik_max_word",  // 细粒度
  "text":"今天天气很好，我们去郊游"
}


// 4.4.4 IK分词器自定义词典

// ① 无法识别的词语
POST _analyze
{
  "analyzer":"ik_smart",
  "text":"埃隆马斯克表示搭乘SpaceX星际飞船前往火星的价格约为十万美元"
}

POST _analyze
{
  "analyzer":"ik_max_word",
  "text":"埃隆马斯克表示搭乘SpaceX星际飞船前往火星的价格约为十万美元"
}


// 自定义词典
// 第一步：在IK分析器的安装目录下的config子目录中创建文件my.dict，在其中添加词语即可。如果有更多的词语需要添加，则每个词语单独一行
// 第二步：添加完成后修改IK分析器的配置文件，路径为config/IKAnalyzer.cfg.xml，将新建的字典文件加入ext_dict选项中
// 第三步：重启ES


POST _analyze
{
  "analyzer":"ik_smart",
  "text":"埃隆马斯克表示搭乘SpaceX星际飞船前往火星的价格约为十万美元"
}



// 4.5 使用同义词

// 4.5.1 方式一：建立索引时使用同义词

// 示例
// ① 创建索引 city
PUT /city 
{
  "settings":{
    "analysis": {
      "filter": {
        "ik_synonyms_filter":{ // 自定义filter
          "type":"synonym",
          "synonyms":[
            "北京,首都,京城,北平",
            "深圳,示范区",
            "台湾省,宝岛"
          ]
        }
      },
      "analyzer":{
        "ik_analyzer_synonyms":{ // 自定义analyzer
          "tokenizer":"ik_smart",
          "filter":[
            "lowercase",          // 内置filter
            "ik_synonyms_filter"  // 自定义filter
          ]
        }
      }
    }
  },
  "mappings":{
    "properties":{
      "title":{
        "type":"text",
        "analyzer":"ik_analyzer_synonyms"  // 自定义analyzer
      }
    }
  }
}

// ② 插入数据
POST /city/_doc/001
{
  "title":"北京"
}

POST /city/_doc/002
{
  "title":"深圳"
}

POST /city/_doc/003
{
  "title":"宝岛"
}

// ③  检索数据
GET /city/_search
{
  "query":{
    "match":{
      "title":"示范区"
    }
  }
}

GET /city/_search
{
  "query":{
    "match":{
      "title":"台湾省"
    }
  }
}


// 4.5.2 方式二：建立索引时使用同义词
// 通过修改文件配置同义词，按照 Elastic 的官方文档，我们可以把所有的同义词放到一个文档中。首先，我们在 Elasticsearch 的 config 目录中，创建一个叫做 analysis 的子目录，然后创建一个叫做 synonyms.txt 的文档


// 示例
// ① 创建索引
PUT /syn_case
{
  "settings":{
    "analysis":{
      "filter":{
        "my_synonyms_filter":{  // 自定义filter
          "type":"synonym",
          "synonyms_path":"analysis/synonyms.txt"
        }
      },
      "analyzer":{
        "ik_analyzer_synonyms":{ // 自定义analyzer
          "tokenizer":"ik_smart",
          "filter":[
            "lowercase",  // 内置filter
            "my_synonyms_filter" // 自定义filter
          ]
        }
      }
    }
  },
  "mappings":{
    "properties":{
      "content":{
        "type":"text",
        "analyzer":"ik_analyzer_synonyms"  // 自定义
      }
    }
  }
}

// ② 写入数据
PUT /syn_case/_doc/001
{
  "content":"Hello我来自中国"
}

PUT /syn_case/_doc/002
{
  "content":"地球被猫星人占领了"
}


// ③ 检索数据
// 测试1
GET /syn_case/_search
{
  "query":{
    "match":{
      "content":"你好"
    }
  }
}

// 测试2
GET /syn_case/_search
{
  "query":{
    "match":{
      "content":"喵喵"
    }
  }
}

// 测试3：查看索引时的分词结果
GET /syn_case/_analyze
{
  "analyzer":"ik_analyzer_synonyms",
  "text":"Hello我来自中国"
}



// 4.5.3 查询时使用同义词
// 在ES内置的分词过滤器中还有个分词过滤器叫作synonym_graph，它是一种支持查询时用户自定义同义词的分词过滤器。

// 示例
// ① 创建索引
PUT /country
{
  "settings": {
    "analysis":{
      "filter":{
        "ik_synonyms_filter":{ // 自定义filter
          "type":"synonym_graph",
          "updateable":true, // 动态更新
          "synonyms_path":"analysis/synonyms.txt"
        }
      },
      "analyzer":{
        "ik_synonyms_search_analyzer":{
          "tokenizer":"ik_max_word",
          "filter":[
            "lowercase",  // 内置filter
            "ik_synonyms_filter"  // 自定义filter
          ]
        }
      }
    }
  },
  "mappings":{
    "properties":{
      "name":{
        "type":"text",
        "analyzer":"ik_max_word",  // 索引时
        "search_analyzer":"ik_synonyms_search_analyzer" // 搜索时
      }
    }
  }
}

// ② 查看分词结果
POST /country/_analyze
{
  "analyzer": "ik_synonyms_search_analyzer",
  "text":"中国是历史悠久的国家"
}

// ③ 输入数据
POST /country/_doc/1
{
  "name":"中国是历史悠久的国家"
}

POST /country/_doc/2
{
  "name":"美国是霸权主义的典范"
}

// 动态更新同义词词典synonyms.txt
// 添加同义词：
// 中国,CHINA,中华人民共和国
// 美国,USA,美利坚

POST /country/_reload_search_analyzers


// ④ 查询数据
GET /country/_search
{
  "query":{
    "match":{
      "name":"CHINA"
    }
  }
}




// 4.6 使用停用词

// 4.6.1 使用停用词过滤器
// 可以通过创建自定义分析器的方式使用停用词，方法是在分析器中指定停用词过滤器，在过滤器中可以指定若干个停用词。

// 示例
// ① 创建索引
PUT /stop_case_1
{
  "settings": {
    "analysis":{
      "filter":{
        "my_stop":{ // 自定义filter
          "type":"stop",
          "stopwords":[
            "我",
            "的",
            "这"
          ]
        }
      },
      "analyzer":{
        "ik_analyzer_stop":{ // 自定义analyzer
          "tokenizer":"ik_smart",
          "filter":[
            "my_stop"   // 自定义filter
          ]
        }
      }
    }
  },
  "mappings":{
    "properties": {
      "name":{
        "type":"text",
        "analyzer":"ik_analyzer_stop"  // 自定义
      }
    }
  }
}

// ② 测试：用自定义analyzer进行分词
GET /stop_case_1/_analyze
{
  "analyzer":"ik_analyzer_stop",
  "text":"这里的东西不在我这"
}



// 4.6.2 在内置analyzer中使用停用词
// 像standard这种常用的分析器都自带有停用词过滤器，只需要对其参数进行相应设置即可。

// ① 创建索引
PUT /stop_case_2
{
  "settings":{
    "analysis":{
      "analyzer":{
        "my_standard":{
          "type":"standard",  // 内置analyzer
          "stopwords":["我","的","这"]  // 停用词
        }
      }
    }
  },
  "mappings":{
    "properties": {
      "name":{
        "type":"text",
        "analyzer":"my_standard"
      }
    }
  }
}


// ② 测试
GET /stop_case_2/_analyze
{
  "analyzer":"my_standard",
  "text":"这里的东西不在我这"
}



// 4.6.3 在IK analyzer中使用停用词

// ① 示例：在默认情况下，IK分析器的分词器只有英文停用词，没有中文停用词。
POST /_analyze
{
  "analyzer":"ik_smart",
  "text":"哦对的，我在深圳工作、生活。"
}


// ② 如果用户想要添加中文停用词，需要通过自定义停用词文件的形式进行添加。在plugins/IK/config目录下创建my_stopword.dict文件，并在其中添加中文停用词即可

POST /_analyze
{
  "analyzer":"ik_smart",
  "text":"哦对的，我在深圳工作、生活呀。"
}




// 4.7 拼音搜索

// 示例1：分词效果
POST /_analyze
{
  "analyzer": "pinyin",
  "text":"王府井"
}


// 示例2：
// ① 创建索引
PUT /countries
{
  "mappings": {
    "properties": {
      "name":{
        "type":"text",
        "analyzer": "pinyin"
      }
    }
  }
}

// ② 写入数据
POST /_bulk
{"index":{"_index":"countries", "_id":"1"}}
{"name":"中国"}
{"index":{"_index":"countries", "_id":"2"}}
{"name":"美国"}
{"index":{"_index":"countries", "_id":"3"}}
{"name":"俄罗斯"}

// ③ 检索数据
// 测试1：
GET /countries/_search
{
  "query":{
    "match":{
      "name":"zg"
    }
  }
}

// 测试2:
GET /countries/_search
{
  "query":{
    "match":{
      "name":"zhongguo"
    }
  }
}

// 测试3：
GET /countries/_search
{
  "query":{
    "match":{
      "name":"mg"
    }
  }
}


// 测试4：
GET /countries/_search
{
  "query":{
    "match":{
      "name":"els"
    }
  }
}