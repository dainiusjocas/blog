---
aliases: [lucene-text-analysis]
projects: [search]
title: lmgrep - Lucene Text Analysis
authors: [Dainius Jocas]
date: '2021-04-23'
tags: [Lucene, tokenization]
categories:
  - Lucene
summary: "`lmgrep` exposes easy to use interface to work with Lucene text analysis"
image:
  caption: "Lucene"
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

`lmgrep` provides an easy way to play with various text analysis options.
Just download the `lmgrep` binary, run it with `--only-analyze`, and observe the list of tokens.

```shell
echo "Dogs and CATS" | lmgrep \
  --only-analyze \
  --analysis='
    {
      "tokenizer": {"name": "standard"},
      "token-filters": [
        {"name": "lowercase"},
        {"name": "englishminimalstem"
      }
    ]
  }'
=>
["dog","and","cat"]
```

## Text Analysis

The Elasticsearch documentation describes text analysis as:

> the process of converting unstructured text into a structured format that’s optimized for search[^1].

Therefore, to learn how the full-text search works it is important to understand how the text is analyzed.
Let's focus on how text analysis is done in Lucene which is the library that powers Elasticsearch and Solr.

# Lucene

Text analysis in the Lucene land is defined by 3 types of components:

- list of character filters (changes to the text before tokenization, e.g. HTML stripping, character replacement, etc.),
- **one** tokenizer (splits text into tokens),
- list of token filters (normalizes the tokens).

The particular combination of text analysis components makes an `Analyzer`. You can think that an analyzer is a recipe 

## `lmgrep` and Text Analysis

`lmgrep` is a search tool that is based on the Lucene Monitor library. 
To do the full-text search it needs to do the same thing that likes of Elasticsearch are doing: to analyze text.
`lmgrep` packs many Lucene text [analysis components](https://github.com/dainiusjocas/lucene-grep/blob/v2021.04.23/docs/analysis-components.md). 
Also, `lmgrep` provides a list of [predefined analyzers](https://github.com/dainiusjocas/lucene-grep/blob/v2021.04.23/docs/predefined-analyzers.md).
No surprises here, the same battle tested components that gets the job done.

However, the clever bit that `lmgrep` provides is the way to specify an analyzer using data in JSON, e.g.:

```shell
echo "<p>foo bars baz</p>" | \
  lmgrep \
  --only-analyze \
  --analysis='
  {
    "char-filters": [
      {"name": "htmlStrip"},
      {
        "name": "patternReplace",
         "args": {
           "pattern": "foo",
           "replacement": "bar"
        }
      }
    ],
    "tokenizer": {"name": "standard"},
    "token-filters": [
      {"name": "englishMinimalStem"},
      {"name": "uppercase"}
    ]
  }
  '
```

Again, no surprises here, just read the docs of an interesting component, e.g. 
character filter [`patternReplace`](https://lucene.apache.org/core/8_3_0/analyzers-common/org/apache/lucene/analysis/pattern/PatternReplaceCharFilterFactory.html),
add to the `--analysis`, and apply it on your text.

Conceptually it is very similar to what Elasticsearch or Solr are providing: `analysis` JSON in Elasticsearch index config and XML in Solr.

All the analysis component has this structure: `{"name": "THE_NAME", "args": {"ARG_NAME": "ARG_VALUE"}}`.

NOTE: some components, e.g. `stop` token filter, expect a file as an argument.
To support such components `lmgrep` brutally patched Lucene to support loading data from arbitrary files while preserving the predefined analyzers with their stop-words files.

## `--only-analyze`

I like the Elasticsearch's [Analyze API](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-analyze.html).
It allows me to look at the raw tokens that are either stored in the index or produced from the search query.

To make the life of `lmgrep` users easier I wanted to expose similar functionality.

`--only-analyze` flag does just that: the output is a list of tokens that is produced by applying an analyzer on the input text, e.g.:

```shell
echo "the quick brown fox" | lmgrep --only-analyze
=>
["the","quick","brown","fox"]
```
NOTE: the output lines are valid JSON.

However, I'm constantly frustrated with Elasticsearch's Analysis API because I can't specify details of char filters, tokenizer, and token filter directly in the call to the Analysis API: I have to create an index with an analyzer
and only then call Analyze API. `lmgrep` avoids this pain point by allowing to write down everything that an analyzer needs in one JSON object.

Performance wise, the `lmgrep` on my laptop is able to analyze ~1GB of text in a couple of seconds. Not bad at all. The design is to use one thread that just reads the input (either STDIN or a file), another thread that writes to the output, and the remaining CPU can used to analyze text.

NOTE: the `explain` flag is coming to `lmgrep`.

## Implementation

All this analyzer construction wizardry is possible because of `AbstractAnalysisFactory` class and features provided by its subclasses. `CustomAnalyzer` builder exposes methods that expects a `Class` as an argument. The trick is that, e.g. the class `TokenFilterFactory` provides a method `availableTokenFilters` that returns a set of `names` of token filters and with that `name` you can get a `Class` that ca be supplied to the builder of the `CustomAnalyzer`.

The discovery of available classes is based on classpath analysis, e.g. fetching all classes whose name matches a pattern like `.*FilterFactory` and are subclasses of a `TokenFilterFactory`. However, for the reasons that were beyond my understanding, when I wanted to create my own TokenFilterFactory implementations these were not discovered by Lucene. 

Yeah, great, but `lmgrep` is compiled with the GraalVM native image that assumes closed-world and dynamism of JVM is out the window and then how does this class discovery works? Native images are static at run-time, but it can be worked around by providing the reflection configuration for interesting classes, and those interesting classes can be discovered at compile-time where the dynamism works as expected.  

To instruct `native-image` to touch the classes in Clojure you can specify the class under the regular `def` because to the `native-image` these look like constants and are compiled in.

## References

## Footnotes

[^1]: https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html
