# EventXGames AWS Architecture

This repository contains the target AWS architecture design for the EventXGames platform migration from Azure.

## Contents

- [AWS_ARCHITECTURE.md](./docs/AWS_ARCHITECTURE.md) - Complete architecture document

## Architecture Overview

```
                                ┌─────────────────────────────────────────────────────────────┐
                                │                        AWS Cloud                             │
┌──────────┐                    │  ┌─────────────┐     ┌──────────────────────────────────┐   │
│  Users   │───────────────────────│  CloudFront │────▶│           S3 (Assets)            │   │
└──────────┘                    │  └──────┬──────┘     └──────────────────────────────────┘   │
                                │         │                                                    │
                                │         ▼                                                    │
                                │  ┌─────────────┐     ┌──────────────────────────────────┐   │
                                │  │     ALB     │────▶│           EKS Cluster            │   │
                                │  └─────────────┘     └──────────────────────────────────┘   │
                                │                                     │                        │
                                │                                     ▼                        │
                                │  ┌──────────────────────────────────────────────────────┐  │
                                │  │              Aurora PostgreSQL (Multi-AZ)            │  │
                                │  └──────────────────────────────────────────────────────┘  │
                                │                                     │                        │
                                │                                     ▼                        │
                                │  ┌──────────────────────────────────────────────────────┐  │
                                │  │              ElastiCache Redis (Cluster Mode)        │  │
                                │  └──────────────────────────────────────────────────────┘  │
                                │                                     │                        │
                                │                                     ▼                        │
                                │  ┌──────────────────────────────────────────────────────┐  │
                                │  │                    Amazon Bedrock                    │  │
                                │  └──────────────────────────────────────────────────────┘  │
                                └─────────────────────────────────────────────────────────────┘
```

## Key Components

### Compute (EKS)
- **Kubernetes Version:** 1.29
- **Region:** us-east-1 (Primary), us-west-2 (DR)
- **Availability Zones:** 3 AZs minimum
- **Node Groups:** System, Application, Spot

### Database (Aurora PostgreSQL)
- **Engine Version:** 15.4
- **Configuration:** Multi-AZ with 1 writer + 2 readers
- **Instance Types:** db.r6g.2xlarge (writer), db.r6g.xlarge (readers)
- **Storage:** Encrypted with CMK

### Cache (ElastiCache Redis)
- **Engine Version:** 7.1
- **Mode:** Cluster mode enabled
- **Shards:** 3 shards with 2 replicas each
- **Instance Type:** cache.r6g.xlarge

### CDN (CloudFront + S3)
- **Origin:** S3 bucket with OAC
- **TLS:** 1.2 minimum
- **WAF:** Enabled with managed rules

### AI (Amazon Bedrock)
- **Model:** Claude 3 Sonnet/Haiku
- **Features:** Content filtering, guardrails
- **Access:** IRSA-based from EKS

## Cost Estimates

| Component | Monthly Cost |
|-----------|-------------|
| EKS Cluster | ~$2,000 |
| Aurora PostgreSQL | ~$2,600 |
| ElastiCache Redis | ~$4,150 |
| CloudFront + S3 | ~$1,000 |
| Bedrock | Variable |
| **Total Base** | **~$10,000/month** |

## Quick Links

- [Main Project Index](https://github.com/eventxgames-nclouds/nclouds-overview)
- [Infrastructure Code](https://github.com/eventxgames-nclouds/nclouds-infrastructure)
- [Security Documentation](https://github.com/eventxgames-nclouds/nclouds-security)
