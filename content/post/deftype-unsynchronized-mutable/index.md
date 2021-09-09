---
aliases: [deftype-unsynchronized-mutable]
projects: [clojure]
title: Clojure deftype `unsynchronized-mutable`
authors: [Dainius Jocas]
date: '2021-09-09'
tags: [clojure, java-interop]
categories:
  - clojure
  - java-interop
summary: A use-case for the `unsynchronized-mutable`
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

In this post let's take a closer look at this Clojure code snippet and learn what the `^:unsynchronized-mutable` is doing in it.

```clojure
(ns user
  (:import (net.thisptr.jackson.jq JsonQuery Scope Output)
           (com.fasterxml.jackson.databind JsonNode)))

(definterface IGetter
  (^com.fasterxml.jackson.databind.JsonNode getValue []))

(deftype OutputContainer [^:unsynchronized-mutable ^JsonNode container]
  Output
  (emit [_ output-json-node] (set! container output-json-node))
  IGetter
  (getValue [_] container))

(defn query-json-node [^JsonNode data ^JsonQuery query]
  (let [output-container (OutputContainer. nil)]
    (.apply query (Scope/newEmptyScope) data output-container)
    (.getValue output-container)))
```
Note that this code snippet is a bit trimmed down for brevity. The full code can be found [here](https://github.com/dainiusjocas/clj-jq/blob/main/src/jq/core.clj).

## Introduction

I was working on [`clj-jq`](https://github.com/dainiusjocas/clj-jq): a Clojure library which is intended to be just a very thin wrapper for the [`jackson-jq`](https://github.com/eiiches/jackson-jq) Java library.
The goal of `clj-jq` library is to embed a `jq` processor directly into the application. 
The exact use cases are for future blog posts.

## Wrapper 

The `jackson-jq` provides a nice example [class](https://github.com/eiiches/jackson-jq/blob/develop/1.x/jackson-jq/src/test/java/examples/Usage.java) that helped me to start with the wrapper.
A straightforward approach to create a Clojure library out of a similar Java class is to "translate" the Java code to Clojure using the [Java interop](https://clojure.org/reference/java_interop) more or less verbatim, parameterize the hardcoded parts, and you'd be mostly done.

The "translation" was as expected until I had to deal with a code that implements the [`Output`](https://github.com/eiiches/jackson-jq/blob/6bf785ddb29618f53dafe2f336cd80bdf18a6b45/jackson-jq/src/main/java/net/thisptr/jackson/jq/Output.java) interface:
```java
final List<JsonNode> out = new ArrayList<>();
q.apply(childScope, in, out::add);
```
In the example an `ArrayList` and some Java magic was used (we'll talk about what and why a the [appendix](#appendix)).
I've "translated" the code, it worked, but I wasn't happy about it.

Using an `ArrayList` in such a code might be OK for a simple example but for a library something more specialized is desired because of several reasons:

- the `ArrayList` will always hold just one value;
- allocating an `ArrayList` is not super cheap;

It got me thinking about how to make something better.

## Requirements

The requirements for the `Output` implementation:

- a specialized and efficient class;
- that class has to act as a mutable[^1] container that holds one value;

Efficiency is desired because the library is intended to be used in "hot spots" of the applications.

Before starting to work on `clj-jq` I expected the work would not require writing any Java code because that complicates the release of the library.
Therefore, I wanted to implement my `Output` class in pure Clojure.
Also, `gen-class` always feels a bit too much for this problem.

## Implementation

The `Output` interface defines behaviour, therefore I've chosen the [`deftype`](https://clojuredocs.org/clojure.core/deftype).
The implementation class needs mutable state to hold one value.
`deftype` has two options for mutability of fields: `volatile-mutable` and `unsynchronized-mutable`.
The docs advice that the usage of both options is discouraged and are for experts only. 
I skipped the first part of the advice because I wanted to learn a lesson.
For the second part my thinking was that if I used it "correctly" I could consider myself an expert :-)

![Expert](grumpy-expert.jpg)

What are the implications of those two `deftype` field options:

- `volatile-mutable` means that field is marked to be `volatile`. Volatile in Java means that the variable is stored in the main memory and **not** in the CPU cache. This guarantees multi-threaded applications to read the latest values of a variable. Note that writes are not atomic. 
Therefore, the `volatile-mutable` field is OK to be used when the field is going to be written to by one thread and read from multiple threads.
- The field marked with `unsynchronized-mutable` option is backed by a regular Java mutable field.
In contrast to `volatile-mutable`, the value might be stored in the CPU cache.
Therefore, `unsynchronized-mutable` field is OK when it is meant to be read and written only by one thread[^2].

Since the scope in which my container class is going to be used is small and only single threaded,
I can choose an option that promises a better performance: `unsynchronized-mutable`[^3].

The resulting `Output` type definition:

## Go back to the code

The most interesting bit is the `deftype`:

```clojure
(deftype OutputContainer [^:unsynchronized-mutable ^JsonNode container]
  Output
  (emit [_ output-json-node] (set! container output-json-node))
  IGetter
  (getValue [_] container))
```

The class generated by `deftype` will be named `OutputContainer`.
The class has one mutable field: `container` that is of `JsonNode` type. 
By declaring method mutable deftype generates `set!` function to mutate the mutable field.

`OutputContainer` is in the scope of one function:

```clojure
(defn query-json-node [^JsonNode data ^JsonQuery query]
  (let [output-container (OutputContainer. nil)]
    (.apply query (Scope/newEmptyScope) data output-container)
    (.getValue output-container)))
```

Inside we create an instance of the `OutputContainer`.
Then pass it to `apply` method of the `JsonQuery`.
Somewhere in the `apply` the `Output::emit` will be called[^5].
Finally, we take out the value from the `output-container` instance.

## Summary

`unsynchronized-mutable` is rarely used in the wild: only [776](https://github.com/search?l=Clojure&q=unsynchronized-mutable&type=Code) out of [13,362](https://github.com/search?l=Clojure&q=deftype&type=Code) uses of `deftype` according to GitHub code search as of 2021-09-09.
But when we do understand the implications, and we need exactly what the option provides, we can go ahead and use it. Don't forget to test the code.

## <a id="appendix" />Appendix

Once again, let's look at this piece of code:

```java
final List<JsonNode> out = new ArrayList<>();
q.apply(childScope, in, out::add);
```

Here method reference to `add` of a `List<JsonNode>` object  is used as an argument to a method that expects `Output` type.
How does it work?

### A bit of Java theory

Any interface in Java with a SAM (Single Abstract Method) is a [**functional interface**](https://www.baeldung.com/java-8-functional-interfaces).
And for an argument of a functional interface type one can provide either:
- a class object that implements the interface,
- lambda expression.

A [method reference](https://docs.oracle.com/javase/tutorial/java/javaOO/methodreferences.html) can replace a lambda expression when passed and returned value types match. 
You can use whichever is [easier to read](https://stackoverflow.com/a/24493905/1728133).

### Why does it work anyway?

The [`Output`](https://github.com/eiiches/jackson-jq/blob/6bf785ddb29618f53dafe2f336cd80bdf18a6b45/jackson-jq/src/main/java/net/thisptr/jackson/jq/Output.java) is an interface that provides just one method `emit` (default methods doesn't count).
Therefore, it is a functional interface (however, not annotated as such).

We can provide a lambda expression where a functional interface type is expected.
Also, in the same place we can provide a method reference whose types match the expected input-output types.

What are the expected types for `Output::emit`? 
`JsonNode` for input and `void` for output.

When a variable is of type `List<JsonNode>` then the `add` method accepts `JsonNode` type and returns `void`.

We can see that types match, therefore method reference to `out::add` can be used when  `Output` is required. Voila!

## Footnotes

[^1]: Mutable and not-shared might be OK.
[^2]: Consult the [Java Memory Model](https://docs.oracle.com/javase/specs/jls/se8/html/jls-17.html#jls-17.4) docs to learn more.
[^3]: A proper discussion on [`deftype` mutability](https://stackoverflow.com/questions/21127636/what-are-the-semantic-implications-of-volatile-mutable-versus-unsynchronized-m) and [here](https://stackoverflow.com/questions/3132931/mutable-fields-in-clojure-deftype)
[^4]: Image credits to https://www.pinterest.com/pin/703476404269452082/ 
[^5]: For example [here](https://github.com/eiiches/jackson-jq/blob/6bf785ddb29618f53dafe2f336cd80bdf18a6b45/jackson-jq/src/main/java/net/thisptr/jackson/jq/internal/tree/ArrayConstruction.java#L29)
