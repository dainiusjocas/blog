
---
aliases: [gitlab-ci-clojure-dependencies]
projects: [clojure]
title: Using Gitlab CI Cache for Clojure Dependencies
authors: [Dainius Jocas]
date: '2019-11-11'
tags: [clojure, devops]
categories:
  - clojure
  - gitlab
  - ci
  - devops
summary: A guide on how to use Gitlab CI Cache for Clojure Dependencies
image:
  caption: "[Photo by Pankaj Patel on Unsplash](https://cdn-images-1.medium.com/max/1200/0*iCqeXpczJLh7PD8z)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

I want to share my hard-won lessons on how to setup the Gitlab CI for Clojure projects based on tools.deps. I think that the Gitlab CI is a wonderful tool for CI workloads. But when you're going a bit sideways from the documented ways of doing things you have to do a bit of discovery for yourself.

## Gitlab CI Cache Setup

Usually I want to cache dependencies between all build and all branches. To achieve this I hard-code the cache key at the root of the `.gitlab-ci.yml` file e.g.:

```yaml
cache:
  key: one-key-to-rule-them-all
```

When it comes to caching Clojure dependencies we have to be aware that there different types of dependencies. Two most common ones are: Maven and gitlibs.

The Gitlab CI cache works **only** with directories **inside the project directory**. While local repositories (i.e. cache) for Clojure dependencies **by default** are stored **outside the project directory** (`~/.m2` and `~/.gitlibs`). Therefore, we have to provide parameters for our build tool to change the default directories for storing the dependencies.

To specify Maven local repository we can provide `:mvn/local-repo` parameter e.g.:

```yaml
clojure -Sdeps '{:mvn/local-repo "./.m2/repository"}' -A:test
```

Having configured local maven repository in our `gitlab-ci.yml` we can specify:

```yaml
cache:
  key: one-key-to-rule-them-all
  paths:
    - ./.m2/repository
```

When it comes to gitlibs there is no public API for changing the default directory in `tools.deps`. But the underlying `tools.gitlibs` uses an environment variable to set where to store the [gitlibs conveniently named **GITLIBS**](https://github.com/clojure/tools.gitlibs/blob/b7acb151b97952409103094794f5fc6f4d7d3840/src/main/clojure/clojure/tools/gitlibs.clj#L23). E.g.

```bash
$ (export GITLIBS=".gitlibs/" && clojure -A:test)
```

Of course, we should not forget to configure the cache:

```yaml
cache:
  key: one-key-to-rule-them-all
  paths:
    - ./.gitlibs
```

To use caching for both types of dependencies:

```bash
(export GITLIBS=".gitlibs/" && clojure -Sdeps '{:mvn/local-repo "./.m2/repository"}' -A:test)
```

And setup the cache:

```yaml
cache:
  key: one-key-to-rule-them-all
  paths:
    - ./.m2/repository
    - ./.gitlibs
```

If you want to disable cache for a particular job (e.g. you're linting with [clj-kondo](https://github.com/borkdude/clj-kondo), which is delivered as a [GraalVM](https://www.graalvm.org/) compiled [native image](https://www.graalvm.org/docs/reference-manual/native-image/)), just give an empty map for a job's cache setup, e.g.:

```yaml
lint:
  stage: test
  image: borkdude/clj-kondo
  cache: {}
  when: always
  script:
    - clj-kondo --lint src test
```

I've used the Gitlab CI cache while working on a streaming-text search library [Beagle](https://github.com/tokenmill/beagle). A full .gitlab-ci.yml file example of the setup can be found [here](https://github.com/tokenmill/beagle/blob/master/.gitlab-ci.yml).

Hope this helps!
