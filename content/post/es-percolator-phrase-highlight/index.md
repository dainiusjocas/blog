---
aliases: [es-percolator-phrase-highlight]
projects: [elasticsearch]
title: Phrase Highlighting with the Elasticsearch Percolator
authors: [Dainius Jocas]
date: '2019-12-18'
tags: [elasticsearch, search]
categories:
  - elasticsearch
  - percolator
summary: Investigating the Elasticsearch percolator.
image:
  caption: "[Photo by www.badgerbroscoffee.com](https://images.app.goo.gl/32WoDFDQp6wm1F7Q9)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

If you google `How can you match a long query text to a short text field?` it will point you to the [Stack Overflow page](https://stackoverflow.com/questions/51865747/elasticsearch-match-long-query-text-to-short-field) [or here](https://discuss.elastic.co/t/match-long-query-text-to-short-field/144584/3) where the answer is to use [Elasticsearch Percolator]().

My search items are phrases meaning that it should match all terms in order. Let's create a sample setup in Kibana (v7.5) Dev dashboard.

1. Create an index for percolation:
```
PUT /my-index
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "term_vector": "with_positions_offsets"
      },
      "query": {
        "type": "percolator"
      }
    }
  }
}
```

Note on `"term_vector": "with_positions_offsets"`: this allows [Fast Vector Highlighter](https://www.elastic.co/guide/en/elasticsearch/reference/6.6/search-request-highlighting.html#fast-vector-highlighter) to highlight combined phrase not just separate qeury terms. 

2. Store one phrase query:
```
PUT /my-index/_doc/1?refresh
{
  "query": {
    "match_phrase": {
      "message": "bonsai tree"
    }
  }
}
```

3. Percolate a document:
```
GET /my-index/_search?
{
  "query": {
    "percolate": {
      "field": "query",
      "document": {
        "message": "A new bonsai tree in the office"
      }
    }
  },
  "highlight": {
    "fields": {
      "message": {
        "type": "fvh"
      }
    }
  }
}
```
Note on `"type": "fvh"`: this instructs Elasticsearch to use the Fast Vector Highlighter.

The query yields:

```
{
  "took" : 23,
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
    "max_score" : 0.26152915,
    "hits" : [
      {
        "_index" : "my-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.26152915,
        "_source" : {
          "query" : {
            "match_phrase" : {
              "message" : "bonsai tree"
            }
          }
        },
        "fields" : {
          "_percolator_document_slot" : [
            0
          ]
        },
        "highlight" : {
          "message" : [
            "A new <em>bonsai tree</em> in the office"
          ]
        }
      }
    ]
  }
}
```

As we see highlighter correctly marker the search phrase.

## Storing additional data with percolator queries

Percolation result can be used to connect pieces of information in your system, e.g. store a `subscriber_email` attribute of the user that wants to be notified when the query matches along with the percolator query.

```
PUT /my-index/_doc/1?refresh
{
  "query": {
    "match_phrase": {
      "message": "bonsai tree"
    }
  },
  "subscriber_email": "subscriber_email@example.com"
}
```

Then query:
```
GET /my-index/_search?
{
  "query": {
    "percolate": {
      "field": "query",
      "document": {
        "message": "A new bonsai tree in the office"
      }
    }
  },
  "highlight": {
    "fields": {
      "message": {
        "type": "fvh"
      }
    }
  }
}
```

This query yields:

```
{
  "took" : 10,
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
    "max_score" : 0.26152915,
    "hits" : [
      {
        "_index" : "my-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.26152915,
        "_source" : {
          "query" : {
            "match_phrase" : {
              "message" : "bonsai tree"
            }
          },
          "subscriber_email" : "subscriber_email@example.com"
        },
        "fields" : {
          "_percolator_document_slot" : [
            0
          ]
        },
        "highlight" : {
          "message" : [
            "A new <em>bonsai tree</em> in the office"
          ]
        }
      }
    ]
  }
}

```

Now, take the email under the `"subscriber_email"` from the response and send an email with the highlight.
