# 市长信箱邮件查询服务: 使用Elasticsearch 替代 Mysql

我在上一篇文章中实现一个基于Springboot构建的web应用: 市长信箱邮件查询服务. 应用将邮件信息抓取后,将数据保存在Mysql中,用以提供给搜索Web使用.使用mysql能快速实现功能, 但使用like查询性能上不太好, 数据量大了就必须考虑使用搜索引擎了. 这次我们把Mysql替换为Elasticsearch(ES).

Elasticsearch提供了两种API来进行操作: Rest API与Java Native. Java Native的执行效率更高, 且与项目集成更方便. 我们这里使用Java native Api操作Elasticsearch. 引入Java native Api,需要在根pom.xml添加依赖:
``` xml
<dependency>
    <groupId>org.elasticsearch</groupId>
    <artifactId>elasticsearch</artifactId>
    <version>2.3.3</version>
</dependency>
``` 

使用ES替换Mysql,需要考虑这两个方面:
1. 存储: 将Mysql的insert保存数据,改为使用调用ES的添加文档操作.
2. 查询: 替换Mysql的查询sql语句,改为调用ES的搜索接口.

## 存储:
  我们的邮件结构没有内嵌其他复杂对象, 所以从mysql转换为ES的文档非常简单. 只需简单调用ES的添加文档API即可:
``` java
public static void indexMails(Iterable<Mail> mails) {
    BulkRequestBuilder bulkRequest = client.prepareBulk();
    for (Mail mail : mails) {
        addMailIndexRequest(bulkRequest, mail);
    }
    BulkResponse bulkResponse = bulkRequest.get();
    System.out.println(JSON.toJSONString(bulkResponse));
}

private static void addMailIndexRequest(BulkRequestBuilder bulkRequest, Mail mail) {
    try {
        bulkRequest.add(client.prepareIndex("chengdu12345", "mail")
            .setSource(XContentFactory.jsonBuilder()
                .startObject()
                .field("content", mail.content)
                .field("createDate", mail.createDate)
                .field("acceptUnit", mail.acceptUnit)
                .field("category", mail.category)
                .field("result", mail.result)
                .field("sender", mail.sender)
                .field("status", mail.status)
                .field("title", mail.title)
                .field("url", mail.url)
                .field("views", mail.views)
                .endObject())
        );
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
}
```
以上代码中,我使用了批量添加(bulkRequest)的API, 以减少API调用次数.

## 查询: 
  邮件的查询sql如下:
``` sql
select m from cheng12345 m where m.title like ? or m.content like ? or m.result like ? order by create_date desc limit ?,?
```
  这里对多个字段...


``` json
{
  "query": {
    "multi_match": {
      "query": "红牌楼",
      "fields": [
        "content",
        "result",
        "title"
      ],
      "type": "phrase"
    }
  },
  "sort": [
    {
      "createDate": {
        "order": "desc"
      }
    }
  ],
  "size": 100,
  "from": 0
}
``` 