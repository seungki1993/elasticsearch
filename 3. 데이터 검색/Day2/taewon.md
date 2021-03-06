# 7. 검색 확장 기능
## 7.1 Suggest API
### 7.1.1 Term Suggest API
- 비슷한 단어가 있을 경우 제안을 하는 API
- 키워드가 오타가 있을 경우 단어 제안을 하는 기능에 사용될 수 있음
- 편집거리 계산 알고리즘을 이용하여 비슷한 단어들을 제안

### 7.1.2 Completion Suggest API
- 자동완성 기능에 사용될 수 있는 API
- completion 이라는 별도의 데이터 타입을 지정해야 함
- prefix가 일치하는 단어들만 결과로 출력됨

# ETC
## [similarity algorithm](https://www.elastic.co/blog/practical-bm25-part-2-the-bm25-algorithm-and-its-variables)
- default는 bm25
- 5.0 버전에서 TF/IDF 에서 bm25로 변경

![Alt text](https://images.contentstack.io/v3/assets/bltefdd0b53724fa2ce/blt78ee47f523d10430/5c57eb6165ace9e30b316318/bm25_equation.png)
```
    1. qi is the ith query term.
    2. IDF(gi) is the inverse document frequency of the ith query term.
        - IDF(gi)는 query term의 횟수가 얼마나 자주 발생되는가에 대한 역수 
    3. We see that the length of the field is divided by the average field length in the denominator as fieldLen/avgFieldLen.
        - 필드의 길이가 평균 필드의 길이로 나눠진 값을 이용함(fieldLen/avgFieldLen)
```
- https://www.slideshare.net/kjmorc/ss-80803233 (32페이지부터)
- 찾는 검색어가 문서에 많을수록 score가 높으며 전체 문서에 많이 출연한 단어일수록 score 가 낮음

## [How Shards Affect Relevance Scoring in Elasticsearch](https://www.elastic.co/blog/practical-bm25-part-1-how-shards-affect-relevance-scoring-in-elasticsearch)
- 스코어 계산은 default 로 샤드 기준으로 계산을 함
```
By default, Elasticsearch calculates scores on a per-shard basis.
```
- 샤드에 document가 어떤 분포로 들어가느냐에 따라 score가 다르게 나올 수 있음

## [Function score query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-function-score-query.html)
- 검색 결과와 임의로 조정한 score 값을 합쳐서 결과를 출력
- result example
```json
{
    "query": {
        "function_score": {
          "query": { "match_all": {} },
          "boost": "5", 
          "functions": [
              {
                  "filter": { "match": { "test": "bar" } },
                  "random_score": {}, 
                  "weight": 23
              },
              {
                  "filter": { "match": { "test": "cat" } },
                  "weight": 42
              }
          ],
          "max_boost": 42,
          "score_mode": "max",
          "boost_mode": "multiply",
          "min_score" : 42
        }
    }
}
```

- 옵션은 score_mode, boost_mode, weight
    - score_mode
        - sub query 에서 나온 결과를 어떻게 조합하는가
        - multiply(default), sum, avg, first, max, min
    - boost_mode
        - sub query로 나온 값과 main query로 나온 값을 어떻게 조합할 것인가를 결정
        - multiply(default), sum, avg, first, max, min
    - weight
        - weight 값이 존재할 경우 해당 weight 값으로 score 값을 곱함                
- 함수는 field_value_factor, script_score, Decay functions 
    - field_value_factor
        - 해당 필드에 값이 존재할 경우 그 값을 score 에 포함시킴
        - 해당 함수의 boost 값으로 추가되는 값을 조정할 수 있음
        - missing 은 해당 필드가 존재하지 않을 경우 들어가야 하는 값을 정의함
        - 결과는 음수가 되면 안됨
        - log를 사용할 경우 인자 값이 0이 되면 연산이 되지 않기 때문에 주의해야 함
        - log1p, log2p를 사용하여 해당 문제를 회피할 수 있음
    - script_score
        - painless로 작성 된 스크립트를 이용해서 score 에 포함 될 값을 선정
    - Decay functions
        - 필드의 값이 기준 값에 얼마나 떨어져있는지를 이용해서 얼마나 score를 조작할 것인가를 선정
        - 종류는 gauss, lin, exp
        ![Alt text](https://www.elastic.co/guide/en/elasticsearch/reference/current/images/decay_2d.png)
        - date 필드에 적용하여 시간순으로 정렬할 수 있음
                    
## [search_as_you_type](https://www.elastic.co/blog/elasticsearch-7-2-0-released)
```
For more flexible search-as-you-type searches that do not use suggesters, 
see the search_as_you_type field type.
```
- 7.2 버전에서 새로 추가됨
- Shingle Token Filter 를 이용하여 기존 필드 이외에 3개의 필드를 더 추가함
    - my_field._2gram
    - my_field._3gram
    - my_field._index_prefix
    #### [Shingle Token Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-shingle-tokenfilter.html)
        the sentence "please divide this sentence into shingles" might be tokenized into shingles 
        "please divide", "divide this", "this sentence", "sentence into", and "into shingles".  
- 추가된 필드를 이용하여 prefix로 검색 할 수 있고, 중간에서 있는 값도 검색이 가능함
- 공식 문서의 예제에서는 'multi_match' 와 'match_bool_prefix'를 조합하여 사용
    ### [match_bool_prefix](https://www.elastic.co/guide/en/elasticsearch/reference/7.2/query-dsl-match-bool-prefix-query.html)
    - if message is "**quick brown f**", then query is        
        ```json
        {
            "query": {
                "bool" : {
                    "should": [
                        { "term": { "message": "quick" }},
                        { "term": { "message": "brown" }},
                        { "prefix": { "message": "f"}}
                    ]
                }
            }
        }
        ```
        
## [rank_feature](https://www.elastic.co/blog/easier-relevance-tuning-elasticsearch-7-0)
- 7.0 버전에서 새로 생긴 field 와 query
- 필드는 rank_feature 와 rank_features
- 필드에 저장되어 있는 값을 기존 score에 더하는 방식으로 score 조정
```
    score = bm25_score + satu(pagerank)
```
- 7.0에서 추가된 Faster top k retrieval 을 support 하기 위한 기능으로 추정됨
```
this query has the benefit of being able to efficiently skip non-competitive hits 
when track_total_hits is not set to true
```
- 제공되는 함수들은 saturation, logarithm, sigmoid
```
    saturation = S / (S + pivot)
    logarithm = log(scaling_factor + S)
    sigmoid = S^exp^ / (S^exp^ + pivot^exp^)
``` 
- Function score query 에서 값을 조정하는 것과 차이점이 뭔지 모르기 때문에 더 찾아봐야 함 