---
aliases: [intro-to-lucene-grep]
projects: [search]
title: Introducing lmgrep - Lucene Monitor based grep-like CLI utility
authors: [Dainius Jocas]
date: '2021-01-16'
tags: [Lucene, GraalVM, grep]
categories:
  - Lucene
  - GraalVM
summary: Introduction into `lmgrep`
image:
  caption: "Grep with Lucene"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

Have you ever wished that `grep` supported stemming so that you dont have to write wildcard regexes all the time? Me too and one nice day I've tried to scratch that itch by exposing the Lucene query syntax as a CLI utility. `lmgep` is the result of my effort, [give it a try](https://github.com/dainiusjocas/lucene-grep) and let me know how it goes.

## Full-text Search vs. grep

I'm perfectly aware that comparing Lucene and `grep` is like comparing apples to oranges. However, I think that `lmgrep` is best compared with the very tool that inspired it, that is `grep`.

What distinguishes `lmgrep` search from `grep`:
- Lucene query syntax is better suited for full-text search;
- Boolean operators allow to construct complex, well-designed queries;
- Text analysis can be customized to the language of the documents;
- Flexible text analysis pipeline that includes, lowercasing, ASCII-filding, stemming, etc;
- Supports regexp within other query parts;
- Is not line-oriented, e.g. phrases can span multiple lines

Several limitation of `lmgrep` compared to `grep`:
- `grep` is faster when it comes to raw performance;
- Memory usage is a higher than `grep`;
- Not all options of `grep` are supported;

## Why to use Lucene?

Lucene is a Java Library that provides indexing and search features. Lucene has been more than 20 years in development and it is the library that powers many search applications. Also, there are many developers that are already familiar with the Lucene query syntax and know how to leverage it to solve complicated information retrieval problems.  

However powerful Lucene is, it is not well-suited for usage as a the CLI applications. The main problem is the startup time of JVM. To reduce the startup time I've compiled `lmgrep` with the `native-image` tool privided by [GraalVM](https://www.graalvm.org/). In this way the startup time is 0.01s for Linux, MacOS, and yes, even Windows.

## How does it work?

`lmgrep` by default expects two parameters: a search query and a [GLOB pattern](https://docs.oracle.com/javase/8/docs/api/java/nio/file/FileSystem.html#getPathMatcher-java.lang.String-) (similar to regex) to find files to execute `lmgrep` on. I assume that the dear reader doesn't want to be tortured by reading the explanation on how the file names are being matched with GLOB, so I'll skip it. Instead, I'll focus on explaining how the searh works within a file.

`lmgrep` creates a [Lucene Monitor (Monitor)](https://lucene.apache.org/core/8_7_0/monitor/org/apache/lucene/monitor/Monitor.html) object by providing the search query. Then text file is split into lines[^1]. Each line of text is passed to the Monitor for searching. The Monitor then creates an in-memory Lucene index with a single document created out of the line of text. Then the Monitor runs the search query on that in-memory index in the good ol' Lucene way[^2]. `lmgrep` takes the hits, formats them, and sends results to `STDOUT`. That is the how `lmgrep` does the full-text search.

The approach is very similar to the [Percolator](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-percolate-query.html) in Elasticsearch. `lmgrep` just limits the number of stored search queries to one and treats the text as a single document. A cool thing compared with the Percolator is that `lmgrep` provides exact offsets of the matched terms while Elasticsearch does not expose offsets just highlights.

The procedure seems to be rather inneficient, however the query parsing for **all** the lines (and files) is done only once. Also, the searching itself is very efficient thanks to Lucene in general and when queries are complicated thanks to the [Presearcher](https://lucene.apache.org/core/8_2_0/monitor/org/apache/lucene/monitor/Presearcher.html) of the Lucene Monitor in particular. Presearcher extracts terms from the serch query and if none of these terms are in the index then full query is not even executed. Of course, many optimizations can be implemented for `lmgrep` such as batching of the documents, but the performance is limited by the Lucene Monitor. 

What about the text analysis pipeline? By default it uses the `StandardTokenizer` to tokenize text. Then the tokens are passed though several token filters in the following order: `LowerCaseFilter`, `ASCIIFoldingFilter`, and `SnowballFilter` which is given the `EnglishStemmer`. The same analysis pipeline is used for both the indexing and querying. All the components of the analysis pipeline are configurable via flags, see the [README](https://github.com/dainiusjocas/lucene-grep/blob/main/README.md#supported-tokenizers). However, the order of the token filters, as of now, is not configurable. Moreover, there are various filters that are not exposed (e.g. `StopwordsFilter`, or [WordDelimiterGraphFilter](https://lucene.apache.org/core/7_4_0/analyzers-common/org/apache/lucene/analysis/miscellaneous/WordDelimiterGraphFilter.html), etc.). Supporting a more flexible analysis pipeline configuration is left out for future releases. The more usage the tool has the faster new features will be implemented ;)

## Prehistory of the `lmgrep`

Almost every NLP project that I've worked on had the component called dictionary annotator. Also, the wast majority of the projects used Elasticsearch in one way or another. The more familiar I've got with Elasticsearch I've got the more of my NLP workload shifted towards implementing it in Elasticsearch. One day I've discovered a tool caled [Luwak](https://github.com/flaxsearch/luwak) (cool name isn't it?) and read some [more about it](https://web.archive.org/web/20201124175132/https://www.flax.co.uk/blog/2016/03/08/helping-bloomberg-build-real-time-news-search-engine/). It kind of opened my eyes: the dictionary annotator can be implemented using Elasticsearch and the dictionary entries can be seen as Elasticsearch queries. Thankfully, Elasticsearch has Percolator that hides all the complexity of managing temporary indices, etc.

Then I was given was an NLP project where one of the requirements was to implement data analysis using AWS serverless stuff: Lambda for processing and Dynamo DB for storage. Of course, one of the NLP componens needed was the dictionary annotator. Since Elasticsearch was not available because it is not serverless but I still wanted to continue working with dictionary entries search queries, I've decided to leverage the Luwak library. From the experiences of that exact project the [Beagle](https://web.archive.org/web/20201124175132/https://www.flax.co.uk/blog/2016/03/08/helping-bloomberg-build-real-time-news-search-engine/) library was born.

When thinking about how to imlement `lmgrep` I knew that I want it to be based on Lucene because of the full-text search features, but to be usable as a CLI tool it has to be compiled with the `native-image` tool of the GraalVM. I've tried but the `native-image` doesn't support [Method Handles](https://web.archive.org/web/20201124175132/https://www.flax.co.uk/blog/2016/03/08/helping-bloomberg-build-real-time-news-search-engine/). So some more hacking needed to be done. I was lucky when one day I've discovered a [toy project](https://web.archive.org/web/2/https://www.morling.dev/blog/how-i-built-a-serverless-search-for-my-blog/) where the blog search was backed by AWS lambda that ran the Lucene that was compiled by the `native-image` tool. I've cloned the code, `mvnw install`, included artefacts to the dependencies list, and `lmgrep` compiled with the `native-image` tool successfully.

Then the most complicated part was to prepare binaries for different operating systems. Plenty of CPU and RAM, VirtualBox with Windows and MacOS virtual machines, and [here we go](https://github.com/dainiusjocas/lucene-grep/releases/tag/v2021.01.15). 

Did I say how much I enjoyed trying to get stuff done with Windows? None at all. How come that multiple different(!) command promts are needed to get GraalVM to compile executable? Now I know that it would better just to suffer the pain and to setup the Github Actions pipelines to compile the binaries.

## What is missing?

- [ ] The analysis pipeline is not very flexible
- [ ] Leverage the multicore CPUs by executing the search in parallel
- [ ] Batch documents for matching

## What other ways to use the thing?

- [ ] Piping the data and getting stuff back.
- [ ] Give an alias and have a code search (Java Example)
- [ ] Why not to expose sci script as TokenFilter?
- [ ] Why not ngrams token filter then it is resilient to the typing errors?
- [ ] Static site search, like AWS Lambda that has lmgrep and goes through all files on demand

## References

- https://web.archive.org/web/20210116173133/https://swtch.com/~rsc/regexp/regexp4.html
- https://web.archive.org/web/20161018234331/http://www.techrepublic.com/article/graduating-from-grep-powerful-text-file-searching-with-isearch/

## Footnotes

[^1]: there is no necessity to split file into lines, it is just to mimick how `grep` operates.
[^2]: of course, the description is over-simplified, but it is accurate enough to get the general idea.
