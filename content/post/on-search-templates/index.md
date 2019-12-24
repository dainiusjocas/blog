---
aliases: [search-templates]
projects: [elasticsearch]
title: Using Search Templates in Elasticsearch
authors: [Dainius Jocas]
date: '2019-12-23'
tags: [elasticsearch]
categories:
  - elasticsearch
summary: A couple of examples and notes on using Elasticsearch search templates
image:
  caption: "Photo from ze internets"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

I want to take a look at [Search Templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-template.html) for Elasticsearch. Let's apply them to examples from [previous post on Synonym Graphs](https://www.jocas.lt/blog/post/synonym-graph-phrase-search/). 

## Setup

I'm using Elasticsearch 7.5.1.

Index configuration:
```
DELETE test_index-1
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

Index the document:
```
PUT test_index-1/_doc/1
{
  "description": "This description is about a Very Important Thing and something else.",
  "entity": "Very Important Thing"
}
```

Search queries:
- `prefix very important thing suffix`

## Templates

I'm very interested in one particular use of the search templates: how flexible is the management of stored seach templates? Can I update a search template while receiving queries?

Add a template:
```
POST _scripts/synonym-graph-search
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": {
          "entity": {
            "query": "{{query_string}}",
            "boost": 2
          }
        }
      }
    }
  }
}
```

Try to run the search:
```
GET test_index-1/_search/template
{
  "id": "synonym-graph-search",
  "params": {
    "query_string": "suffix very important thing prefix"
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

Exactly as expected.

When using a stored search template the Elasticsearch client doesn't need to handle the complex query construction.

## Templates are updateable

Let's try to update the template with a higher boost value:
```
POST _scripts/synonym-graph-search
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": {
          "entity": {
            "query": "{{query_string}}",
            "boost": 5
          }
        }
      }
    }
  }
}
```

Works.

Now let's run the same query:
```
GET test_index-1/_search/template
{
  "id": "synonym-graph-search",
  "params": {
    "query_string": "suffix very important thing prefix"
  }
}
```

This yields:
```
{
  "took" : 4,
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
    "max_score" : 1.4384103,
    "hits" : [
      {
        "_index" : "test_index-1",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.4384103,
        "_source" : {
          "description" : "This description is about a Very Important Thing and something else.",
          "entity" : "Very Important Thing"
        }
      }
    ]
  }
}
```

The scores are  0.5753642 and 1.4384103 that is ~2/5. Cool! This means that without changing (and redeploying) the Elasticsearch client we can change the querying logic, making the query an more dynamic.

## Corner Cases

What if we run query has more attributes, e.g.:
```
GET test_index-1/_search/template
{
  "id": "synonym-graph-search",
  "params": {
    "query_string": "suffix very important thing prefix",
    "new_attr": "123"
  }
}
```
Works as expected!

When `query_string` is `null`:
```
GET test_index-1/_search/template
{
  "id": "synonym-graph-search",
  "params": {
    "query_string": null
  }
}
```
Works!

What if the param is not provided:
```
GET test_index-1/_search/template
{
  "id": "synonym-graph-search",
  "params": {
    "new_attr": "value"
  }
}
```

No error!

What if we provide a list instead of a string:
```
GET test_index-1/_search/template
{
  "id": "synonym-graph-search",
  "params": {
    "query_string": ["this", "Very Important Thing"]
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
    "max_score" : 1.4384103,
    "hits" : [
      {
        "_index" : "test_index-1",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.4384103,
        "_source" : {
          "description" : "This description is about a Very Important Thing and something else.",
          "entity" : "Very Important Thing"
        }
      }
    ]
  }
}
```
Instead of profiding one value we can replace it with a list. Good!

## Metadata of the search template

It would be great to be able to store some metadata with the search template script, e.g. Git commit SHA of the query. I couldn't find a way to do this. A workaround might be to `_name` attribute of the query. E.g.:
```
POST _scripts/synonym-graph-search
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "match": {
          "entity": {
            "_name": "GIT COMMIT SHA",
            "query": "{{query_string}}",
            "boost": 5
          }
        }
      }
    }
  }
}
```

The response:
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
    "max_score" : 1.4384103,
    "hits" : [
      {
        "_index" : "test_index-1",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.4384103,
        "_source" : {
          "description" : "This description is about a Very Important Thing and something else.",
          "entity" : "Very Important Thing"
        },
        "matched_queries" : [
          "GIT COMMIT SHA"
        ]
      }
    ]
  }
}
```

Not great but might be useful.

## Discussion

- Templates doesn't support search index specification.
- Field names can be parameterized, this feature alows to start/stop using a new/old field.
- Search template can be tested in (even in production cluster) independently.
- We can run our query against [multiple search templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/multi-search-template.html). Combine this with the Profile API and performance can be compared. Explain API also is supported.
