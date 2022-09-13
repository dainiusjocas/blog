---
aliases: [tricks-with-elasticsearch-completion-suggesters]
projects: [search, elasticsearch, opensearch]
title: Tricks with Elasticsearch Completions Suggesters
authors: [Dainius Jocas]
date: '2022-09-13'
tags: [elasticsearch, opensearch]
categories:
  - elasticsearch
summary: The missing pieces of the Elasticsearch documentation
image:
  caption: "[Stable diffusion on search suggestions](https://github.com/CompVis/stable-diffusion)"
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

Elasticsearch text analyzers can supercharge search suggesters. 

{{< toc >}}

## Introduction

So, you are a [search engineer](https://opensourceconnections.com/blog/2020/07/16/what-is-a-relevance-engineer/) that happily uses [Elasticsearch Completion Suggester](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#completion-suggester) feature: lightning speed prefix suggestions works just like a charm[^lucene].

But one day the product manager comes to you with a requirement: `could we also suggest if users start typing a word from the middle of the suggested string?`. Of course, the deadline is yesterday, as always. On top he adds that he doesn't care whether that makes sense from search engineers perspective, it must be done[^1]. You try to argue that it will take forever to change upstream indexing pipeline because it is owned by another department of your company. PM doesn't blink.

## Requirements

Your app suggests artist names. The problematic artist currently rocking in the charts and attracting a lot of attention is `Britney Spears & Elton John`. The PM specifies that it is needed that artists name and surname should be the source of suggestions. This means and that all `b`, `s`, `e`, `j` should suggest that artist.

The actual song {{< youtube 8hLtlzkoGPk >}}.

## Implementation

Generating multiple strings upstream is not an option due to the time constraints (because in a week the song will not be in the charts anymore). Also, the team discards generation of multiple strings using an [ingest processor](https://www.elastic.co/guide/en/elasticsearch/reference/master/ingest-processors.html) because nobody in our team likes to work with them. In the anemic Elasticsearch [documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#_parameters_for_completion_fields_2) there is a hint that using text analyzers (stopwords are mentioned) it is possible to achieve different entry points for suggestions which sounds like the trick that might work for our case. You decide to go to the rabbit hole of the analyzers.

After the technical refinement meeting the notes on implementation of use cases look like:
- `b` -> the classic use case which already works;
- `s` -> from the second word; trick is to use a different analyser at the search time;
- `e` -> from the second entity;
- `j` -> the last word.

Cool, it's time to open the Kibana dev tools and hack. All examples are worked out and tested with version `8.4.1`. 

### Baseline suggestions setup

Nothing fancy here, copy-paste from the documentation verbatim, for the sake of completeness.

```text
DELETE music
PUT music
{
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion"
      }
    }
  }
}

PUT music/_doc/1?refresh
{
  "suggest" : {
    "input": "Britney Spears & Elton John"
  }
}

POST music/_search?filter_path=suggest
{
  "suggest": {
    "artist": {
      "prefix": "b",
      "completion": {
        "field": "suggest"
      }
    }
  }
}
```

And it returns:

```text
{
  "suggest": {
    "artist": [
      {
        "text": "b",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ]
  }
}
```

### Suggest from the second word

The idea here is to have a subfield for suggestions that uses different analyzers for indexing and searching. For indexing we want an analyzer that drops the first token. For search time analyzer we want to use a standard analyzers because we should not drop first token from the search string. Do not forget to set `preserve_position_increments` as `false` for the new field.

```text
DELETE music
PUT music
{
  "settings": {
    "analysis": {
      "filter": {
        "remove_first_word": {
          "type": "predicate_token_filter",
          "script": {
            "source": "token.position != 0"
          }
        }
      },
      "analyzer": {
        "drop_first_word": {
          "tokenizer": "standard",
          "filter": [
            "remove_first_word",
            "lowercase"
          ]
        }
      }
    }
  }, 
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "fields": {
          "from_second_word": {
            "type": "completion",
            "analyzer": "drop_first_word",
            "search_analyzer": "standard",
            "preserve_position_increments": false
          }
        }
      }
    }
  }
}

PUT music/_doc/1?refresh
{
  "suggest" : {
    "input": "Britney Spears & Elton John"
  }
}

POST music/_search?filter_path=suggest
{
  "suggest": {
    "artist": {
      "prefix": "s",
      "completion": {
        "field": "suggest.from_second_word"
      }
    }
  }
}
```

It returns:
```text
{
  "suggest": {
    "artist": [
      {
        "text": "s",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ]
  }
}
```

Great start!

What about starting from the 3rd and 4th word? Could we just create a new subfield and change the script for the `predicate_token_filter` from `token.position != 0` to `token.position != 0 && token.position != 1` and so on? IMO, could work but there should be a "better" way.

### Suggest from the second entity

In other words we need to suggest text that starts after the separator. The implementation assumes that you have 2 entities. Once again, we will leverage different index and search time analyzers. For the indexing to achieve the required functionality with token filters would be complicated. The hack would be to use `char_filter` to get rid of the first "entity".

```text
DELETE music
PUT music
{
  "settings": {
    "analysis": {
      "char_filter": {
        "remove_until_separator": {
          "type": "pattern_replace",
          "pattern": "(.*and )|(.*& )",
          "replacement": ""
        }
      },
      "analyzer": {
        "drop_first_entity": {
          "tokenizer": "standard",
          "char_filter": ["remove_until_separator"],
          "filter": [ "lowercase"]
        }
      }
    }
  }, 
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "fields": {
          "from_second_entity": {
            "type": "completion",
            "analyzer": "drop_first_entity",
            "search_analyzer": "standard",
            "preserve_position_increments": false
          }
        }
      }
    }
  }
}

PUT music/_doc/1?refresh
{
  "suggest" : {
    "input": "Britney Spears & Elton John"
  }
}

POST music/_search?filter_path=suggest
{
  "suggest": {
    "artist": {
      "prefix": "e",
      "completion": {
        "field": "suggest.from_second_entity"
      }
    }
  }
}
```

Returns:

```text
{
  "suggest": {
    "artist": [
      {
        "text": "e",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ]
  }
}
```

### Suggest from the last word

The idea is not to tokenize the string, reverse the string, tokenize the reversed string, and reverse the tokens once again. 

```text
DELETE music
PUT music
{
  "settings": {
    "analysis": {
      "analyzer": {
        "last_word": {
          "tokenizer": "keyword",
          "filter": [ 
            "lowercase", 
            "reverse",
            "word_delimiter_graph", 
            "reverse"
          ]
        }
      }
    }
  }, 
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "fields": {
          "from_last_word": {
            "type": "completion",
            "analyzer": "last_word",
            "search_analyzer": "standard",
            "preserve_position_increments": false
          }
        }
      }
    }
  }
}

PUT music/_doc/1?refresh
{
  "suggest" : {
    "input": "Britney Spears & Elton John"
  }
}

POST music/_search?filter_path=suggest
{
  "suggest": {
    "artist": {
      "prefix": "j",
      "completion": {
        "field": "suggest.from_last_word"
      }
    }
  }
}
```

returns

```json
{
  "suggest": {
    "artist": [
      {
        "text": "j",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ]
  }
}
```

Note that the suggestion for "john e" also works. Which might be a bit unexpected.

## Summary

Every separate use case is covered with a dedicated field. To support all of them at once you just need to add all the subfields into index and query all of them with multiple `suggest` clauses. How to combine those suggestions is out of scope for this post, but it should be implemented in your app.

## Bonus 1: Synonyms for suggestions

After the "successful" release of the new functionality the very next day PM (under the usual influence of some exotic and probably illegal substances) once again came up with new idea: "in search suggestions we need to support variants of artist names". His example was: "I've heard that most babies can't pronounce `britney` and she say something like [`ditney`](https://mom.com/baby-names/girl/19714/britney), so to make our product more successful among the baby searchers segment we must support this use case". 

Somewhat reasonable :)

The idea is to add synonym token filter for the artist names so that the synonym token would be on the 0 position in the token stream.

```kibana
DELETE music
PUT music
{
  "settings": {
    "analysis": {
      "filter": {
        "name_synonyms": {
          "type": "synonym",
          "synonyms": [
            "britney, ditney"
          ]
        }
      },
      "analyzer": {
        "synonyms": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "name_synonyms"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "analyzer": "synonyms"
      }
    }
  }
}

PUT music/_doc/1?refresh
{
  "suggest" : {
    "input": "Britney Spears & Elton John"
  }
}

POST music/_search?filter_path=suggest
{
  "suggest": {
    "artist": {
      "prefix": "d",
      "completion": {
        "field": "suggest"
      }
    }
  }
}
```

Returns:

```text
{
  "suggest": {
    "artist": [
      {
        "text": "d",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ]
  }
}
```

It works, amazing! Baby searchers are happy.

## Bonus 2: shingles for suggestions

After some sleeping on the mind-bending development experience a colleague asked: "can't we just use one field for all the requirements?". His reasoning was that we index the same string multiple times and our clusters doesn't have infinite capacity. Also, we've seen what and how PM is thinking, and it is only a matter of time when we will have to support suggestions from everywhere in the string". We've already seen that using synonyms it is doable.

His idea was to analyze text in such a way that it would produce multiple tokens with the position=0 for all the potential suggestion "entry points". To achieve it we could [shingle](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-shingle-tokenfilter.html) the string, take only shingles that has position=0 and take the last word from every shingle. Let's try.

```kibana
DELETE music
PUT music
{
  "settings": {
    "index.max_shingle_diff": 10,
    "analysis": {
      "filter": {
        "after_last_space": {
          "type": "pattern_replace",
          "pattern": "(.* )",
          "replacement": ""
        },
        "preserve_only_first": {
          "type": "predicate_token_filter",
          "script": {
            "source": "token.position == 0"
          }
        },
        "big_shingling": {
          "type": "shingle",
          "min_shingle_size": 2,
          "max_shingle_size": 10,
          "output_unigrams": true
        }
      },
      "analyzer": {
        "dark_magic": {
          "tokenizer": "standard",
          "filter": [
            "lowercase",
            "big_shingling",
            "preserve_only_first",
            "after_last_space"
          ]
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "suggest": {
        "type": "completion",
        "analyzer": "dark_magic",
        "search_analyzer": "standard"
      }
    }
  }
}

PUT music/_doc/1?refresh
{
  "suggest" : {
    "input": "Britney Spears & Elton John"
  }
}
```

Let's test how this analyzer works:

```kibana
POST music/_analyze
{
  "explain": false, 
  "text": ["Britney Spears & Elton John"],
  "analyzer": "dark_magic"
}
```

Returns 

```json
{
  "tokens": [
    {
      "token": "britney",
      "start_offset": 0,
      "end_offset": 7,
      "type": "<ALPHANUM>",
      "position": 0
    },
    {
      "token": "spears",
      "start_offset": 0,
      "end_offset": 14,
      "type": "shingle",
      "position": 0,
      "positionLength": 2
    },
    {
      "token": "elton",
      "start_offset": 0,
      "end_offset": 22,
      "type": "shingle",
      "position": 0,
      "positionLength": 3
    },
    {
      "token": "john",
      "start_offset": 0,
      "end_offset": 27,
      "type": "shingle",
      "position": 0,
      "positionLength": 4
    }
  ]
}
```

Positions of all the tokens are 0. Good start.

Let's test the suggestions with all the potential queries in the same request:

```text

POST music/_search?filter_path=suggest
{
  "suggest": {
    "britney": {
      "prefix": "b",
      "completion": {
        "field": "suggest"
      }
    },
    "spears": {
      "prefix": "s",
      "completion": {
        "field": "suggest"
      }
    },
    "elton": {
      "prefix": "e",
      "completion": {
        "field": "suggest"
      }
    },
    "john": {
      "prefix": "j",
      "completion": {
        "field": "suggest"
      }
    }
  }
}
```

It returns:

```json
{
  "suggest": {
    "britney": [
      {
        "text": "b",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ],
    "elton": [
      {
        "text": "e",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ],
    "john": [
      {
        "text": "j",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ],
    "spears": [
      {
        "text": "s",
        "offset": 0,
        "length": 1,
        "options": [
          {
            "text": "Britney Spears & Elton John",
            "_index": "music",
            "_id": "1",
            "_score": 1,
            "_source": {
              "suggest": {
                "input": "Britney Spears & Elton John"
              }
            }
          }
        ]
      }
    ]
  }
}
```

Wow! Note, that a query (that spans several tokens) e.g. `britney sp` doesn't match anything. Fixable, but let's leave the fix out of scope for now.

## Fin

Thank you and congratulations: You got to the very end of the blog post. Tell me about your craziest adventures with the search suggestions in the comments below?  

## Footnotes

[^1]: All the characters in this story are completely fictional.
[^lucene]: Have a look at the impressive engineering of [Lucene](https://github.com/apache/lucene/blob/main/lucene/suggest/src/java/org/apache/lucene/search/suggest/analyzing/AnalyzingSuggester.java).
