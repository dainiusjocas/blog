---
aliases: [elasticsearch-ingest-pipeline-kafka-connect]
projects: [elasticsearch]
title: How to Use Elasticsearch Ingest Pipelines with Kafka Connect Elasticsearch Sink Connector
authors: [Dainius Jocas]
date: '2020-08-02'
tags: [Elasticsearch, Kafka Connect]
categories:
  - Elasticsearch
  - Kafka Connect
summary: A workaround on how to leverage the Elasticsearch Ingest Pipelines when using Kafka Connect
image:
  caption: "Kafka Connect Elasticsearch Sink"
  focal_point: "Center"
  placement: 1
  preview_only: false
output:
  blogdown::html_page:
    toc: true
    number_sections: true
    toc_depth: 1
---

### TL;DR

Specify your pipeline with the [`index.default_pipeline`](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#dynamic-index-settings) setting in the index (or index template) settings.

### The Problem

We need to index the log data into the [Elasticsearch](https://www.elastic.co/) cluster using a [Kafka Connect Elasticsearch Sink Connector](https://docs.confluent.io/current/connect/kafka-connect-elasticsearch/index.html) [^1], the data should be split into daily indices, and we need to specify the Elasticsearch ingest pipeline.

The [documentation of the connector](https://docs.confluent.io/current/connect/kafka-connect-elasticsearch/configuration_options.html) doesn't mention anything about ingest pipelines. After a quick consultation with the Internet you discover that there is an open [issue](https://github.com/confluentinc/kafka-connect-elasticsearch/issues/72) that Kafka Connect Elasticsearch Sink Connector doesn't support specifying an Elasticsearch ingest pipeline. WAT?

### The Workaround

Say[^4], our pipeline[^5] just renames an attribute, e.g.:
```shell script
PUT _ingest/pipeline/my_pipeline_id
{
  "description" : "renames the field name",
  "processors" : [
    {
      "rename": {
          "field": "original_field_name",
          "target_field": "target_field_name"
        }
    }
  ]
}
```

The Elasticsearch ingest pipeline for indexing can be specified in several ways:
1. for each index request as a [URL parameter](https://www.elastic.co/guide/en/elasticsearch/reference/master/ingest.html),
2. per bulk index request as a URL parameter,
3. for every [bulk index request operation](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html#docs-bulk-api-query-params),
4. index settings ([a dynamic attribute](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules.html#dynamic-index-settings)),
5. index template.

First three options are not supported by Kafka Connect. The fourth option is not convenient in our case because the data should be split into time-based (e.g. daily) indices and we don't want to do repetitive tasks[^3]. The natural option to follow is to define an [index template](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html). In the index template we can specify the `index.default_pipeline` parameter, e.g.
```shell script
PUT _index_template/template_1
{
  "index_patterns": ["daily_log*"],
  "template": {
    "settings": {
      "index.default_pipeline": "my_pipeline_id"
    }
  }
}
``` 
Note, that for indexing not to fail, we should create the Elasticsearch ingest pipeline[^2] **before** setting up the index template.

That is it, now when Kafka Connect will create a new daily index the Elasticsearch ingest pipeline is going to be applied to every document without any issues, for free, and in no time.
  
### Bonus

One thing to note is that only one pipeline can be specified for `index.default_pipeline` which might sound a bit limiting. A clever trick to overcome that limitation is to use a series of [pipeline processors](https://www.elastic.co/guide/en/elasticsearch/reference/current/pipeline-processor.html) that can invoke other pipelines in the specified order, i.e. pipeline of pipelines.

Also, there is an index setting called `index.final_pipeline` that if specified is going to be executed after all other pipelines.

Testing pipelines can be done using the [`_simulate` API](https://www.elastic.co/guide/en/elasticsearch/reference/master/simulate-pipeline-api.html).

### Fin

Thanks for reading and leave comments or any other feedback on this blog post in the [Github issue](https://github.com/dainiusjocas/blog/issues/9). Examples were tested to work with Elasticsearch and Kibana 7.8.1.

[^1]: or any other technology that doesn't support, or it is just not possible to specify the Elasticsearch ingest pipeline.
[^2]: technically, before an index is created that matches the template pattern.
[^3]: set `index.default_pipeline=my_pipeline_id` for every new daily index with, say, a cron-job at midnight.
[^4]: yes, I know that the same job can be done with the [Kafka Connect Transformations](https://docs.confluent.io/current/connect/transforms/index.html).
[^5]: let's leave out the Kafka Connector setup.
