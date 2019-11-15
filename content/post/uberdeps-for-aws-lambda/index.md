---
aliases: [uberdeps-for-aws-lambda]
projects: [clojure]
title: Using Uberdeps to Build AWS Lambda Uberjar
authors: [Dainius Jocas]
date: '2019-11-15'
tags: [clojure, devops, aws, lambda]
categories:
  - clojure
  - aws
  - lambda
  - devops
summary: A guide on how to build an Uberjar for AWS Lambda with `tonsky/uberdeps`
image:
  caption: "[Photo by nexmo.com](https://www.nexmo.com/blog/2016/05/31/building-sms-google-sheets-application-aws-lambda-dr)"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

I was writing a Clojure application and the plan was to deploy it as a AWS Lambda. The question I'm going to answer in this blog post is: how to build an uberjar for AWS Lambda with [Uberdeps](https://github.com/tonsky/uberdeps)?

## TL;DR

Add an alias to the `deps.edn` for uberjar building:
```
{:aliases {:uberjar
           {:extra-deps {uberdeps {:mvn/version "0.1.6"}}
            :main-opts  ["-m" "uberdeps.uberjar"]}}}
```

Create an executable file `compile.clj` in the project root folder:
```bash
touch compile.clj
chmod +x compile.clj
```

Put this code in the `compile.clj` file:

<script src="https://gist.github.com/dainiusjocas/e9b154d7a1cbdca8558cd7c5d730d5d0.js"></script>

Run:
```bash
(rm -rf classes && \
	mkdir classes && \
	./compile.clj && \
	clojure -A:uberjar --target target/UBERJAR_NAME.jar)
```

I'd advise put that last script into a `Makefile` ;)

---
## Introduction

To deploy your Clojure code to AWS Lambda you need to package it as an uberjar. If your project is managed with `deps.edn`, basically you're on your own to find a suitable library to package your code.

For some time to build uberjars for `deps.edn` projects I was using [Cambada](https://github.com/luchiniatwork/cambada). It did the job but I was not entirely happy with the library for a couple of reasons:

- the library seems to be no longer maintained;
- it has various [bugs](https://github.com/luchiniatwork/cambada/issues) with transitive Git dependencies. I've found out that these bugs are fixed in a [fork](https://github.com/xfthhxk/cambada) of the Cambada and I used it as a git dependency.

Because building an uberjar for `deps.edn` boils down to just finding a library there is always temptation to try something new.

## Enter Uberdeps

For my toy project I wanted to try out [Uberdeps](https://github.com/tonsky/uberdeps). The introduction [blog post](https://tonsky.me/blog/uberdeps/) got me interested and I really liked the main idea:

> Takes deps.edn and packs an uberjar out of it.

Sounds like exactly what I need.

## Trouble

I've written my application, added all the things needed to deploy it as an AWS Lambda, build an uberjar with Uberdeps, deployed the app with the AWS CloudFormation, but when I've invoked the Lambda I've received an error:

```
{
   "message" : "Internal server error"
}
```

After searching through the AWS CloudWatch logs I've found:
```
Class not found: my.Lambda: java.lang.ClassNotFoundException
java.lang.ClassNotFoundException: my.Lambda
	at java.net.URLClassLoader.findClass(URLClassLoader.java:382)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:424)
	at java.lang.ClassLoader.loadClass(ClassLoader.java:357)
	at java.lang.Class.forName0(Native Method)
	at java.lang.Class.forName(Class.java:348)
```
The `my.Lambda` class was not found.

After taking a look at the contents of the uberjar I've noticed that the `my.Lambda` class is indeed not inside the Uberjar. Ah, it seems that AOT (Ahead-of-Time) is not done out of the box. After searching and not finding a flag or some parameter that I need to pass to force the AOT compilation in the Uberdeps README, I've discovered an already closed [pull request](https://github.com/tonsky/uberdeps/pull/11): the AOT compilation functionality is not implemented.

I was in trouble.

## Solution

The solution was to manually perform AOT compilation of the relevant namespaces right before building an uberjar and then instruct Uberdeps to put the resulting class files into the uberjar.

To do AOT compilation I've written a Clojure script `compile.clj`:
<script src="https://gist.github.com/dainiusjocas/e9b154d7a1cbdca8558cd7c5d730d5d0.js"></script>

Inspiration on how to write the script was taken from [here](https://www.reddit.com/r/Clojure/comments/8ltsrs/standalone_script_with_clj_including_dependencies/) and [here](https://github.com/tonsky/datascript/blob/master/release.clj).

To instruct Uberdeps to put class files to the uberjar I've added `classes` directory to the `:paths` vector in `deps.edn`.

Just for the convenience, in the Makefile I've put commands for AOT compilation right before the command to build an uberjar:
```
uberjar:
	rm -rf classes
	mkdir classes
	./compile.clj
	clojure -A:uberjar --target target/my-jar-name.jar
```

And that is it! I have an uberjar with `my.Lambda` class and the AWS Lambda runtime is happy.

## Discussion

The solution is not bullet proof because:

- it assumes that the main `deps.end` file is called `deps.edn`;
- compiled classes are put in the `classes` directory;
- the alias for which namespaces should be AOT compiled is the default alias.

I hope that when a more generic solution will be needed either the Uberdeps will have an option for AOT compilatoin or I'll be clever enough to deal with the situation and write a follow up blog post with the workaround.
