---
aliases: [elasticsearch-full-phrase-with-slop]
projects: [elasticsearch]
title: Elasticsearch Full Phrase with Slop
authors: [Dainius Jocas]
date: '2023-01-29'
tags: [clojure]
categories:
- Elasticsearch
summary: Match entire document with slop
image:
  caption: "Photo generated with the stable diffusion 2 from the blog title"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

## Intro

Recently I've done a little Elasticsearch based demo where there was one requirement that I believe is worth sharing: match the entire text where words could be swapped. 
This post is divided into 2 parts: requirements for the demo, and several implementation options.

## Requirements

The simplified version of the relevant requirement goes something like this:

1. Documents are message bodies in a chat app, 
2. The text should be normalized: case-insensitive; diacritics insensitive.
3. We need to match a message body with other messages where any two words are swapped.

Example message
```text
this is my message
```
should match
```text
is this my message
```

At first, this looks like nothing fascinating, just set up an [analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-analyzers.html) with the required [token filters](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-tokenfilters.html`) and do a [phrase_match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html) with a slop equal to 2.
The tricky bit is that we also need to take into account the **length of the message**, because with short query messages we might match longer messages which is what we don't want.
E.g. We don't want that the query `is this my message` would match `this is my message which is longer`.

## Implementation strategies

We'll discuss 2 implementation strategies: fingerprinting based on token count and a clever trick leveraging the text analysis pipeline.

### Fingerprint

An instinctive approach is to [fingerprint](https://www.elastic.co/guide/en/elasticsearch/reference/master/fingerprint-processor.html) the message text. 
A simplest fingerprint could just be the count of tokens. 
Elasticsearch conveniently offers a [token count field type](https://www.elastic.co/guide/en/elasticsearch/reference/current/token-count.html).

One serious downside of this approach is that at the query time we'd have to get the count of tokens of the query.
This means either one additional round-trip to Elasticsearch to get the token count, e.g. [`_analyze` API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-analyze.html) or using some `script`.
Let's rule out the round trip to the `_analyze` API approach.
Then how to implement the `script` to get query token count?
The [Chat GPT](https://chat.openai.com/chat#) was helpful to hint at a possible `script` based solution :)

![Script](query-token-count.png)

A significant problem of the above approach is that the text is split by a whitespace character (or something that [`split`](https://docs.oracle.com/javase/8/docs/api/java/util/regex/Pattern.html#split%2Djava.lang.CharSequence%2D) accepts).
Any seasoned search engineer immediately notices that the text is not tokenized by the same analyzer that was used to tokenize the text during the index-time for the token counting.
Of, course we could set up both `tokenizers` to work the same way, but that approach seems somewhat fragile because part of the logic is in index mappings and other part is in the script that lives in the query constructor which is probably inside your application.

### Text analysis

We could also be more clever about how to ensure that the length of the matched text is equal.
My strategy to ensure that length is equal is to require both conditions to be true:
1. Match the text **from the beginning of the string** with a `phrase_match` query with a slop=2.
2. Match the **reversed** text from the beginning of the string with a `phrase_match` query with a slop=2.

This gives us 2 puzzles to solve:
1. How to match exactly from the beginning of the text?
2. How to reverse the text?

To ensure the matching from the beginning we could insert a synthetic `PREFIX` token at the position 0.
(Of course, I've been thinking about leveraging [`span_first`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-span-first-query.html) query, but it requires another inner term level span query.
This would make the query construction complicated because tokenization should be done inside your application.)

For the text reversal the idea is to:
1. Do not use a tokenizer to tokenize the text, i.e. use the [keyword tokenizer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-keyword-tokenizer.html)
2. Use the [reverse](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-reverse-tokenfilter.html) token filter on the entire text,
3. Split the text into tokens using the [word delimiter graph token filter](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-word-delimiter-graph-tokenfilter.html#analysis-word-delimiter-graph-tokenfilter).
4. Add a synthetic token at the position 0.

Let's work out an example from requirements: `this is my message` should match `this is my message` but should not match `this is my message which is longer`.

The `messages` index config:
```json
{
  "settings": {
    "number_of_replicas": 0,
    "number_of_shards": 1,
    "analysis": {
      "analyzer": {
        "normal_direction_analyzer": {
          "type": "custom",
          "tokenizer": "keyword",
          "filter": [
            "token_splitter",
            "prefixer"
          ]
        },
        "backwards_direction_analyzer": {
          "type": "custom",
          "tokenizer": "keyword",
          "filter": [
            "reverse",
            "token_splitter",
            "prefixer"
          ]
        }
      },
      "filter": {
        "token_splitter": {
          "type": "word_delimiter_graph",
          "split_on_case_change": false,
          "split_on_numerics": false
        },
        "inject_prefix": {
          "type": "pattern_replace",
          "pattern": "^(.*)$",
          "replacement": "PREFIX $1"
        },
        "prefixer": {
          "type": "condition",
          "filter": [
            "inject_prefix",
            "token_splitter"
          ],
          "script": {
            "source": "token.getPosition() == 0"
          }
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "body": {
        "type": "text",
        "fields": {
          "normal_direction": {
            "type": "text",
            "analyzer": "normal_direction_analyzer"
          },
          "backwards_direction": {
            "type": "text",
            "analyzer": "backwards_direction_analyzer"
          }
        }
      }
    }
  }
}
```

Let's index 2 documents:
```text
POST _bulk?refresh=true
{ "index" : { "_index" : "messages", "_id" : "1" } }
{ "body": "this is my message" }
{ "index" : { "_index" : "messages", "_id" : "2" } }
{ "body": "this is my message which is longer" }
```

Let's query the `messages` index:
```json
{
  "query": {
    "bool": {
      "must": [
        {
          "match_phrase": {
            "body.normal_direction": {
              "query": "is this my message",
              "slop": 2
            }
          }
        },
        {
          "match_phrase": {
            "body.backwards_direction": {
              "query": "is this my message",
              "slop": 2
            }
          }
        }
      ],
      "_name": "full phrase with accounted length"
    }
  }
}
```

The response:
```json
{
  "took": 2,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 1,
      "relation": "eq"
    },
    "max_score": 0.46947598,
    "hits": [
      {
        "_index": "messages",
        "_id": "1",
        "_score": 0.46947598,
        "_source": {
          "body": "this is my message"
        },
        "matched_queries": [
          "full phrase with accounted length"
        ]
      }
    ]
  }
}
```

The response is as expected which does match the shorter but not the longer message body.

Downsides of the approach:
- The text is indexed 2 times. But it is OK to index your text analyzed in multiple ways, e.g. stemming.
- Analyzers got somewhat complicated. But the complicated bit is isolated on dealing with the first token, which can be illustrated with a couple of examples with the `_analyze` API. 
- The text length is limited to the `Integer.MAX_VALUE` which is 2147483647 characters. But ~2 GB should be enough.
- "Tokenization" is done with the `word_delimiter_graph`. But it is a standard Lucene feature that you should learn anyway.
- Also, all the gotchas of the `slop` are relevant, e.g. if one token was dropped/added from/to the query, then there still would be a match.

A nice thing is that this solution is contained within the text analysis pipeline: no ingest pipelines, no scripting in queries, etc.

## Summary

In this post we've defined a problem of matching entire documents with some flexibility in terms of token position changes.
We've discussed 2 approaches: fingerprinting and leveraging text analysis pipeline to account for the text length.
IMO, both approaches are somewhat hacky and have their downsides.
I've picked to work out the text analysis approach, and I've got my demo done.

Let me know how you would approach this problem in the comments below.
