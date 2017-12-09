# Vector Scoring Plugin for Elasticsearch

This plugin allows you to score documents based on arbitrary raw vectors, 
using dot product or cosine similarity.

### Releases

Master branch targets Elasticsearch 5.6.5. 

[Branch es-2.4](https://github.com/MLnick/elasticsearch-vector-scoring/tree/es-2.4) targets Elasticsearch 2.4.x

## Overview

The aim of this plugin is to enable real-time scoring of vector-based 
models, in particular factor-based recommendation models.

In this case, user and item factor vectors are indexed using 
the [Delimited Payload Token Filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-delimited-payload-tokenfilter.html), 
e.g. the vector `[1.2, 0.1, 0.4, -0.2, 0.3]` is indexed as a string: 
`0|1.2 1|0.1 2|0.4 3|-0.2 4|0.3`.

This stores the vector indices as "terms" and the vector values as 
"payloads".

## Scoring

This plugin provides a native script `payload_vector_score` for use 
in `function_score` queries.

The script computes the dot product between the query vector and the 
document vector. In pseudo-code:

```java
for (i : vector_indices_terms) {
    payload = indexTermField(i).getPayload()
    score += payload * queryVector(i)
}
```

## Plugin installation

Targets Elasticsearch `5.6.5` and Java `1.8`.

### Build from source

1. Build: `mvn package`
2. Install plugin in Elasticsearch: `ELASTIC_HOME/bin/elasticsearch-plugin install file:///PROJECT_HOME/target/releases/elasticsearch-vector-scoring-5.6.5.zip` (stop ES first).


Start Elasticsearch: `ELASTIC_HOME/bin/elasticsearch`. You should see the plugin registered at Elasticsearch startup:
```
...
[2017-03-29T13:46:57,804][INFO ][o.e.p.PluginsService     ] [2Zs8kW3] loaded plugin [elasticsearch-vector-scoring]
...
```

## Example usage

### Index setup

```json
PUT test_vs
{
    "settings" : {
        "analysis": {
                "analyzer": {
                   "payload_analyzer": {
                      "type": "custom",
                      "tokenizer":"whitespace",
                      "filter":"delimited_payload_filter"
                    }
          }
        }
     }
}



PUT test_vs/_mapping/movies
{
    "movies" : {
        "properties" : {
            "model_factor": {
                            "type": "text",
                            "term_vector": "with_positions_offsets_payloads",
                            "analyzer" : "payload_analyzer"
                     }
        }
    }
}


PUT test_vs/movies/1
{
    "model_factor":"0|1.2 1|0.1 2|0.4 3|-0.2 4|0.3",
    "name": "Test 1"
}

PUT test_vs/movies/2
{
    "model_factor":"0|0.1 1|2.3 2|-1.6 3|0.7 4|-1.3",
    "name": "Test 2"
}


PUT test_vs/movies/3
{
    "model_factor":"0|-0.5 1|1.6 2|1.1 3|0.9 4|0.7",
    "name": "Test 3"
}


GET test_vs/movies/1/_termvector
{
  "fields" : ["model_factor"],
  "payloads" : true,
  "positions" : true
}



POST test_vs/movies/_search?pretty
{
    "query": {
        "function_score": {
            "query" : {
                "query_string": {
                    "query": "*"
                }
            },
            "script_score": {
                "script": {
                  "source": "payload_vector_score",
                  "lang": "native",
                  "params": {
                      "field": "model_factor",
                      "vector": [0.1,2.3,-1.6,0.7,-1.3],
                      "cosine" : true
                    }
        }
            },
            "boost_mode": "replace"
        }
    }
}
```

