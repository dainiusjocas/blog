---
aliases: [elasticsearch-normalizers]
projects: [elasticsearch]
title: A Neat Trick with Elasticsearch Normalizers
authors: [Dainius Jocas]
date: '2020-07-08'
tags: [Elasticsearch]
categories:
  - Elasticsearch
summary: In this article I’ll explain what the normalizer is and show a use case for normalizing URLs.
image:
  caption: "[Photo by ze intenets](https://github.com/borkdude/babashka)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

To analyze textual data Elasticsearch uses analyzers while for keyword analysis there is a thing called normalizer. In this article I'll explain what the normalizer is and show a use case for **normalizing** URLs.

## TL;DR

A neat use case for keyword normalizers is to extract a specific part of the URL.

## Introduction

In Elasticsearch the textual data is represented with two data types: `text` and `keyword`. The `text` type is meant to be used for full-text search use cases while `keyword` is mean for filtering and aggregation. 

### TL;DR About Analyzers

To make a better use of `text` data you can setup the [analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer-anatomy.html) which is a combination of three components: 
- exactly one **tokenizer**,
- zero or more **character filters**,
- zero or more **token filters**.

Basically, an analyzer transforms a single *string* into *words*, e.g. `"This is my text"` can be transformed into `["this", "my", "text"]` which you can read as:
- text is split into tokens by tokenizer,
- each token is lowercased with the a token filter,
- stopwords are removed with another token filter.

### Normalizers

The [documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-normalizers.html) says that:
> Normalizers are similar to analyzers except that they may only emit a single token.

Normalizers can only be applied to `keyword` datatype. But note that only a subset of available filters can be used by normalizer: filter must work on a **per-character basis**.

### Normalizers for Normalizing URL Data

Storing a URL in keyword field allows to filter and aggregate your data per URL. But what if you need to filter and aggregate by just one part of the URL and you have little to no control over the upstream data source? You have a couple of options:
- convince upstream to extract that one part in their code,
- setup a `text` field with an analyzer that produces one token and enable field data.
- setup a `keyword` field with a normalizer.
- give up.

I want to explore the `keyword` option. In the next section I'll show how to setup normalizers for Elasticsearch URLs.

### The Problem

We have URLs without a hostname that were used to query Elasticsearch, e.g.: `/my_search_index/_search?q=elasticsearch` and we need to split it into parts such as: index, operation endpoint, e.g.: `_search` or `_count`, query filters, etc. In the following example I'll focus on the extracting the index part of the URL.

Let's create an index:
```
PUT elasticsearch_url_index
{
  "settings": {
    "index": {
      "analysis": {
        "normalizer": {
          "index_extractor_normalizer": {
            "type": "custom",
            "char_filter": [
              "index_name_extractor"
            ]
          }
        },
        "char_filter": {
          "index_name_extractor": {
            "type": "pattern_replace",
            "pattern": "/(.+)/.*",
            "replacement": "$1"
          }
        }
      }
    }
  },
  "mappings" : {
    "properties": {
      "url": {
        "type": "keyword",
        "fields": {
          "index": {
            "type": "keyword",
            "normalizer": "index_extractor_normalizer"   
          }
        }
      }
    }
  }
}
```

Here we setup the index with a normalizer `index_extractor_normalizer` that has a char filter `index_name_extractor` that uses a regex `pattern_replace` to extract characters between two slashes. Mappings have a property `url` which is of `keyword` type and have a field `index` which is keyword and is set up to use a normalizer `index_extractor_normalizer`. 

Since the normalizer is basically a collection of filters we can use our good old friend `_analyze` API to see how it works.
```
POST elasticsearch_url_index/_analyze
{
  "char_filter": ["index_name_extractor"],
  "text": ["/my_search_index/_search?q=elasticsearch"]
}
```

Produces:
```
{
  "tokens" : [
    {
      "token" : "my_search_index",
      "start_offset" : 0,
      "end_offset" : 40,
      "type" : "word",
      "position" : 0
    }
  ]
}
```
Good, exactly as we wanted: `/my_search_index/_search?q=elasticsearch` => `my_search_index`.

Let's index some data:
```
PUT elasticsearch_url_index/_doc/0
{
  "url": "/my_search_index/_search?q=elasticsearch"
}
```

Let's try to filter URLs by index name:
```
GET elasticsearch_url_index/_search?q=url:my_search_index
```
Produces:
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
No results? What? Oh! Wrong field: `url` instead of `url.index`. Let's try once again:
```
GET elasticsearch_url_index/_search?q=url.index:my_search_index
```
Produces:
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
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "elasticsearch_url_index",
        "_type" : "_doc",
        "_id" : "0",
        "_score" : 0.2876821,
        "_source" : {
          "url" : "/my_search_index/_search?q=elasticsearch"
        }
      }
    ]
  }
}
```
As expected. Cool.

Another neat trick is that we can get out the `index` part of the URL from Elasticsearch using `docvalue_fields` option in a request ,e.g.:
```
GET elasticsearch_url_index/_search?q=url.index:my_search_index
{
  "docvalue_fields": ["url.index"]
}
```
Produces: 
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
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "elasticsearch_url_index",
        "_type" : "_doc",
        "_id" : "0",
        "_score" : 0.2876821,
        "_source" : {
          "url" : "/my_search_index/_search?q=elasticsearch"
        },
        "fields" : {
          "url.index" : [
            "my_search_index"
          ]
        }
      }
    ]
  }
}
```

Observe the part:
```
"fields" : {
  "url.index" : [
    "my_search_index"
  ]
}
```

A neat thing about it is that the `my_search_index` value is not comming form the `_source` of the document. This means that we can use `keywords` and by extension normalized `keywords` to fetch an exact value from ES and not necessarily the one that was sent to Elasticsearch which somewhat solves our dependency from the upstream systems.

## Notes

The setup is done in the Kibana Dev Tools with the Elasticsearch of version 7.7.0. 

The pattern `"/(.+)/.*"` is a bit simplified for presentaiton purposes and doesn't work as expected for URLs with more than 2 slashes, e.g.: `/index/type/_search` would produce `index/type`. You need something a bit more involved like `"/([^/]+)/.*"`.  

## Fin

That is all I wanted to show you today. Hope it might be useful/interesting to someone down the line. Leave comments on a github issue [here](https://github.com/dainiusjocas/blog/issues/7). Cheers!
