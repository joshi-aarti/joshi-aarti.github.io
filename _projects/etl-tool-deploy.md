---
title: "Event-Driven Data Loader Architecture"
layout: single
date: 2025-09-01
excerpt: "AWS-based event-driven ETL pipeline for file loading and conversion with Step Functions, S3, EFS, and PostgreSQL."
tags: ["AWS", "ETL", "Step Functions", "S3", "EventBridge", "PostgreSQL", "DevOps"]
---

{% include figure image_path="/assets/images/etl-tool-loader.png" alt="ETL tool loader architecture diagram" popup=true %}

This document describes an AWS-based, event-driven data processing pipeline for file loading and conversion. The design uses only AWS publicly available services and terminology to achieve **reusability**, **scalability**, and **performance**.

---

## High-Level Overview

The pipeline is triggered when a new file is uploaded to an **Amazon S3** bucket. An **Amazon EventBridge** rule starts an **AWS Step Functions** orchestrator, which runs either a **Loader** workflow (transform and load into PostgreSQL) or a **File Convertor** workflow (format conversion and validation). **Amazon CloudWatch Logs** provides centralized logging, and **Amazon SNS** or **Amazon EventBridge** is used to emit operational events (success/failure).

---

## Architecture Components

### 1. Ingress and Orchestration

| Component | AWS Service | Role |
|-----------|-------------|------|
| **Input** | Amazon S3 | "Input file bucket" – source of new files; **Object Created** events drive the pipeline |
| **Event bus** | Amazon EventBridge | Receives S3 events and routes them to the orchestrator |
| **Orchestrator** | AWS Step Functions | Runs the chosen state machine (Loader or File Convertor) |
| **Logging** | Amazon CloudWatch Logs | Centralized log collection and monitoring across components |

**Flow:** New file uploaded → S3 Object Created → EventBridge → Step Functions state machine started → CloudWatch Logs for observability.

---

### 2. Shared Data Stores and Services

| Component | AWS Service | Role |
|-----------|-------------|------|
| **Job metadata** | Amazon DynamoDB | Job records and metadata (job ID, status, configuration) for both workflows |
| **Working storage** | Amazon EFS (Elastic File System) | Shared, scalable file system for intermediate data (S3 → EFS → processing → EFS → S3) |
| **Relational data** | Amazon RDS for PostgreSQL | Target database for loaded data in the Loader workflow |
| **Output storage** | Amazon S3 | "Output file bucket" – destination for processed/loaded artifacts |
| **Notifications** | Amazon SNS / Amazon EventBridge | Operational events (success/failure) for alerting and downstream integration |

These services are **shared** across the Loader and File Convertor workflows, reducing duplication and standardizing patterns.

---

### 3. Loader State Machine Workflow

ETL path: transform input files and load into PostgreSQL.

| Step | Action | AWS / Pattern |
|------|--------|----------------|
| 1 | Parse S3 input; create job ID; create job record with metadata | S3 + DynamoDB |
| 2 | Run **s3-efs-downloader** – copy input files from S3 to EFS (or EBS volume) | S3, EFS, processing component (e.g. Lambda or ECS) |
| 3 | **ETL tool** transforms input data to PostgreSQL-ready format; generate `.sql` files and save to EFS | EFS path: `/data/<jobid>/<ddmmyyyyhhmmss>/file.zip` (or equivalent) |
| 4 | Run **sql-processor** – read SQL from EFS and load data into PostgreSQL | EFS, RDS for PostgreSQL |
| 5 | Run **efs-s3-uploader** – copy processed files from EFS to output S3 bucket | EFS, S3 |
| 6 | Emit operational events (success/failure) to SNS topic or EventBridge | SNS / EventBridge |

---

### 4. File Convertor State Machine Workflow

Conversion and validation path.

| Step | Action | AWS / Pattern |
|------|--------|----------------|
| 1 | Parse S3 input; create job ID; create job record with metadata | S3 + DynamoDB |
| 2 | Run **s3-efs-downloader** – copy input files from S3 to EFS | S3, EFS |
| 3 | **Converts files to final format** – read from EFS, perform conversion | EFS, processing component |
| 4 | **Validation via reconversion** – read converted file from EFS, validate, write final file back to EFS | EFS |
| 5 | Emit operational events (success/failure) to SNS or EventBridge | SNS / EventBridge |

---

## Reusability

- **Modular workflows:** The Step Functions orchestrator can invoke either the Loader or File Convertor state machine based on input or routing logic. Workflows are separate, reusable units.
- **Shared services:** Amazon S3 (input/output), Amazon DynamoDB (metadata), and Amazon EFS (working files) are used by both workflows. No duplicate storage or metadata stores for each use case.
- **Event-driven decoupling:** Amazon EventBridge decouples producers (e.g. S3) from consumers (Step Functions, and future subscribers). New workflows or subscribers can be added without changing existing producers.
- **Consistent job model:** A single job metadata model in DynamoDB supports both loader and converter flows, enabling consistent tracking and operational tooling.

---

## Scalability

- **Serverless and managed services:** The design uses fully managed, scalable AWS services (S3, EventBridge, Step Functions, DynamoDB, EFS, SNS, CloudWatch Logs, and typically Lambda or Amazon ECS/Fargate for processing). They scale with workload without manual capacity management.
- **Amazon S3:** Effectively unlimited storage and throughput for object storage.
- **AWS Step Functions:** Supports large numbers of concurrent workflow executions, scaling with demand.
- **Amazon DynamoDB:** Single-digit millisecond latency at scale with on-demand or provisioned capacity.
- **Amazon EFS:** File system that scales in size and supports many concurrent clients; suitable for shared working directories.
- **Amazon RDS for PostgreSQL:** Vertical scaling (instance size) and horizontal scaling (read replicas) to match load.

---

## Performance

- **Asynchronous, event-driven execution:** Work is triggered by events and runs asynchronously. Multiple jobs can run in parallel; initiation is decoupled from completion, improving throughput.
- **Step Functions:** Built-in error handling, retries, and choice states improve reliability and reduce manual intervention on failures.
- **EFS for processing:** Moving files from S3 to EFS for transformation and SQL generation allows fast, shared file-system access during compute-heavy steps instead of repeated S3 API calls.
- **Managed service optimizations:** Using AWS managed services benefits from AWS network and storage optimizations, multi-AZ options, and regional placement for low latency.

---

## Summary

| Attribute | How it is addressed |
|-----------|----------------------|
| **Reusability** | Shared S3, DynamoDB, EFS; separate Loader and File Convertor state machines; EventBridge for loose coupling; single job metadata model. |
| **Scalability** | Serverless and managed services (S3, EventBridge, Step Functions, DynamoDB, EFS, SNS, CloudWatch); scaling without manual provisioning. |
| **Performance** | Event-driven, asynchronous processing; EFS for efficient file access during ETL; Step Functions reliability and retries; AWS managed service performance. |

All components and terms used in this writeup refer to **AWS publicly available services and features** (Amazon S3, Amazon EventBridge, AWS Step Functions, Amazon DynamoDB, Amazon EFS, Amazon RDS for PostgreSQL, Amazon SNS, Amazon CloudWatch Logs, AWS Lambda, Amazon ECS, AWS Fargate) as described in AWS documentation.
