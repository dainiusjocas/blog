---
aliases: [babashka-aws-lambda]
projects: [babashka]
title: Deploy babashka script to AWS Lambda
authors: [Dainius Jocas]
date: '2020-03-21'
tags: [AWS Lambda, babashka, clojure, GraalVM]
categories:
  - AWS Lambda
  - babashka
  - Clojure
  - GraalVM
summary: Adventures with babashka and AWS Lambda.
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

I've managed to package a simple [babashka](https://github.com/borkdude/babashka) script to an AWS Lambda Custom Runtime. [Here](https://github.com/dainiusjocas/babashka-lambda) is the code, try for yourself.

## Motivation

Wouldn't it be great to deploy little Clojure code snippets to Custom Lambda Runtime? The main benefits would be:
- you would not suffer from java cold-start problems;
- you wouldn't need to compile your project with GraalVM `native-image` tool which is time consuming and for anything more advanced is not likely to work anyway;
- babashka supports scripting with a subset of Clojure, which might do the work for you.

## The plan

I know what it takes to deploy to Lambda Custom Runtime. Last year I've created a Clojure project template for deploying [GraalVM compiled AWS Lambda Custom Runtime](https://github.com/tokenmill/clojure-graalvm-aws-lambda-template). And babashka is just another self contained binary. It should be too hard to bring two things together and get it working? Challenge accepted.

## Packaging

I like to build software inside Docker containers. In this experiment, for the first attempt I've used this Dockerfile:
```
FROM borkdude/babashka:latest as BABASHKA

FROM clojure:tools-deps-alpine as BUILDER
RUN apk add --no-cache zip
WORKDIR /var/task

COPY --from=BABASHKA /usr/local/bin/bb bb

ENV GITLIBS=".gitlibs/"
COPY lambda/bootstrap bootstrap

COPY deps.edn deps.edn

RUN clojure -Sdeps '{:mvn/local-repo "./.m2/repository"}' -Spath > cp
COPY src/ src/
COPY resources/ resources/

RUN zip -q -r function.zip bb cp bootstrap .gitlibs/ .m2/ src/ resources/ deps.edn
```
Here:
- copy `bb` binary from babashka Docker image,
- download the dependencies for babashka script using `clojure` (both, maven and git dependencies are supported, like is described [here](https://www.jocas.lt/blog/post/gitlab-ci-clojure-dependencies/)),
- write a classpath to the `cp` file,
- copy all source code,
- zip the required contents to the `function.zip`.

Every line of this dockerfile is packed with details but I'll leave it for the future posts.

I've packaged all dependencies for lambda into `function.zip`. The contents of the archive are:
- `bb`: babashka binary
- `bootstrap`: AWS Lambda entry point script
- `cp`: generated classpath text file
- `deps.edn`
- `.gitlibs`: directory with gitlibs
- `.m2`: directory with Maven dependencies
- `resources`:
- `src`: directory with babashka scripts

## Custom runtime discoveries

Finally, having all dependencies packaged up, I've deployed the `function.zip` to AWS Lambda. The first error message was not very [encouraging](https://gist.github.com/dainiusjocas/feafeef5653ff2c6e8c7b2d9627a831d):

```text
Util_sun_misc_Signal.ensureInitialized: CSunMiscSignal.create() failed.  errno: 38  Function not implemented
Fatal error: Util_sun_misc_Signal.ensureInitialized: CSunMiscSignal.open() failed.
JavaFrameAnchor dump:
No anchors
TopFrame info:
TotalFrameSize in CodeInfoTable 32
VMThreads info:
VMThread 0000000003042750  STATUS_IN_JAVA (safepoints disabled)  java.lang.Thread@0x264fa98
VM Thread State for current thread 0000000003042750:
0 (8 bytes): com.oracle.svm.jni.JNIThreadLocalEnvironment.jniFunctions = (bytes) 
0000000003042750: 0000000002293a88
8 (32 bytes): com.oracle.svm.core.genscavenge.ThreadLocalAllocation.regularTLAB = (bytes) 
0000000003042758: 00007f7809500000 00007f7809600000
0000000003042768: 00007f7809507160 0000000000000000
40 (8 bytes): com.oracle.svm.core.heap.NoAllocationVerifier.openVerifiers = (Object) null
48 (8 bytes): com.oracle.svm.core.jdk.IdentityHashCodeSupport.hashCodeGeneratorTL = (Object) null
56 (8 bytes): com.oracle.svm.core.snippets.SnippetRuntime.currentException = (Object) null
64 (8 bytes): com.oracle.svm.core.thread.JavaThreads.currentThread = (Object) java.lang.Thread  000000000264fa98
72 (8 bytes): com.oracle.svm.core.thread.ThreadingSupportImpl.activeTimer = (Object) null
80 (8 bytes): com.oracle.svm.jni.JNIObjectHandles.handles = (Object) com.oracle.svm.core.handles.ThreadLocalHandles  00007f7809501558
88 (8 bytes): com.oracle.svm.jni.JNIThreadLocalPendingException.pendingException = (Object) null
96 (8 bytes): com.oracle.svm.jni.JNIThreadLocalPinnedObjects.pinnedObjectsListHead = (Object) null
104 (8 bytes): com.oracle.svm.jni.JNIThreadOwnedMonitors.ownedMonitors = (Object) null
112 (8 bytes): com.oracle.svm.core.genscavenge.ThreadLocalAllocation.freeList = (Word) 0  0000000000000000
120 (8 bytes): com.oracle.svm.core.graal.snippets.StackOverflowCheckImpl.stackBoundaryTL = (Word) 1  0000000000000001
128 (8 bytes): com.oracle.svm.core.stack.JavaFrameAnchors.lastAnchor = (Word) 0  0000000000000000
136 (8 bytes): com.oracle.svm.core.thread.VMThreads.IsolateTL = (Word) 25636864  0000000001873000
144 (8 bytes): com.oracle.svm.core.thread.VMThreads.OSThreadHandleTL = (Word) 50477184  0000000003023880
152 (8 bytes): com.oracle.svm.core.thread.VMThreads.OSThreadIdTL = (Word) 50477184  0000000003023880
160 (8 bytes): com.oracle.svm.core.thread.VMThreads.nextTL = (Word) 0  0000000000000000
168 (4 bytes): com.oracle.svm.core.graal.snippets.StackOverflowCheckImpl.yellowZoneStateTL = (int) -16843010  fefefefe
172 (4 bytes): com.oracle.svm.core.snippets.ImplicitExceptions.implicitExceptionsAreFatal = (int) 0  00000000
176 (4 bytes): com.oracle.svm.core.thread.Safepoint.safepointRequested = (int) 2147473200  7fffd730
180 (4 bytes): com.oracle.svm.core.thread.ThreadingSupportImpl.currentPauseDepth = (int) 0  00000000
184 (4 bytes): com.oracle.svm.core.thread.VMThreads$StatusSupport.safepointsDisabledTL = (int) 1  00000001
188 (4 bytes): com.oracle.svm.core.thread.VMThreads$StatusSupport.statusTL = (int) 1  00000001
VMOperation dump:
No VMOperation in progress
Dump Counters:
Raw Stacktrace:
00007ffeb8e0a940: 000000000186e776 000000000207b9d0
00007ffeb8e0a950: 0000000001873000 000000000085b37c
00007ffeb8e0a960: 000000000084540a 00000000008454ca
00007ffeb8e0a970: 000000000264f128 000000000264ef58
00007ffeb8e0a980: 00007f78095018d8 0000000002650640
00007ffeb8e0a990: 000000000264f128 0000002602650c18
00007ffeb8e0a9a0: 0000000000845444 00007ffeb8e0a970
00007ffeb8e0a9b0: 0000000000000000 0000000000845f6e
00007ffeb8e0a9c0: 0000000002650e18 0000000002650c18
00007ffeb8e0a9d0: 0000000002650e18 0000000002070c60
00007ffeb8e0a9e0: 00000000021f48f8 00000000012b77e6
00007ffeb8e0a9f0: 0000000002650e18 0000000002650c18
00007ffeb8e0aa00: 0000001000000000 0000000002070c60
00007ffeb8e0aa10: 00007f7809507138 0000000000477f69
00007ffeb8e0aa20: 00007f7809503b88 00007f7809501910
00007ffeb8e0aa30: 00007f7809507138 00000000004831b4
00007ffeb8e0aa40: 0000000000000010 000000000085d16d
00007ffeb8e0aa50: 000000000000003b 00000000008b4bdb
00007ffeb8e0aa60: 000000000291e970 00007f7809504828
00007ffeb8e0aa70: 0000000100000007 0000000001079a70
00007ffeb8e0aa80: 00007f78095070b8 00007f7809507080
00007ffeb8e0aa90: 0000000001873000 000000000291e970
00007ffeb8e0aaa0: 00007f7809506f78 00007f78095070b8
00007ffeb8e0aab0: 0000000000000008 0000000000000010
00007ffeb8e0aac0: 0000000000000010 00000000008144a1
00007ffeb8e0aad0: 0000000000000007 0000000000cd7c2e
00007ffeb8e0aae0: 00007f7809504938 0000000001873000
00007ffeb8e0aaf0: 0000000002205088 00007f78095070b8
00007ffeb8e0ab00: 00007f7809507080 0000000cc0001000
00007ffeb8e0ab10: 0000000000000000 0000000000cd73eb
00007ffeb8e0ab20: 00007f7809503b58 00007f78095070b8
00007ffeb8e0ab30: 00007f7809507080 00007f78095038e0
00007ffeb8e0ab40: 00007f7807c8e388 000000000205e900
00007ffeb8e0ab50: 00007f7809501350 000000240000000c
00007ffeb8e0ab60: 000000000000000c 00007f78095038e0
00007ffeb8e0ab70: d15c483b00000000 00000000004830e5
00007ffeb8e0ab80: 0000000000000007 00007f78095038e0
00007ffeb8e0ab90: 00007f78095038e0 00000000006f2b33
00007ffeb8e0aba0: 000000000205e900 0000000002070448
00007ffeb8e0abb0: 00007f78095070b8 0000000000cd8b3d
00007ffeb8e0abc0: 00000000020864c8 0000000000cbffc1
00007ffeb8e0abd0: 0000000002070448 00007f78095070b8
00007ffeb8e0abe0: 0000000c00000000 00007f7809505ef8
00007ffeb8e0abf0: 00007f78095070d8 00007f7809504840
00007ffeb8e0ac00: 7cab467402070d98 0000000000fbfc08
00007ffeb8e0ac10: 0000000002634470 00007f7809507020
00007ffeb8e0ac20: 0000000001873000 00007f78095070d8
00007ffeb8e0ac30: 00007f7809504840 0000000000cc187e
00007ffeb8e0ac40: 0000000000000000 0000000000000000
00007ffeb8e0ac50: 00007f7807c91840 00007f7809504840
00007ffeb8e0ac60: 0000000002070d98 0000000000cc17b9
00007ffeb8e0ac70: 0000000000c848f0 00007f78095038e0
00007ffeb8e0ac80: 0000000002b33a78 0000000100cc4f83
00007ffeb8e0ac90: 0000000000483140 00000000004b5713
00007ffeb8e0aca0: 0000000002070d98 0000000000cdae9a
00007ffeb8e0acb0: 000000000209a600 00007f78095038e0
00007ffeb8e0acc0: 0000000002b33a78 000000000047c576
00007ffeb8e0acd0: 000000000209a600 000000000209a630
00007ffeb8e0ace0: 0000000002a1b8d8 0000000002a1b408
00007ffeb8e0acf0: 000000000209a600 00000000017acc23
00007ffeb8e0ad00: 0000000000000001 0000000000001000
00007ffeb8e0ad10: 0000000000000000 0000000000000000
00007ffeb8e0ad20: 0000000000000000 0000000000000000
00007ffeb8e0ad30: 0000000000000000 0000000000000000
Stacktrace Stage0:
RSP 00007ffeb8e0a940 RIP 000000000085b3f6 FrameSize 32
RSP 00007ffeb8e0a960 RIP 000000000085b37c FrameSize 16
RSP 00007ffeb8e0a970 RIP 00000000008454ca FrameSize 80
RSP 00007ffeb8e0a9c0 RIP 0000000000845f6e FrameSize 48
RSP 00007ffeb8e0a9f0 RIP 00000000012b77e6 FrameSize 48
RSP 00007ffeb8e0aa20 RIP 0000000000477f69 FrameSize 32
RSP 00007ffeb8e0aa40 RIP 00000000004831b4 FrameSize 320
RSP 00007ffeb8e0ab80 RIP 00000000004830e5 FrameSize 32
RSP 00007ffeb8e0aba0 RIP 00000000006f2b33 FrameSize 256
RSP 00007ffeb8e0aca0 RIP 00000000004b5713 FrameSize 48
RSP 00007ffeb8e0acd0 RIP 000000000047c576 FrameSize 160
RSP 00007ffeb8e0ad70 RIP 000000000047c285 FrameSize 32
RSP 00007ffeb8e0ad90 RIP 00000000006f2b33 FrameSize 256
RSP 00007ffeb8e0ae90 RIP 000000000048f162 FrameSize 32
RSP 00007ffeb8e0aeb0 RIP 00000000007fb05c FrameSize 1
Stacktrace Stage1:
RSP 00007ffeb8e0a940 RIP 000000000085b3f6  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0a960 RIP 000000000085b37c  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0a970 RIP 00000000008454ca  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0a9c0 RIP 0000000000845f6e  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0a9f0 RIP 00000000012b77e6  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0aa20 RIP 0000000000477f69  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0aa40 RIP 00000000004831b4  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0ab80 RIP 00000000004830e5  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0aba0 RIP 00000000006f2b33  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0aca0 RIP 00000000004b5713  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0acd0 RIP 000000000047c576  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0ad70 RIP 000000000047c285  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0ad90 RIP 00000000006f2b33  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0ae90 RIP 000000000048f162  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
RSP 00007ffeb8e0aeb0 RIP 00000000007fb05c  com.oracle.svm.core.code.CodeInfo@0x2618c70 name = image code
Full Stacktrace:
RSP 00007ffeb8e0a940 RIP 000000000085b3f6  [image code] com.oracle.svm.core.jdk.VMErrorSubstitutions.shutdown(VMErrorSubstitutions.java:111)
RSP 00007ffeb8e0a940 RIP 000000000085b3f6  [image code] com.oracle.svm.core.util.VMError.shouldNotReachHere(VMError.java:74)
RSP 00007ffeb8e0a960 RIP 000000000085b37c  [image code] com.oracle.svm.core.util.VMError.shouldNotReachHere(VMError.java:59)
RSP 00007ffeb8e0a970 RIP 00000000008454ca  [image code] com.oracle.svm.core.posix.Util_jdk_internal_misc_Signal.ensureInitialized(SunMiscSubstitutions.java:176)
RSP 00007ffeb8e0a9c0 RIP 0000000000845f6e  [image code] com.oracle.svm.core.posix.Util_jdk_internal_misc_Signal.numberFromName(SunMiscSubstitutions.java:223)
RSP 00007ffeb8e0a9f0 RIP 00000000012b77e6  [image code] sun.misc.Signal.findSignal(Signal.java:78)
RSP 00007ffeb8e0a9f0 RIP 00000000012b77e6  [image code] sun.misc.Signal.<init>(Signal.java:140)
RSP 00007ffeb8e0aa20 RIP 0000000000477f69  [image code] babashka.impl.pipe_signal_handler$handle_pipe_BANG_.invokeStatic(pipe_signal_handler.clj:11)
RSP 00007ffeb8e0aa40 RIP 00000000004831b4  [image code] babashka.main$main.invokeStatic(main.clj:282)
RSP 00007ffeb8e0ab80 RIP 00000000004830e5  [image code] babashka.main$main.doInvoke(main.clj:282)
RSP 00007ffeb8e0aba0 RIP 00000000006f2b33  [image code] clojure.lang.RestFn.applyTo(RestFn.java:137)
RSP 00007ffeb8e0aca0 RIP 00000000004b5713  [image code] clojure.core$apply.invokeStatic(core.clj:665)
RSP 00007ffeb8e0acd0 RIP 000000000047c576  [image code] babashka.main$_main.invokeStatic(main.clj:442)
RSP 00007ffeb8e0ad70 RIP 000000000047c285  [image code] babashka.main$_main.doInvoke(main.clj:437)
RSP 00007ffeb8e0ad90 RIP 00000000006f2b33  [image code] clojure.lang.RestFn.applyTo(RestFn.java:137)
RSP 00007ffeb8e0ae90 RIP 000000000048f162  [image code] babashka.main.main(Unknown Source)
RSP 00007ffeb8e0aeb0 RIP 00000000007fb05c  [image code] com.oracle.svm.core.JavaMainWrapper.runCore(JavaMainWrapper.java:151)
RSP 00007ffeb8e0aeb0 RIP 00000000007fb05c  [image code] com.oracle.svm.core.JavaMainWrapper.run(JavaMainWrapper.java:186)
RSP 00007ffeb8e0aeb0 RIP 00000000007fb05c  [image code] com.oracle.svm.core.code.IsolateEnterStub.JavaMainWrapper_run_5087f5482cc9a6abc971913ece43acb471d2631b(IsolateEnterStub.java:0)
[Native image heap boundaries: 
ReadOnly Primitives: 0x1873008 .. 0x206f048
ReadOnly References: 0x206ff78 .. 0x24fc9f8
Writable Primitives: 0x24fd000 .. 0x26343e0
Writable References: 0x2634470 .. 0x2ba42c0]
[Heap:
[Young generation: 
[youngSpace:
aligned: 0/0 unaligned: 0/0]]
[Old generation: 
[fromSpace:
aligned: 0/0 unaligned: 0/0]
[toSpace:
aligned: 0/0 unaligned: 0/0]
]
[Unused:
aligned: 0/0]]
Fatal error: Util_sun_misc_Signal.ensureInitialized: CSunMiscSignal.open() failed.
RequestId: 263ff1be-425d-4dcb-9ea5-67020dc3041b Error: Runtime exited with error: exit status 99
Runtime.ExitError
```

## The fight

After some Googling I've discovered several related clues [here](https://github.com/oracle/graal/issues/841) and [here](https://github.com/quarkusio/quarkus/issues/4262). They say that signals are not supported in AWS lambda. So, why not to disable signals for babashka and see what happens? I've forked the repo, made a flag that disables PIPE signal handling, deployed babashka to the [docker hub](https://hub.docker.com/r/dainiusjocas/babashka) and tried to deploy lambda once again.

And? It worked:
```shell script
make function-name=$(make get-function-name) invoke-function
=>
{"test":"test914"}{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
```

## Summary

[Here](https://github.com/dainiusjocas/babashka-lambda) is the example of babashka script that can be deployed to AWS Lambda.

- The `function.zip` weights just 18MB.
- The cold startup of the Lambda that is given 128MB of RAM is ~400ms. Subsequent calls ranges from 4ms and 120ms. The more RAM you give the faster lambda gets.
- I can develop the code in Cursive as the structure is like of an ordinary Clojure deps.edn project (and it can be used on the JVM).
- I made a [PR to babashka](https://github.com/borkdude/babashka/pull/305) and I've got accepted.

## Next Steps

- Fix Problem building on macos (`/tmp` dir is not writable).
- Get rid of AWS CloudFormation part.
- Work a bit more to support AWS API Gateway.
- Create a template for such projects.
