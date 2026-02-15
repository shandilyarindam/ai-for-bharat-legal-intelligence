# AttorneyCare - System Design Document

## 1. System Overview

AttorneyCare is a serverless, AI-powered legal document intelligence platform built on AWS cloud infrastructure. The system employs a three-tier architecture: React-based frontend, AWS serverless backend, and Amazon Bedrock AI layer.

### 1.1 Design Principles

- **Serverless-First**: Minimize operational overhead using managed AWS services
- **Security by Design**: Encryption, audit trails, and RBAC at every layer
- **AI Transparency**: Explainable AI outputs with confidence scores
- **Scalability**: Auto-scaling components to handle variable load
- **Cost Optimization**: Pay-per-use pricing model with intelligent caching

### 1.2 Key Design Decisions

- DynamoDB for flexible schema and automatic scaling
- Lambda for stateless, event-driven processing
- S3 for durable document storage with lifecycle policies
- Bedrock for managed LLM without infrastructure management
- CloudFront for global content delivery and caching

## 2. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                                │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  React.js + TypeScript Frontend (Tailwind CSS)               │   │
│  │  - Document Upload UI    - Clause Review Panel               │   │
│  │  - Risk Dashboard        - Audit Trail Viewer                │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  │ HTTPS (TLS 1.3)
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      EDGE & SECURITY LAYER                           │
│  ┌──────────────────┐      ┌─────────────────────────────────────┐ │
│  │  CloudFront CDN  │──────│  AWS WAF (Rate Limiting, DDoS)      │ │
│  └──────────────────┘      └─────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                         API GATEWAY LAYER                            │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  Amazon API Gateway (REST API)                               │   │
│  │  - JWT Authentication    - Request Validation                │   │
│  │  - Rate Limiting         - CORS Configuration                │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      COMPUTE & BUSINESS LOGIC                        │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Auth Lambda    │  │  Document Lambda │  │  Analysis Lambda │   │
│  │  - Login        │  │  - Upload        │  │  - Risk Scoring  │   │
│  │  - Register     │  │  - OCR Trigger   │  │  - Explanation   │   │
│  │  - Token Verify │  │  - Extraction    │  │  - Draft Detect  │   │
│  └─────────────────┘  └──────────────────┘  └──────────────────┘   │
│                                                                       │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  Audit Lambda   │  │  Export Lambda   │  │  Admin Lambda    │   │
│  │  - Log Events   │  │  - PDF Report    │  │  - User Mgmt     │   │
│  │  - Query Logs   │  │  - JSON Export   │  │  - Role Assign   │   │
│  └─────────────────┘  └──────────────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
