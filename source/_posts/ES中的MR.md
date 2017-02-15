---
title: ES中的MR
date: 2017-02-15 14:21:51
tags:
---
elastic search真是个让人既爱又恨的东西，性能强劲，功能强大，但就是在使用中遇到种种问题(多半因为文档太差)。
文章记录一下在es 2.2.0版本中使用Scripted Metric Aggregation（也就是牛X的map-reduce）的方法。
[api](https://www.elastic.co/guide/en/elasticsearch/reference/2.2/search-aggregations-metrics-scripted-metric-aggregation.html)这是官方文档，但是并不详细，看完并不能干出什么事来．这是[java api](https://www.elastic.co/guide/en/elasticsearch/client/java-api/2.2/_metrics_aggregations.html)

下面贴出完整的实践内容和代码(敏感内容已抹去)，目的是根据行为日志得出活跃分值
- 在elasticsearch.yml文件中添加配置启用groovy脚本
```shell
script.engine.groovy.file.aggs: true
script.engine.groovy.file.mapping: true
script.engine.groovy.file.search: true
script.engine.groovy.file.update: true
script.engine.groovy.file.plugin: true
script.engine.groovy.indexed.aggs: true
script.engine.groovy.indexed.mapping: false
script.engine.groovy.indexed.search: true
script.engine.groovy.indexed.update: false
script.engine.groovy.indexed.plugin: false
script.engine.groovy.inline.aggs: true
script.engine.groovy.inline.mapping: true
script.engine.groovy.inline.search: true
script.engine.groovy.inline.update: true
script.engine.groovy.inline.plugin: true
```
- 这是es 中保存的数据 
```shell
curl -XPUT "http://10.1.200.34:9200/behavior-2017.02/candidate/AVoG9St-6pLzqkumYcIr" -d '
{
  "businessLine": "platform",
  "createTime": "2017-02-04T02:30:14.000Z",
  "latitude": 0,
  "longitude": 0,
  "name": "c_login",
  "network": "unknown",
  "ownerId": 6403128,
  "ownerType": "candidate",
  "params": {
    "positionId": "2112620"
  },
   "uuid": "a36556ed286348aeb970e0ba1cda1447"
  }'
curl -XPUT "http://10.1.200.34:9200/behavior-2017.02/candidate/AVoG9SgJP_y-H6mvM9g8" -d '
{
   "businessLine": "platform",
   "clientIp": "*3.1*8.113.*6",
   "createTime": "2017-02-04T02:30:13.683Z",
   "latitude": 0,
   "longitude": 0,
   "name": "c_login",
   "network": "unknown",
   "ownerId": 6403118,
   "ownerType": "candidate",
   "terminal": "pc",
   "userAgent": "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/50.0.2661.102 Safari/537.36",
   "uuid": "0d9b007e3d624180aefd57e7df0b656c",
}'
curl -XPUT "http://10.1.200.34:9200/behavior-2017.02/candidate/AVoG9SgJP_y-H6mvM9g1" -d '
{
   "businessLine": "platform",
   "clientIp": "*23.*8.*3.1*",
   "createTime": "2017-02-04T02:30:13.683Z",
   "latitude": 0,
   "longitude": 0,
   "name": "c_register",
   "network": "unknown",
   "ownerId": 6403127,
   "ownerType": "candidate",
   "terminal": "pc",
   "userAgent": "Mozilla/5.0 (Windows NT 5.1) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/50.0.2661.102 Safari/537.36",
   "uuid": "0d9b007e3d624180aefd57e7df0b656c",
}'
```
- 这是查询用的脚本，以id为分组，统计行为信息。
  ```shell
  curl -XGET "http://10.1.200.34:9200/behavior*/_search/?pretty" -d '
  {
 "aggs": {
  "group_by_ownerId" : { 
   "terms" : {
  "field" : "ownerId" 
 },
   "aggs":{
   "livenessScore": {
  "scripted_metric": {
 "init_script" : {"file": "user-liveness-score-init"},
   "map_script" : {"file": "user-liveness-score-map"},
  "combine_script" : {"file": "user-liveness-score-combine"},
 "reduce_script" : {"file": "user-liveness-score-reduce"},
   "params": {
   "_agg": {"resumeScore":{6403127:60}}
  }
   }
   }
  } 
 }
  },
 "query": {
  "filtered": {
 "query": {
   "match_all": {}
  }
  }
   },
  "fields": [
   "ownerId"
 ]
 }'
 ```
- 脚本及脚本的存放位置
 ```shell
 [root@jiqi001 scripts]# pwd
 /apps/elasticsearch/config/scripts
 [root@jiqi001 scripts]# ls
 user-liveness-score-combine.groovy  user-liveness-score-init.groovy  user-liveness-score-map.groovy  user-liveness-score-reduce.groovy
 ```
 ```shell
 _agg.loginScoreInWeek=0;
 _agg.loginScoreInMonth=0;
 _agg.registerScoreInWeek=0;
 _agg.registerScoreInMonth=0;
 ~ 
 "user-liveness-score-init.groovy" 15L, 383C
 ```
 ````shell 
 xDaysBefore = Math.round((new Date().getTime() - doc.createTime) / 1000 / 60 / 60 / 24);
 behaviorName = doc.name.value;
 resumeScore = _agg.resumeScore.get(String.valueOf(doc.ownerId.value));
 if (behaviorName.equals("c_login")) {
  if (xDaysBefore <= 7) {
  if (_agg.loginScoreInWeek < 4) {
  _agg.loginScoreInWeek += 2;
 }
  } else if (7 < xDaysBefore && xDaysBefore <= 30) {
 if (_agg.loginScoreInMonth < 2) {
 _agg.loginScoreInMonth += 1;
   }
 }
 }  else if (behaviorName.equals("c_register")) {
  if (resumeScore != null && resumeScore > 30) {
 if (xDaysBefore <= 7) {
 if (_agg.registerScoreInWeek == 0) {
  _agg.registerScoreInWeek = 5;
  }
 } else if (7 < xDaysBefore && xDaysBefore <= 30) {
 if (_agg.registerScoreInMonth == 0) {
  _agg.registerScoreInMonth = 3;
  }
 }
  }
  };
  ~   
  "user-liveness-score-map.groovy" 66L, 2657C
```
```shell
  _agg
  ~ 
  "user-liveness-score-combine.groovy" 1L, 5C
```
````shell
  double score = 0;
  loginScoreInWeek=0;
  loginScoreInMonth=0;
  registerScoreInWeek=0;
  registerScoreInMonth=0;
  for (a in _aggs) {
 if(loginScoreInWeek<4){
  loginScoreInWeek += a.get("loginScoreInWeek");
 };
  if(loginScoreInMonth<2){
   loginScoreInMonth += a.get("loginScoreInMonth");
  };
   if(registerScoreInWeek<5){
 registerScoreInWeek += a.get("registerScoreInWeek");
   };
 if(registerScoreInMonth<3){
  registerScoreInMonth += a.get("registerScoreInMonth");
 };
 };
   if(loginScoreInWeek>4){
 loginScoreInWeek =4;
   };
 if(loginScoreInMonth>2){
  loginScoreInMonth =2;
 };
  if(registerScoreInWeek>5){
   registerScoreInWeek =5;
  };
   if(registerScoreInMonth>3){
 registerScoreInMonth =3;
   };
   score += loginScoreInWeek;
 score += loginScoreInMonth;
  score += registerScoreInWeek;
   score += registerScoreInMonth;
 return score;
 "user-liveness-score-reduce.groovy"
```

 - Java代码
 ```java
  private Map<String, Double> getUsersLiveScore(Map<String, Object> userAndResumeScore) throws InterruptedException, ExecutionException {
      Map<String, Double> userAndLivenessScore = new HashMap<>();
      Map<String, Object> param = new HashMap<>();
      param.put("resumeScore", userAndResumeScore);
      Map<String, Object> params = new HashMap<>();
      params.put("_agg", param);
      Client client = eSClient.getClient();
      AggregationBuilder aggregation = AggregationBuilders.terms("group_by_ownerId")
                             .field("ownerId")
                             .subAggregation(
                                 AggregationBuilders.scriptedMetric("livenessScore")
                                    .params(params)
                                    .initScript(new Script("user-liveness-score-init", ScriptService.ScriptType.FILE, "groovy", null))
                                    .mapScript(new Script("user-liveness-score-map", ScriptService.ScriptType.FILE, "groovy", null))
                                    .combineScript(new Script("user-liveness-score-combine", ScriptService.ScriptType.FILE, "groovy", null))
                                    .reduceScript(new Script("user-liveness-score-reduce", ScriptService.ScriptType.FILE, "groovy", null))
                             );
      TermsQueryBuilder ownerId = QueryBuilders.termsQuery("ownerId", userAndResumeScore.keySet());
      BoolQueryBuilder boolQueryBuilder = QueryBuilders.boolQuery();

      boolQueryBuilder.must(ownerId);
      boolQueryBuilder.must(QueryBuilders.rangeQuery("createTime").gte(DateTime.now().plusDays(-30).toDate()));
      SearchResponse response = client.prepareSearch("behavior-*")
                           .setSearchType(SearchType.DFS_QUERY_THEN_FETCH)
                           .setQuery(boolQueryBuilder)
                           .addAggregation(aggregation)
                           .setFrom(0)
                           .setSize(3000)
                           .addField("ownerId")
                           .execute()
                           .get();
      for (Aggregation agg : response.getAggregations()) {
            List<Terms.Bucket> buckets = ((LongTerms) agg).getBuckets();
            for (Terms.Bucket bucket : buckets) {
                String userId = bucket.getKeyAsString();
                for (Aggregation agg2 : bucket.getAggregations()) {
                    double score = (double) ((InternalScriptedMetric) agg2).aggregation();
                    userAndLivenessScore.put(String.valueOf(userId), score);
                }
            }
      }
      return userAndLivenessScore;
   }
```
- Tips
1.参数param必须放在_agg变量里。
2.可以用"combine_script":"_agg;","reduce_script":"_aggs;"来调试脚本。
3.combine组合的结果会以分片为分组，并非整个查询结果的组合．比如查询一个index如果在５个分片上有结果则返回一个长度为５的数组,
