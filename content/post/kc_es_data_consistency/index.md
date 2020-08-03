---
aliases: [kafka-connect-elasticsearch-versioned-deletes]
projects: [elasticsearch]
title: A Fix for a Kafka Connect Elasticsearch data consistency issue
authors: [Dainius Jocas]
date: '2020-08-03'
tags: [Elasticsearch, Kafka Connect]
categories:
  - Elasticsearch
  - Kafka Connect
summary: A shout-out about a lurking bug in the Kafka Connect Elasticsearch Sink connector
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

When the [Elasticsearch indexer](https://github.com/confluentinc/kafka-connect-elasticsearch) is highly concurrent, Kafka record keys are used as Elasticsearch document IDs, and indexer is set to delete records on `null` values, then Kafka Connect might corrupt your data: documents that should not be deleted, end up being deleted. The fix is to use external versioning for deletes in bulk requests.

### The problem

As of version 5.5.1 of the Confluent Platform (last checked on 2020-08-02) the bug that might lead to data corruption is still present.

Elasticsearch uses [optimistic concurrency control](https://www.elastic.co/guide/en/elasticsearch/reference/current/optimistic-concurrency-control.html). The job of this concurrency mechanism is to ensure that older version of the document doesn't override a newer version. By default, order of arrival is applied, but the behaviour can be overriden in [several ways](https://www.elastic.co/blog/elasticsearch-versioning-support) depending on the version of Elasticsearch.

Here we are interested in cases when bulk requests arrive out of order. To help Elasticsearch resolve the out-of-order indexing requests Kafka Connect leverages the [`external` document](https://github.com/confluentinc/kafka-connect-elasticsearch/blob/7256e9473cea690c373058b88fffd1111870cfe6/src/main/java/io/confluent/connect/elasticsearch/jest/JestElasticsearchClient.java#L564) versioning[^1]. Using external versions in Kafka Connect makes sense because we already have versioning in place: kafka topic offsets. If Kafka Connect applies changes to Elasticsearch indices in order of the topic offset, then any ordering problems would be problems in the upstream system. 

When Kafka Record keys are not used as Elasticsearch document IDs versioning is not a problem because every ID is constructed as `{topic}+{partition}+{offset}` which creates new document for every kafka record. But when Kafka Keys are used as Elasticsearch document IDs things get more interesting. 

Let's add to the mix delete operations. Kafka Connect supports a setting `BEHAVIOR_ON_NULL_VALUES_CONFIG` to `"delete"`. This setting instructs that a document in Elasticsearch with an ID of the kafka record key with `null` value is going to be deleted. But for some strange reason the deletes **does not use external versioning**! The line responsible for the described behaviour is [here](https://github.com/confluentinc/kafka-connect-elasticsearch/blob/7256e9473cea690c373058b88fffd1111870cfe6/src/main/java/io/confluent/connect/elasticsearch/jest/JestElasticsearchClient.java#L554). This means that for deletes the order-of-arrival wins. Let's increase the concurrency of bulk requests with the param `MAX_IN_FLIGHT_REQUESTS_CONFIG` to a largish number, and the data consistency problems is just round the corner for data that has large update ratio.

### The Example

The code that demonstrated the faulty behaviour can be found in this [Pull Request](https://github.com/confluentinc/kafka-connect-elasticsearch/pull/422). 

Let's walk throught the code snippet:
```java
Collection<SinkRecord> records = new ArrayList<>();
for (int i = 0; i < numOfRecords - 1 ; i++) {
  if (i % 2 == 0) {
    SinkRecord sinkRecord = new SinkRecord(TOPIC, PARTITION, Schema.STRING_SCHEMA, key, schema, null, i);
    records.add(sinkRecord);
  } else {
    record.put("message", Integer.toString(i));
    SinkRecord sinkRecord = new SinkRecord(TOPIC, PARTITION, Schema.STRING_SCHEMA, key, schema, record, i);
    records.add(sinkRecord);
  }
}
record.put("message", Integer.toString(numOfRecords));
SinkRecord sinkRecord = new SinkRecord(TOPIC, PARTITION, Schema.STRING_SCHEMA, key, schema, record, numOfRecords);
records.add(sinkRecord);

task.put(records);
task.flush(null);
```

Here we send `numOfRecords` which larger than 2 to a Kafka topic. Every second record has `null` body (delete operation) and the rest of the records have a sequence number as a `message` value. The very last record is **always** a non-null record with a `message` value of `numOfRecords`.

Let's setup a connector:
```java
KEY_IGNORE_CONFIG = "false";
MAX_IN_FLIGHT_REQUESTS_CONFIG = Integer.toString(numOfRecords)
BATCH_SIZE_CONFIG = "1"
LINGER_MS_CONFIG = "1"
BEHAVIOR_ON_NULL_VALUES_CONFIG = "delete"
```
Here we set a connector to use Kafka record key as id `KEY_IGNORE_CONFIG = "false"`, set the indexer concurrency to the `numOfRecords`; set the indexing batch size to 1 (this creates as many requests to Elasticsearch as there are records in the Kafka topic); set indexer to send requests immediately with `LINGER_MS_CONFIG = "1"`; and record with a `null` value represents a delete. 

With this setup after the indexing is done we expect that in the index we have a document with ID and whose `message` value is `numOfRecords`. But when ordering of bulk requests is out-of-order then at the end we might have a situation where there is no document in the index at all: the bulk index request with `message = numOfRecords` arrived before one of the bulk requests with the delete operation! 

The situation might seem to be a bit far-fetched but for applications like e-commerce where you have a catalog that is frequently updated (e.g. the catalog item is present or not) and updates are modelled with document deletes it happens a bit more often than it might be expected.

### The fix

The fix is simple: use the same external versioning that is already being used by the indexing requests also for delete requests:
```java
if (record.version != null) {
    req.setParameter("version_type", "external").setParameter("version", record.version);
}
```

The full code can be found [here](ttps://github.com/confluentinc/kafka-connect-elasticsearch/pull/422). Let's hope that Confluent developers will find some time to merge that PR.

### Conclusion

Thank you for reading and leave your feedback [here](https://github.com/dainiusjocas/blog/issues/12).

### P.S.
 
Of course, this is not the only situation when data can get [corrupted](https://github.com/confluentinc/kafka-connect-elasticsearch/issues), e.g. when you delete the topic, repopulate it with up-to-date data (also, you skip deletes), and indexing does pretty much nothing, because all the versions are earlier `external version`. But let's leave them aside.

### 

[^1]: Elasticsearch 7 supports the [external versioning](https://www.elastic.co/guide/en/elasticsearch/reference/current/breaking-changes-7.0.html#_internal_versioning_is_no_longer_supported_for_optimistic_concurrency_control).



