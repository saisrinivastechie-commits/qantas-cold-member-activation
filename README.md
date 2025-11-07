# Qantas Cold Member Activation Feature - Architectural Design

## Project Overview

This repository contains the architectural design documentation for the Qantas Pay Cold Member Activation feature - a backend system designed to convert dormant white card members into active users through a targeted incentive program.

## Challenge Summary

**Business Objective**: Convert cold members (enrolled but unused white cards) into active Qantas Pay users

**Incentive Program**:
- **Goal**: Load $1000 AUD cumulative within 30 days of enrollment
- **Reward**: 2000 bonus Qantas Points
- **Constraints**: Funds must not be withdrawn during the 30-day period

## Documentation Structure

- [`architectural-design-document.md`](./architectural-design-document.md) - Complete architectural design document (5-7 pages)
- [`diagrams/`](./diagrams/) - System architecture and data flow diagrams
- [`state-machine-design.md`](./state-machine-design.md) - Member lifecycle state management
- [`technical-decisions.md`](./technical-decisions.md) - Detailed technology justifications (available upon request)
- [`implementation-roadmap.md`](./implementation-roadmap.md) - Implementation Roadmap  (available upon request)

## Key Architectural Components

1. **Event-Driven Microservices Architecture**
2. **Dual-Path Data Reconciliation Engine**
3. **Time-Bound Execution Framework**
4. **Idempotent Reward Issuance System**
5. **Anti-Fraud Withdrawal Tracking**

## Technology Stack

- **Cloud Provider**: AWS
- **Compute**: AWS Lambda, ECS Fargate
- **Storage**: Amazon Aurora PostgreSQL, Amazon DynamoDB
- **Messaging**: Amazon EventBridge, Amazon SQS, Amazon SNS
- **Orchestration**: AWS Step Functions
- **Scheduling**: Amazon EventBridge Scheduler
- **Observability**: CloudWatch, X-Ray, CloudWatch Logs Insights

## Author

**Srini Kandimalla**
- Sr. Full Stack Developer / Solution Designer / Tech Lead
- 17+ years experience in distributed systems, financial services, and cloud architecture
- Australian Citizen

## Assessment Criteria

This design addresses:
- ✅ Scalability for high-volume transaction processing
- ✅ Resilience and fault tolerance in distributed systems
- ✅ Financial data integrity and consistency
- ✅ Reliable time-bound state management
- ✅ Clear architectural trade-offs and justifications