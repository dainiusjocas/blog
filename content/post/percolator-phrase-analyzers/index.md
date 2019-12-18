---
aliases: [percolator-phrase-analyzers]
projects: [elasticsearch]
title: Elasticsearch Percolator and Text Analyzers
authors: [Dainius Jocas]
date: '2019-12-18'
tags: [elasticsearch, percolator]
categories:
  - elasticsearch
  - percolator
summary: Example on how to use analyzers with the
image:
  caption: "[Photo by www.rohde-schwarz.com](https://www.rohde-schwarz.com/cz/products/test-and-measurement/signal-spectrum-analyzers/pg_overview_63665.html)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

This time I need to percolate texts with different analyzers for index and search analyzers.

Let's elaborate a bit on [previous article](https://www.jocas.lt/blog/post/es-percolator-phrase-highlight/) and explicitly declare analyzers to use.

Define index:
```
PUT /my-index
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "analyzer": "standard", 
        "term_vector": "with_positions_offsets"
      },
      "query": {
        "type": "percolator"
      }
    }
  }
}
```

Then define 2 slightly different percolator queries (notice the difference between `"bonsai tree"` and `"bonsai, tree"`).

```
PUT /my-index/_doc/1?refresh
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "bonsai tree",
        "analyzer": "standard"
      }
    }
  }
}

PUT /my-index/_doc/2?refresh
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "bonsai, tree",
        "analyzer": "standard"
      }
    }
  }
}
```

Let's percolate:
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

This yields:
```
{
  "took" : 80,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 0.26152915,
    "hits" : [
      {
        "_index" : "my-index",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 0.26152915,
        "_source" : {
          "query" : {
            "match_phrase" : {
              "message" : {
                "query" : "bonsai, tree",
                "analyzer" : "standard"
              }
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
      },
      {
        "_index" : "my-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.26152915,
        "_source" : {
          "query" : {
            "match_phrase" : {
              "message" : {
                "query" : "bonsai tree",
                "analyzer" : "standard"
              }
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
As expected: 2 documents matched.

But now lets change the analyzer of the second percolation query to `whitespace`:
```
PUT /my-index/_doc/2?refresh
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "bonsai, tree",
        "analyzer": "whitespace"
      }
    }
  }
}
```

Run the percolator:
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
This yields:
```
{
  "took" : 5,
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
              "message" : {
                "query" : "bonsai tree",
                "analyzer" : "standard"
              }
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
As expected: only 1 percolator query matched our input.

## Phrases with Stopwords

Say, we have a phrase `"bonsai is tree"` and we percolate text `A new bonsai in tree in the office` with the `standard` analyzer for indexing and `english` for search analyzer. There should be no matches. Let's try:
```
DELETE my-index

PUT /my-index
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "analyzer": "standard",
        "search_analyzer": "english",
        "term_vector": "with_positions_offsets"
      },
      "query": {
        "type": "percolator"
      }
    }
  }
}

PUT /my-index/_doc/1?refresh
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "bonsai is tree"
      }
    }
  }
}


GET /my-index/_search?
{
  "query": {
    "percolate": {
      "field": "query",
      "document": {
        "message": "A new bonsai in tree in the office"
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

And, surprisingly, this yields:
```
{
  "took" : 2,
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
              "message" : {
                "query" : "bonsai is tree"
              }
            }
          }
        },
        "fields" : {
          "_percolator_document_slot" : [
            0
          ]
        }
      }
    ]
  }
}
```

We have a match! Also notice that the highlighter is broken!

The problem that these two analyzers have different stopword lists (no stopwords for `standard` and several English stopwords for `english` analyzer) and the phrase contains a stopword that is not shared between analyzers.

Let's fix this surprise with [`search_quote_analyzer`](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer.html#search-quote-analyzer).

```
DELETE my-index

PUT /my-index
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text",
        "analyzer": "standard",
        "search_analyzer": "english",
        "search_quote_analyzer": "standard",
        "term_vector": "with_positions_offsets"
      },
      "query": {
        "type": "percolator"
      }
    }
  }
}

PUT /my-index/_doc/1?refresh
{
  "query": {
    "match_phrase": {
      "message": {
        "query": "bonsai is tree"
      }
    }
  }
}

GET /my-index/_search?
{
  "query": {
    "percolate": {
      "field": "query",
      "document": {
        "message": "A new bonsai in tree in the office"
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
No hits, as expected.

Let's check if the expected behaviour is still there:
```
GET /my-index/_search?
{
  "query": {
    "percolate": {
      "field": "query",
      "document": {
        "message": "A new bonsai is tree in the office"
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
    "max_score" : 0.39229375,
    "hits" : [
      {
        "_index" : "my-index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 0.39229375,
        "_source" : {
          "query" : {
            "match_phrase" : {
              "message" : {
                "query" : "bonsai is tree"
              }
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
            "A new <em>bonsai is tree</em> in the office"
          ]
        }
      }
    ]
  }
}
```

Good. Even the highlighting works.
