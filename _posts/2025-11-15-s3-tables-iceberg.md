---
layout: post
title: "Understanding S3 Tables with Apache Iceberg Integration"
date: 2024-11-29
categories: [data-engineering, aws, iceberg, duckdb]
tags: [s3, apache-iceberg, data-lakes, aws, duckdb]
---

## Introduction

Recently I was involved in migrating an earlier project that was streaming data from our Postgres database, using debezium CDC connector and Kafka, to S3 buckets using an s3 parquet sink.
This project has drawbacks - the metadata files proliferate, and reading without metadata management because very non-performant. The new solution would instead used S3 tables.

## What Are S3 Tables?

S3 Tables are AWS-managed table buckets that provide built-in support for Apache Iceberg table format. Unlike traditional S3 buckets where you manage the Iceberg metadata separately, S3 Tables handle this integration natively, simplifying the operational overhead. 

## Key Concepts

### Apache Iceberg Benefits

Apache Iceberg brings ACID transactions to data lakes, solving several critical challenges:

- **Schema Evolution**: Add, drop, or rename columns without rewriting data
- **Time Travel**: Query historical versions of your data
- **Partition Evolution**: Change partitioning schemes without data migration
- **Hidden Partitioning**: Users query without knowing partition structure

### Integration Advantages

The S3 Tables integration with Iceberg provides:

1. **Automatic Metadata Management**: AWS handles Iceberg metadata storage and catalog operations
2. **Performance Optimization**: Built-in compaction and optimization services
3. **Query Engine Compatibility**: Works with Athena, EMR, Spark, and other Iceberg-compatible engines
4. **Simplified Governance**: Integration with AWS Lake Formation and IAM

## Practical Use Case - Streaming CDC to downstream team

0. Creating s3 tables
1. CDC using debezium Kafka Connect
2. Kafka topics - one per table + one control topic
3. Kafka iceberg sinks - using older and newer iceberg sink connnectors
4. Whole table dumps using duckdb

## Resources

- [AWS S3 Tables Documentation](https://docs.aws.amazon.com/s3/latest/userguide/s3-tables.html)
- [Apache Iceberg Documentation](https://iceberg.apache.org/)

