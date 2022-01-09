---
aliases: [entity-resolution]
projects: [search, elasticsearch]
title: Entity Resolution with Elasticsearch
authors: [Dainius Jocas]
date: '2021-02-08'
tags: [elasticsearch]
categories:
  - elasticsearch
summary: Entity Resolution implementation with nothing more than Elasticsearch
image:
  caption: "[Photo by jenkov.com](http://tutorials.jenkov.com/java-concurrency/volatile.html)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

## TL;DR

The entity resolution can be viewed as a search application and if the requirements are rather simplistic then Elasticsearch is good enough. 

## Introduction

Say that our company have a curated registry of organizations (only organization names with the number of employees) indexed in OpenSearch/Elasticsearch.
Our employer acquired a direct competitor with their own registry (only organization names with their addresses) of organizations and we want to integrate the new registry to our old registry.
Our task is to iterate through the new registry record by record and try to map them to our "Golden registry".
Unfortunately, the only overlapping data is organization names and our matching needs to be based mostly on the organization name. 

## Requirements

Let's say that we have these similarity rules in the descending order for the sample query "Apple":

1. Exact match, e.g. "Apple"
2. Exact first word match, e.g. "Apple Computers"
3. Exact not first word match, e.g.	"Big Apple Company"
4. Partial first word match - start, e.g.	"Applesauce Company"
5. Partial first word match - end, e.g.	"Pineapple Manufacturing"
6. Partial not first word match - end, e.g.	"Canadian Bakeapple"
7. Fuzzy, e.g. "Apply"

Also, we want to be able to add another signal that could change the ordering, 
e.g. the number of employees: the more employees the matching company has the higher its matching score.

"Applesauce Company" with 100 employees should be lower than "Pineapple Manufacturing" with 10000 employees despite that
the rule of a *partial first word match at the start* has higher similarity than the *partial not first word match at the end*.

### Additional notes for development

To make the development easier we also want:

- to know which rule matched for each hit,
- give each rule a normalized score,
- matching of strings is case-insensitive,
- Index contains sample company list

Indexing organizations can be done with:
```text
PUT organizations/_bulk
{ "index" : { "_id" : "1" } }
{ "name" : "Apple" }
{ "index" : { "_id" : "2" } }
{ "name" : "Apple Computers" }
{ "index" : { "_id" : "3" } }
{ "name" : "Big Apple Company" }
{ "index" : { "_id" : "4" } }
{ "name" : "Applesauce Company" }
{ "index" : { "_id" : "5" } }
{ "name" : "Pineapple Manufacturing" }
{ "index" : { "_id" : "6" } }
{ "name" : "Canadian Bakeapple" }
{ "index" : { "_id" : "7" } }
{ "name" : "Apply" }
```

## Implementation Strategy

Let's split the implementation into 3 parts:
- implement the string matching requirements,
- implement the scoring signal based on the number of employees,
- fine-tune the scoring function.

## String Matching Implementation

In the following sections we'll implement all string matching signals with the OpenSearch analysis facilities.
For each rule we will define a subfield of the `name` attribute with an analyzer and a corresponding query clause.

### Exact match

Exact matching is rather simple to implement, just use `keyword` datatype with a normalizer that lowercases the string.

Index configuration:
```kibana
PUT organizations
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercased_keyword": {
          "type": "custom",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "keyword_lowercased": {
            "type": "keyword",
            "normalizer": "lowercased_keyword"
          }
        }
      }
    }
  }
}
```

Query clause:
```kibana
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "name.keyword_lowercased": {
            "value": "Apple",
            "_name": "exact_match"
          }
        }
      },
      "boost": 7
    }
  }
}
```

Matches exactly one organization:
```json
{
  "took" : 3,
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
    "max_score" : 7.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 7.0,
        "_source" : {
          "name" : "Apple"
        },
        "matched_queries" : [
          "exact_match"
        ]
      }
    ]
  }
}
```

### Exact first word match

With this requirement we want the query to match the first word of the organization name.
Of course, this is a bit unrealistic because organization name might contain more than one word.

We can implement the requirement by simply indexing only the first token.
To achieve it we could use the [`limit`](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-limit-token-count-tokenfilter.html) token filter.

Index configuration:
```text
PUT organizations
{
  "settings": {
    "analysis": {
      "analyzer": {
        "standard_one_token_limit": {
          "tokenizer": "standard",
          "filter": [
            "limit"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "first_token": {
            "type": "text",
            "analyzer": "standard_one_token_limit"
          }
        }
      }
    }
  }
}
```

Query:
```text
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "name.first_token": {
            "value": "Apple",
            "_name": "Exact first word match"
          }
        }
      },
      "boost": 6
    }
  }
}
```

Hits:
```json
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
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 6.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 6.0,
        "_source" : {
          "name" : "Apple"
        },
        "matched_queries" : [
          "Exact first word match"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 6.0,
        "_source" : {
          "name" : "Apple Computers"
        },
        "matched_queries" : [
          "Exact first word match"
        ]
      }
    ]
  }
}
```

Note that not only "Apple Computers" matched but also "Apple" matched.
This makes sense because "Apple" has only one word it is *the first word*.

If we wanted to exclude the "Apple" match we could combine the `Exact first word match` with the `Exact match` and construct a bool query, e.g.:
```text
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "bool": {
          "_name": "Exact first word match",
          "must": [
            {
              "term": {
                "name.first_token": {
                  "value": "Apple"
                }
              }
            }
          ],
          "must_not": [
            {
              "term": {
                "name": {
                  "value": "Apple"
                }
              }
            }
          ]
        }
      },
      "boost": 6
    }
  }
}
```

For other cases we could do something similar.

### Exact not first word match

E.g.: query "Apple" should match "Big Apple Company".
The interesting bit is how to differentiate this rule from the `Exact first word match`?
As an implementation we could create a boolean query that filters out the first word match.
Another way to implement the is to not index the first word of the organization name.
A slight complication with this approach is that the query time analyzer should be different from 
the index time analyzer to prevent the removal of the first (and probably the only) word from the query. 
Let's proceed with the second approach.

```text
PUT organizations
{
  "settings": {
    "analysis": {
      "analyzer": {
        "tokenized": {
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        },
        "tokenized_without_first_word": {
          "tokenizer": "standard",
          "filter": [
            "remove_first_word",
            "lowercase"
          ]
        }
      },
      "filter": {
        "remove_first_word": {
          "type": "predicate_token_filter",
          "script": {
            "source": "token.position != 0"
          }
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "tokenized_without_first_word": {
            "type": "text",
            "analyzer": "tokenized_without_first_word",
            "search_analyzer": "tokenized"
          }
        }
      }
    }
  }
}
```

Query:

```text
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "name.tokenized_without_first_word": {
            "query": "Apple",
            "_name": "Exact not first word match"
          }
        }
      },
      "boost": 5
    }
  }
}
```

Hits
```json
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
    "max_score" : 6.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 6.0,
        "_source" : {
          "name" : "Big Apple Company"
        },
        "matched_queries" : [
          "Exact not first word match"
        ]
      }
    ]
  }
}
```

### Partial first word match - start

E.g. Query "Apple" should match "Applesauce Company".
Probably we could get away with a simple prefix query, but if we want to make it fast for larger indices we should do edge n-grams.
The idea is to keep only this first word, and apply edge n-gram filter on it.
Of course, query analyzer should be a standard analyzer to prevent false hits from matching a couple of first characters from the query.

Index configuration:
```text
PUT organizations
{
  "settings": {
    "analysis": {
      "analyzer": {
        "standard_first_token_limit": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase"
          ]
        },
        "standard_first_token_limit_edge_ngrams": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "1_10_edgegrams",
            "lowercase"
          ]
        }
      },
      "filter": {
        "1_10_edgegrams": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 10
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "first_token_edge_ngram": {
            "type": "text",
            "analyzer": "standard_first_token_limit_edge_ngrams",
            "search_analyzer": "standard_first_token_limit"
          }
        }
      }
    }
  }
}
```

Query:
```text
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "name.first_token_edge_ngram": {
            "query": "Apple",
            "_name": "Partial first word match - start"
          }
        }
      },
      "boost": 4
    }
  }
}
```

Hits:
```json
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
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 4.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 4.0,
        "_source" : {
          "name" : "Apple"
        },
        "matched_queries" : [
          "Partial first word match - start"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 4.0,
        "_source" : {
          "name" : "Apple Computers"
        },
        "matched_queries" : [
          "Partial first word match - start"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 4.0,
        "_source" : {
          "name" : "Applesauce Company"
        },
        "matched_queries" : [
          "Partial first word match - start"
        ]
      }
    ]
  }
}
```

Note that "Apple" and "Apple Computers" also matched.
We could prevent matching by doing the bool query with must_not match on the first token.

### Partial first word match - end

E.g. Query "Apple" should match "Pineapple Manufacturing".
In other words, the query string should be the ending of the first word of the organization name.
The idea is to take the first word, reverse it, apply edge n-grams, reverse those n-grams.
The search time analyzer should be standard Analyzer.

Index configuration:
```text
PUT organizations
{
  "settings": {
    "analysis": {
      "analyzer": {
        "standard_first_token_limit": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase"
          ]
        },
        "standard_one_token_limit_endings": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase",
            "reverse",
            "1_10_edgegrams",
            "reverse"
          ]
        }
      },
      "filter": {
        "1_10_edgegrams": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 10
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "first_word_end_ngrams": {
            "type": "text",
            "analyzer": "standard_one_token_limit_endings",
            "search_analyzer": "standard_first_token_limit"
          }
        }
      }
    }
  }
}
```

Query:
```text
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "name.first_word_end_ngrams": {
            "query": "Apple",
            "_name": "Partial first word match - end"
          }
        }
      },
      "boost": 3
    }
  }
}
```

Hits:
```text
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
      "value" : 3,
      "relation" : "eq"
    },
    "max_score" : 3.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 3.0,
        "_source" : {
          "name" : "Apple"
        },
        "matched_queries" : [
          "Partial first word match - end"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 3.0,
        "_source" : {
          "name" : "Apple Computers"
        },
        "matched_queries" : [
          "Partial first word match - end"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 3.0,
        "_source" : {
          "name" : "Pineapple Manufacturing"
        },
        "matched_queries" : [
          "Partial first word match - end"
        ]
      }
    ]
  }
}
```

### Partial not first word match - end

E.g. query "Apple" should match	"Canadian Bakeapple".
The idea is to tokenize, lowercase, remove first word, reverse tokens, create edge n-grams, reverse back.
Search time analyzer should be standard analyzer.

Index configuration:
```text
PUT organizations
{
  "settings": {
    "analysis": {
      "analyzer": {
        "standard_first_token_limit": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase"
          ]
        },
        "standard_remaining_token_endings": {
          "tokenizer": "standard",
          "filter": [
            "remove_first_word",
            "lowercase",
            "reverse",
            "1_10_edgegrams",
            "reverse"
          ]
        }
      },
      "filter": {
        "1_10_edgegrams": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 10
        },
        "remove_first_word": {
          "type": "predicate_token_filter",
          "script": {
            "source": "token.position != 0"
          }
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "remaining_words_end_ngrams": {
            "type": "text",
            "analyzer": "standard_remaining_token_endings",
            "search_analyzer": "standard_first_token_limit"
          }
        }
      }
    }
  }
}
```

Query:
```text
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "name.remaining_words_end_ngrams": {
            "query": "Apple",
            "_name": "Partial not first word match - end"
          }
        }
      },
      "boost": 2
    }
  }
}
```

Hits:
```json
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
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 2.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 2.0,
        "_source" : {
          "name" : "Big Apple Company"
        },
        "matched_queries" : [
          "Partial not first word match - end"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "6",
        "_score" : 2.0,
        "_source" : {
          "name" : "Canadian Bakeapple"
        },
        "matched_queries" : [
          "Partial not first word match - end"
        ]
      }
    ]
  }
}
```

### Fuzzy

E.g. query "Apple" should match "Apply".
It is pretty much the exact match but with some fuzziness allowed.

Index configuration:
```text
PUT organizations
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercased_keyword": {
          "type": "custom",
          "filter": [
            "lowercase"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "keyword_lowercased": {
            "type": "keyword",
            "normalizer": "lowercased_keyword"
          }
        }
      }
    }
  }
}
```

Query:
```text
GET organizations/_search
{
  "query": {
    "constant_score": {
      "filter": {
        "match": {
          "name": {
            "query": "Apple",
            "fuzziness": 1,
            "_name": "fuzzy_1"
          }
        }
      },
      "boost": 1
    }
  }
}
```

Hits:
```json
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
      "value" : 2,
      "relation" : "eq"
    },
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "name" : "Apple"
        },
        "matched_queries" : [
          "fuzzy_1"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "7",
        "_score" : 1.0,
        "_source" : {
          "name" : "Apply"
        },
        "matched_queries" : [
          "fuzzy_1"
        ]
      }
    ]
  }
}
```

In some circumstances we might allow fuzziness set to more than 1.

## Let's combine the pieces together

Phew, that was pretty involved.
Let's combine the text matching into one setup.

Index configuration is constructed by merging analysis components and adding fields from all the examples:
```text
PUT organizations
{
  "settings": {
    "analysis": {
      "normalizer": {
        "lowercased_keyword": {
          "type": "custom",
          "filter": [
            "lowercase"
          ]
        }
      },
      "filter": {
        "1_10_edgegrams": {
          "type": "edge_ngram",
          "min_gram": 1,
          "max_gram": 10
        },
        "remove_first_word": {
          "type": "predicate_token_filter",
          "script": {
            "source": "token.position != 0"
          }
        }
      },
      "analyzer": {
        "standard_one_token_limit": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase"
          ]
        },
        "tokenized": {
          "tokenizer": "standard",
          "filter": [
            "lowercase"
          ]
        },
        "tokenized_without_first_word": {
          "tokenizer": "standard",
          "filter": [
            "remove_first_word",
            "lowercase"
          ]
        },
        "standard_first_token_limit_edge_ngrams": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase",
            "1_10_edgegrams"
          ]
        },
        "standard_first_token_limit": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase"
          ]
        },
        "standard_remaining_token_endings": {
          "tokenizer": "standard",
          "filter": [
            "remove_first_word",
            "lowercase",
            "reverse",
            "1_10_edgegrams",
            "reverse"
          ]
        },
        "standard_one_token_limit_endings": {
          "tokenizer": "standard",
          "filter": [
            "limit",
            "lowercase",
            "reverse",
            "1_10_edgegrams",
            "reverse"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "name": {
        "type": "keyword", 
        "fields": {
          "first_token": {
            "type": "text",
            "analyzer": "standard_one_token_limit"
          },
          "tokenized_without_first_word": {
            "type": "text",
            "analyzer": "tokenized_without_first_word",
            "search_analyzer": "tokenized"
          },
          "first_token_edge_ngram": {
            "type": "text",
            "analyzer": "standard_first_token_limit_edge_ngrams",
            "search_analyzer": "standard_first_token_limit"
          },
          "first_word_end_ngrams": {
            "type": "text",
            "analyzer": "standard_one_token_limit_endings",
            "search_analyzer": "standard_first_token_limit"
          },
          "remaining_words_end_ngrams": {
            "type": "text",
            "analyzer": "standard_remaining_token_endings",
            "search_analyzer": "standard_first_token_limit"
          },
          "keyword_lowercased": {
            "type": "keyword",
            "normalizer": "lowercased_keyword"
          }
        }
      }
    }
  }
}
```

Here 

The query is constructed from all the previous examples simply by adding constant score queries to the dis_max query:
```text
GET organizations/_search
{
  "query": {
    "dis_max": {
      "queries": [
        {
          "constant_score": {
            "filter": {
              "term": {
                "name.keyword_lowercased": {
                  "value": "Apple",
                  "_name": "exact_match"
                }
              }
            },
            "boost": 7
          }
        },
        {
          "constant_score": {
            "filter": {
              "match": {
                "name.first_token": {
                  "query": "Apple",
                  "_name": "Exact first word match"
                }
              }
            },
            "boost": 6
          }
        },
        {
          "constant_score": {
            "filter": {
              "match": {
                "name.tokenized_without_first_word": {
                  "query": "Apple",
                  "_name": "Exact not first word match"
                }
              }
            },
            "boost": 5
          }
        },
        {
          "constant_score": {
            "filter": {
              "match": {
                "name.first_token_edge_ngram": {
                  "query": "Apple",
                  "_name": "Partial first word match - start"
                }
              }
            },
            "boost": 4
          }
        },
        {
          "constant_score": {
            "filter": {
              "match": {
                "name.first_word_end_ngrams": {
                  "query": "Apple",
                  "_name": "Partial first word match - end"
                }
              }
            },
            "boost": 3
          }
        },
        {
          "constant_score": {
            "filter": {
              "match": {
                "name.remaining_words_end_ngrams": {
                  "query": "Apple",
                  "_name": "Partial not first word match - end"
                }
              }
            },
            "boost": 2
          }
        },
        {
          "constant_score": {
            "filter": {
              "match": {
                "name": {
                  "query": "Apple",
                  "fuzziness": 1,
                  "_name": "fuzzy_1"
                }
              }
            },
            "boost": 1
          }
        }
      ]
    }
  }
}
```

Finally, the list of organizations ranked by the name similarity:
```json
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
      "value" : 7,
      "relation" : "eq"
    },
    "max_score" : 7.0,
    "hits" : [
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 7.0,
        "_source" : {
          "name" : "Apple"
        },
        "matched_queries" : [
          "exact_match",
          "Partial first word match - end",
          "Partial first word match - start",
          "Exact first word match",
          "fuzzy_1"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "2",
        "_score" : 6.0,
        "_source" : {
          "name" : "Apple Computers"
        },
        "matched_queries" : [
          "Partial first word match - end",
          "Partial first word match - start",
          "Exact first word match"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "3",
        "_score" : 5.0,
        "_source" : {
          "name" : "Big Apple Company"
        },
        "matched_queries" : [
          "Exact not first word match",
          "Partial not first word match - end"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "4",
        "_score" : 4.0,
        "_source" : {
          "name" : "Applesauce Company"
        },
        "matched_queries" : [
          "Partial first word match - start"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "5",
        "_score" : 3.0,
        "_source" : {
          "name" : "Pineapple Manufacturing"
        },
        "matched_queries" : [
          "Partial first word match - end"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "6",
        "_score" : 2.0,
        "_source" : {
          "name" : "Canadian Bakeapple"
        },
        "matched_queries" : [
          "Partial not first word match - end"
        ]
      },
      {
        "_index" : "organizations",
        "_type" : "_doc",
        "_id" : "7",
        "_score" : 1.0,
        "_source" : {
          "name" : "Apply"
        },
        "matched_queries" : [
          "fuzzy_1"
        ]
      }
    ]
  }
}
```

Nice, from the most similar "Exact match" all the way down to the "fuzzy" matches.

I believe this post got a bit too long to continue working out the remaining requirements.
I'll finish in the future post.
Also, it shouldn't be too hard to load bigger dataset to test our entity resolution algorithm, something like [Companies House registry](http://download.companieshouse.gov.uk/en_output.html).

## Summary

I hope this exercise was interesting.
To sum up:
- I've demonstrated how to do the text analysis tricks that would shape the organization name string according to your requirements.
- As a way to normalize search scores I've used the `constant_score` query. To come up with this trick is not very obvious because Elasticsearch scoring by default is unbounded and therefore not very well suited 
to satisfy the requirements of matching semi structured short strings.
- Another neat trick was to use the `dis_max` query to calculate the score by taking only the most significant similarity signal.
In this way we've prevented the situations where less similar matches would go up because they matched several less significant 
similarity rules and their score were added up.
- Also, by using `_name` parameters we get which similarity rules matched despite the fact that we've took the score 
of the highest scoring similarity rule. These flags might be used for further rescoring in your application.

In case of any questions don't hesitate to leave comments under this Github [issue](https://github.com/dainiusjocas/blog/issues/25).

## Footnotes

[^docker]: https://opensearch.org/docs/latest/opensearch/install/docker/#sample-docker-compose-file-for-development
