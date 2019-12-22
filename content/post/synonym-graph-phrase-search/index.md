---
aliases: [phrase-search-synonym-graph-token-filter]
projects: [elasticsearch]
title: Phrase Search with Synonym Graph Token Filter in Elasticsearch
authors: [Dainius Jocas]
date: '2019-12-22'
tags: [elasticsearch]
categories:
  - elasticsearch
summary: An idea on how to search for short string with a long query string
image:
  caption: "[Photo by www.infoworld.com](https://www.infoworld.com/article/3442739/review-elasticsearch-boosts-search-with-sql-optimizations.html)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

I've [written](https://www.jocas.lt/blog/post/es-percolator-phrase-highlight/) that if you google for `How can you match a long query text to a short text field?` you're advised to use Elasticsearch Percolator. Today I'll show an alternative way of solving the same problem with Elasticsearch.

The main idea is to use [Synonym Graph Token Filter](https://www.elastic.co/guide/en/elasticsearch/reference/master/analysis-synonym-graph-tokenfilter.html) with some data preparation.

## Problem Statement

Say that we learned how extract some entity from free form text with techniques such as NER, dictionary annotations, or some fancy Machine Learning. And when this entity is mentioned in the search query we want to boost documents that mention this entity. Also, say you've ruled out using Elasticsearch Percolator because it increases network latency because it requires additional call to Elasticsearch. 

For further discussion our unstructured text is going to be `This description is about a Very Important Thing and something else.` and the extracted entity `Very Important Thing`. Our test document looks like :
```json
{
  "description": "This description is about a Very Important Thing and something else.",
  "entity": "Very Important Thing"
}
```

Search queries:
- `prefix very important thing suffix`
- `prefix very important another thing suffix`
- `prefix thing suffix`

All examples are tested on Elasticsearch 7.5.1.

### Naive Setup

Let's create an index for our documents:
```
PUT /test_index-2
{
  "mappings": {
    "properties": {
      "description": {
        "type": "text"
      },
      "entity": {
        "type":  "text"
      }
    }
  }
}
```
Entity field is of type `text` because we want it to be searchable. `keyword` type won't work because it does only exact matches and out query most likely will be longer than our entity string.  

Index our document:
```
PUT test_index-2/_doc/1
{
  "description": "This description is about a Very Important Thing and something else.",
  "entity": "Very Important Thing"
}
```

Search the index with the query that mentions our `very important thing`:
```
GET test_index-2/_search
{
  "query": {
    "match": {
      "entity": {
        "query": "prefix very important thing suffix",
        "boost": 2
      }
    }
  }
}
```

This yields:
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.7260926,
    "hits" : [
      {
        "_index" : "test_index-2",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.7260926,
        "_source" : {
          "description" : "This description is about a Very Important Thing and something else.",
          "entity" : "Very Important Thing"
        }
      }
    ]
  }
}
```

Cool, we found what we we looking for.

Let's try another query, this time with a mention of `very important another thing`:
```
GET test_index-2/_search
{
  "query": {
    "match": {
      "entity": {
        "query": "prefix very important another thing suffix",
        "boost": 2
      }
    }
  }
}
```

This yields:
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 1.7260926,
    "hits" : [
      {
        "_index" : "test_index-2",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.7260926,
        "_source" : {
          "description" : "This description is about a Very Important Thing and something else.",
          "entity" : "Very Important Thing"
        }
      }
    ]
  }
}
```

Oh, the results are the same as with the previous query despite the fact that we mention `Another Thing` here. But it still might be OK because we matched all the terms of the entity.

Let's try another query:
```
GET test_index-2/_search
{
  "query": {
    "match": {
      "entity": {
        "query": "prefix thing suffix",
        "boost": 2
      }
    }
  }
}
```

This yields:
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.5753642,
    "hits" : [
      {
        "_index" : "test_index-2",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.5753642,
        "_source" : {
          "description" : "This description is about a Very Important Thing and something else.",
          "entity" : "Very Important Thing"
        }
      }
    ]
  }
}
```
Oh no, we still matched our `Very Important Thing` while only `thing` term is  present in the query. But at least this time the score is lower than with previous twoqueries, 0.5753642 vs. 1.7260926. Here we clearly see the problem: we are matching short strings with long strings and partial matches raises problems.

## Proposed Solution

Let's leverage Synonym Graph Token Filter to solve our problem.

```
PUT /test_index-1
{
  "mappings": {
    "properties": {
      "descrition": {
        "type":  "text"
      },
      "entity": {
        "type":  "text",
        "analyzer": "lowercase_keyword_analyzer",
        "search_analyzer": "synonym_graph_analyzer"
      }
    }
  },
  "settings": {
    "index": {
      "analysis": {
        "analyzer": {
          "synonym_graph_analyzer": {
            "tokenizer": "standard",
            "filter": [
              "lowercase",
              "my_synonym_graph"
            ]
          },
          "lowercase_keyword_analyzer": {
            "tokenizer": "keyword",
            "filter": [
              "lowercase"
            ],
            "char_filter": [
              "spaces_to_undescores_filter"
            ]
          }
        },
        "char_filter": {
          "spaces_to_undescores_filter": {
            "type": "mapping",
            "mappings": [
              " \\u0020 => _"
            ]
          }
        },
        "filter": {
          "my_synonym_graph": {
            "type": "synonym_graph",
            "lenient": true,
            "synonyms": [
              "very important thing => very_important_thing"
            ]
          }
        }
      }
    }
  }
}
```

Let's decompose this large index configuration piece by piece:
1. The `entity` attribute now has separate analyzers for both index and search phases.
2. The `lowercase_keyword_analyzer` uses keyword tokenizer which means that tokenization will result in the sequence of token of size 1, then it normalizes tokens by lowercasing them and finally `spaces_to_undescores_filter`, replaces spaces to underscores. E.g. a string `"Very Important Thing"` is transformed into list of tokens `["very_important_thing"]`. Or use out friend `_analyze` API:
```
POST test_index-1/_analyze
{
  "text": ["Very Important Thing"],
  "analyzer": "lowercase_keyword_analyzer"
}
```
This yields:

```
{
  "tokens" : [
    {
      "token" : "very_important_thing",
      "start_offset" : 0,
      "end_offset" : 20,
      "type" : "word",
      "position" : 0
    }
  ]
}
```

3. The `synonym_graph_analyzer` use standard tokenizer, which is followed by the `lowercase` filter, and then the `my_synonym_graph` token filter is applied. We've set up one synonym `"very important thing => very_important_thing"`. E.g.
```
POST test_index-1/_analyze
{
  "text": ["prefix very important thing suffix"],
  "analyzer": "synonym_graph_analyzer"
}
```
This yields:
```
{
  "tokens" : [
    {
      "token" : "prefix",
      "start_offset" : 0,
      "end_offset" : 6,
      "type" : "<ALPHANUM>",
      "position" : 0
    },
    {
      "token" : "very_important_thing",
      "start_offset" : 7,
      "end_offset" : 27,
      "type" : "SYNONYM",
      "position" : 1
    },
    {
      "token" : "suffix",
      "start_offset" : 28,
      "end_offset" : 34,
      "type" : "<ALPHANUM>",
      "position" : 2
    }
  ]
}
```
After analysis we have 3 tokens `["prefix", "very_important_thing", "suffix"]`. Notice `"very_important_thing"` token: this is equal to the right-hand-side from our synonym definitions.  Now let's run queries from the previous section:
```
GET test_index-1/_search
{
  "query": {
    "match": {
      "entity": {
        "query": "prefix very important thing suffix",
        "boost": 2
      }
    }
  }
}
```
This yields:
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.5753642,
    "hits" : [
      {
        "_index" : "test_index-1",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.5753642,
        "_source" : {
          "description" : "This description is about a Very Important Thing and something else.",
          "entity" : "Very Important Thing"
        }
      }
    ]
  }
}
```

As expected: exact match -> hit.

Another query:
```
GET test_index-1/_search
{
  "query": {
    "match": {
      "entity": {
        "query": "prefix very important another thing suffix",
        "boost": 2
      }
    }
  }
}
```

This yields:
```
{
  "took" : 0,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```
No hits! Good! The document is not going to be boosted despite the fact that all tokens match.

And the last one:
```
GET test_index-1/_search
{
  "query": {
    "match": {
      "entity": {
        "query": "prefix thing suffix",
        "boost": 2
      }
    }
  }
}
```

This yields:
```
{
  "took" : 1,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 0,
      "relation" : "eq"
    },
    "max_score" : null,
    "hits" : [ ]
  }
}
```

No hits! Good. This means that also substring doesn't match.

## Discussion

Synonym Graph Token Filter can "replace" a sequence of tokens (e.g. a phrase) with another sequence of tokens. In this particular example: many tokens were replaced with one token.

1. One field can have only one analyzer pair for index and search phases. If we want another analysis pipeline for the `entity` attribute we have to create another field with the analyzers specified, e.g. stemmed phrase with lower boost.
2. The synonym list must be prepared before the index creation.
3. Management of the synonym list might complicate index management, e.g. you use templates for your index management. 
4. The overal solution in general might look a bit too complicated.
