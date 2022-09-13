---
aliases: [clojure-iteration-laziness]
projects: [clojure]
title: Interesting behaviour of `iteration` and laziness
authors: [Dainius Jocas]
date: '2022-05-18'
tags: [clojure, laziness]
categories:
  - Clojure
summary: Interesting behaviour of `iteration` and laziness
image:
  caption: "[Photo by borkdude](https://github.com/borkdude/babashka)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

TL;DR

Clojure `iteration` step function when consumed lazily is called two times initially.

## Consume paginated API in Clojure

Clojure 1.11 introduces `iteration` function for iterated API calls.

[iteration](https://clojure.atlassian.net/browse/CLJ-2555)

I've read an interesting blob [post](https://www.juxt.pro/blog/new-clojure-iteration).

I wanted to refactor my little libary [`lazy-elasticsearch-scroll`](https://github.com/dainiusjocas/lazy-elasticsearch-scroll).




