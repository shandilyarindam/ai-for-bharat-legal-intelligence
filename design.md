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
┌─────────────────────────────────────────────────────────────────────┐
│                      DATA & STORAGE LAYER                            │
│  ┌─────────────────┐  ┌──────────────────┐  ┌──────────────────┐   │
│  │  DynamoDB       │  │  S3 Buckets      │  │  Amazon Bedrock  │   │
│  │  - Users        │  │  - Documents     │  │  - Claude 3      │   │
│  │  - Documents    │  │  - Reports       │  │  - Titan Embed   │   │
│  │  - Clauses      │  │  - Backups       │  │                  │   │
│  │  - AuditLogs    │  └──────────────────┘  └──────────────────┘   │
│  └─────────────────┘                                                 │
└─────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    MONITORING & OBSERVABILITY                        │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐  │
│  │  CloudWatch      │  │  CloudTrail      │  │  X-Ray           │  │
│  │  - Metrics       │  │  - Audit Trail   │  │  - Tracing       │  │
│  │  - Logs          │  │  - Compliance    │  │  - Performance   │  │
│  │  - Alarms        │  └──────────────────┘  └──────────────────┘  │
│  └──────────────────┘                                                │
└─────────────────────────────────────────────────────────────────────┘
```

## 3. Database Schema Design

### 3.1 DynamoDB Table: Users

**Table Name**: `AttorneyCare-Users`  
**Partition Key**: `userId` (String)  
**Billing Mode**: On-Demand

**Attributes**:
- `userId` (PK): Unique user identifier (UUID)
- `email`: User email address
- `role`: User role (Viewer, Reviewer, Admin, Auditor)
- `cognitoId`: Amazon Cognito user pool ID
- `createdAt`: ISO 8601 timestamp
- `lastLogin`: ISO 8601 timestamp
- `isActive`: Boolean flag

**GSI-1**: `EmailIndex`
- Partition Key: `email`
- Use Case: User lookup by email during authentication

### 3.2 DynamoDB Table: Documents

**Table Name**: `AttorneyCare-Documents`  
**Partition Key**: `documentId` (String)  
**Sort Key**: `version` (Number)  
**Billing Mode**: On-Demand

**Attributes**:
- `documentId` (PK): Unique document identifier (UUID)
- `version` (SK): Version number (starts at 1)
- `userId`: Owner user ID
- `fileName`: Original file name
- `s3Key`: S3 object key for original file
- `fileType`: MIME type (application/pdf, application/docx)
- `fileSize`: Size in bytes
- `uploadedAt`: ISO 8601 timestamp
- `processingStatus`: PENDING | PROCESSING | COMPLETED | FAILED
- `extractionMethod`: OCR engine used (Textract, Tesseract)
- `totalClauses`: Number of extracted clauses
- `metadata`: Map of custom metadata

**GSI-1**: `UserDocumentsIndex`
- Partition Key: `userId`
- Sort Key: `uploadedAt`
- Use Case: List all documents for a user, sorted by upload time

**GSI-2**: `StatusIndex`
- Partition Key: `processingStatus`
- Sort Key: `uploadedAt`
- Use Case: Query documents by processing status for monitoring

### 3.3 DynamoDB Table: Clauses

**Table Name**: `AttorneyCare-Clauses`  
**Partition Key**: `clauseId` (String)  
**Sort Key**: `version` (Number)  
**Billing Mode**: On-Demand

**Attributes**:
- `clauseId` (PK): Unique clause identifier (UUID)
- `version` (SK): Version number for clause versioning
- `documentId`: Parent document reference
- `clauseNumber`: Original clause numbering (e.g., "3.2.1")
- `category`: Clause category (PAYMENT, LIABILITY, TERMINATION, IP, etc.)
- `extractedText`: Full clause text content
- `originalPosition`: Page and position in original document
- `extractedAt`: ISO 8601 timestamp
- `extractionConfidence`: Confidence score (0.0 - 1.0)
- `formattingMetadata`: JSON object with formatting details

**GSI-1**: `DocumentClausesIndex`
- Partition Key: `documentId`
- Sort Key: `originalPosition`
- Use Case: Retrieve all clauses for a document in order

**GSI-2**: `CategoryIndex`
- Partition Key: `category`
- Sort Key: `extractedAt`
- Use Case: Query clauses by category across documents

**Clause Versioning Strategy**:
- Each clause extraction creates a new version entry
- Version 1 is the initial extraction
- Re-processing creates version 2, 3, etc.
- Allows comparison of extraction improvements over time
- Immutable history for audit compliance

### 3.4 DynamoDB Table: AIAnalysisResults

**Table Name**: `AttorneyCare-AIAnalysis`  
**Partition Key**: `analysisId` (String)  
**Billing Mode**: On-Demand

**Attributes**:
- `analysisId` (PK): Unique analysis identifier (UUID)
- `clauseId`: Reference to analyzed clause
- `documentId`: Reference to parent document
- `analysisType`: RISK_ASSESSMENT | EXPLANATION | DRAFT_DETECTION
- `riskLevel`: LOW | MEDIUM | HIGH | CRITICAL
- `confidenceScore`: AI confidence (0.0 - 1.0)
- `explanation`: Plain-language explanation text
- `draftDetectionScore`: Probability of AI-generated content (0.0 - 1.0)
- `modelVersion`: Bedrock model identifier (e.g., "anthropic.claude-3-sonnet")
- `promptVersion`: Version of prompt template used
- `analyzedAt`: ISO 8601 timestamp
- `processingTimeMs`: Analysis duration in milliseconds
- `userApprovalStatus`: PENDING | ACCEPTED | REJECTED | null
- `userApprovedAt`: ISO 8601 timestamp of user decision

**GSI-1**: `ClauseAnalysisIndex`
- Partition Key: `clauseId`
- Sort Key: `analyzedAt`
- Use Case: Retrieve all analyses for a specific clause

**GSI-2**: `DocumentRiskIndex`
- Partition Key: `documentId`
- Sort Key: `riskLevel`
- Use Case: Query high-risk clauses for a document

### 3.5 DynamoDB Table: AuditLogs

**Table Name**: `AttorneyCare-AuditLogs`  
**Partition Key**: `logId` (String)  
**Sort Key**: `timestamp` (Number)  
**Billing Mode**: On-Demand

**Attributes**:
- `logId` (PK): Unique log entry identifier (UUID)
- `timestamp` (SK): Unix timestamp in milliseconds
- `userId`: User who performed the action
- `actionType`: Action category (DOCUMENT_UPLOAD, CLAUSE_VIEW, AI_QUERY, etc.)
- `resourceType`: Resource affected (Document, Clause, User)
- `resourceId`: ID of affected resource
- `actionDetails`: JSON object with action-specific data
- `ipAddress`: Client IP address (masked for privacy)
- `userAgent`: Client user agent string
- `previousHash`: SHA-256 hash of previous log entry
- `currentHash`: SHA-256 hash of this entry (timestamp + userId + actionType + previousHash)
- `isAIGenerated`: Boolean flag indicating if action was AI-generated
- `aiRecommendation`: Original AI recommendation (if applicable)
- `userDecision`: User's decision on AI recommendation (ACCEPTED | REJECTED | null)

**GSI-1**: `UserActivityIndex`
- Partition Key: `userId`
- Sort Key: `timestamp`
- Use Case: Query all actions by a specific user

**GSI-2**: `ResourceAuditIndex`
- Partition Key: `resourceId`
- Sort Key: `timestamp`
- Use Case: Retrieve audit trail for a specific resource

**Append-Only Audit Model**:
- No UPDATE or DELETE operations allowed on audit logs
- Each entry contains hash of previous entry (blockchain-style chaining)
- Hash chain ensures tamper detection
- DynamoDB Streams trigger Lambda to verify hash integrity
- Any hash mismatch triggers security alert

**Hash Chain Implementation**:
```
Entry N:
  previousHash = SHA256(Entry N-1)
  currentHash = SHA256(timestamp + userId + actionType + previousHash)

Verification:
  For each entry, recompute currentHash and compare with stored value
  Verify previousHash matches previous entry's currentHash
```

## 4. AI Invocation Design

### 4.1 Prompt Structure Design

**Risk Assessment Prompt Template**:
```
You are a legal contract analysis assistant. Analyze the following contract clause and provide a structured risk assessment.

CLAUSE TEXT:
{clause_text}

CLAUSE CATEGORY: {category}
DOCUMENT TYPE: {document_type}

Provide your analysis in the following JSON format:
{
  "riskLevel": "LOW" | "MEDIUM" | "HIGH" | "CRITICAL",
  "confidenceScore": 0.0 to 1.0,
  "riskFactors": ["factor1", "factor2", ...],
  "explanation": "Plain-language explanation of risks",
  "recommendations": ["recommendation1", "recommendation2", ...]
}

Focus on:
- Unfair or one-sided terms
- Ambiguous language
- Hidden obligations
- Liability imbalances
- Termination conditions
```

**Explanation Prompt Template**:
```
You are a legal educator helping non-lawyers understand contract clauses.

CLAUSE TEXT:
{clause_text}

Provide a plain-language explanation suitable for someone without legal training.

Output JSON format:
{
  "simplifiedExplanation": "Easy-to-understand explanation",
  "keyTerms": [{"term": "...", "definition": "..."}],
  "realWorldExample": "Practical example of what this means",
  "potentialImpact": "What this could mean for the user"
}

Use simple language. Avoid legal jargon. Be concise.
```

**Draft Detection Prompt Template**:
```
Analyze the following text to determine if it was likely generated by an AI language model.

TEXT:
{clause_text}

Look for AI writing patterns:
- Generic phrasing
- Repetitive structure
- Lack of specific details
- Overly formal or stilted language
- Common AI boilerplate patterns

Output JSON format:
{
  "draftDetectionScore": 0.0 to 1.0,
  "confidence": 0.0 to 1.0,
  "indicators": ["indicator1", "indicator2", ...],
  "reasoning": "Explanation of detection"
}
```

### 4.2 JSON Structured Outputs

**Bedrock Invocation Configuration**:
- Model: `anthropic.claude-3-sonnet-20240229`
- Temperature: 0.3 (low for consistency)
- Max Tokens: 2048
- Top P: 0.9
- Response Format: JSON mode enabled

**JSON Schema Validation**:
- All AI responses validated against predefined JSON schemas
- Invalid responses trigger retry with schema reminder
- Maximum 3 retry attempts before fallback to error state

### 4.3 Temperature Strategy

**Risk Assessment**: Temperature = 0.3
- Requires consistency and reliability
- Low temperature reduces hallucination
- Deterministic risk scoring

**Explanation Generation**: Temperature = 0.5
- Allows slight creativity for readability
- Maintains factual accuracy
- Varied phrasing for user engagement

**Draft Detection**: Temperature = 0.2
- Highly deterministic analysis
- Pattern matching requires consistency

### 4.4 Confidence Scoring

**Confidence Score Calculation**:
- Model's internal confidence (from logprobs if available)
- Prompt clarity score (well-formed input)
- Response completeness (all required fields present)
- Historical accuracy (model performance on similar clauses)

**Confidence Thresholds**:
- High Confidence: > 0.85 (display result directly)
- Medium Confidence: 0.65 - 0.85 (display with caution notice)
- Low Confidence: < 0.65 (flag for human review, display with strong disclaimer)

### 4.5 Error Handling

**Bedrock API Errors**:
- `ThrottlingException`: Exponential backoff retry (max 5 attempts)
- `ModelTimeoutException`: Reduce input size, retry with truncated context
- `ValidationException`: Log prompt for debugging, return user-friendly error
- `ServiceUnavailableException`: Queue request for later processing

**Fallback Strategy**:
1. Primary: Claude 3 Sonnet
2. Fallback 1: Claude 3 Haiku (faster, cheaper)
3. Fallback 2: Return partial results with "AI unavailable" notice
4. Fallback 3: Queue for manual review

### 4.6 Bedrock Timeout Handling

**Timeout Configuration**:
- Lambda timeout: 60 seconds
- Bedrock invocation timeout: 45 seconds
- Buffer: 15 seconds for processing and logging

**Timeout Recovery**:
- Split large clauses into smaller chunks
- Process chunks sequentially
- Aggregate results
- If still timing out, mark for async processing via SQS

**Async Processing Flow**:
```
User Request → Lambda → SQS Queue → Background Lambda → Bedrock
                  ↓
            Return "Processing" status
                  ↓
            Poll for completion
```

## 5. Security Enforcement Flow

### 5.1 JWT Validation Sequence

```
┌──────────┐         ┌─────────────┐         ┌──────────────┐
│  Client  │────────>│ API Gateway │────────>│   Lambda     │
└──────────┘         └─────────────┘         └──────────────┘
     │                      │                        │
     │ 1. Request with      │                        │
     │    JWT in header     │                        │
     │─────────────────────>│                        │
     │                      │                        │
     │                      │ 2. Validate JWT        │
     │                      │    with Cognito        │
     │                      │    User Pool           │
     │                      │                        │
     │                      │ 3. Extract claims      │
     │                      │    (userId, role)      │
     │                      │                        │
     │                      │ 4. Pass claims in      │
     │                      │    request context     │
     │                      │───────────────────────>│
     │                      │                        │
     │                      │                        │ 5. Verify role
     │                      │                        │    authorization
     │                      │                        │
     │                      │                        │ 6. Execute
     │                      │                        │    business logic
     │                      │                        │
     │                      │<───────────────────────│
     │<─────────────────────│                        │
     │                      │                        │
```

**JWT Validation Steps**:
1. API Gateway extracts JWT from `Authorization: Bearer <token>` header
2. Gateway validates JWT signature using Cognito public keys
3. Gateway verifies token expiration and issuer
4. Gateway extracts claims: `userId`, `role`, `email`
5. Claims passed to Lambda via `requestContext.authorizer.claims`
6. Lambda performs role-based authorization check

### 5.2 Role-Based Authorization Flow

**Lambda Authorization Logic**:
```typescript
// Pseudo-code for Lambda authorization
function authorizeRequest(claims, resource, action) {
  const userRole = claims.role;
  const userId = claims.userId;
  
  // Role permission matrix
  const permissions = {
    Viewer: ['read:document', 'read:clause', 'read:analysis'],
    Reviewer: ['read:*', 'write:comment', 'request:explanation'],
    Admin: ['read:*', 'write:*', 'delete:*', 'manage:users'],
    Auditor: ['read:*', 'read:auditlog']
  };
  
  // Check if role has permission for action
  if (!hasPermission(permissions[userRole], action)) {
    throw new ForbiddenError('Insufficient permissions');
  }
  
  // Resource-level authorization
  if (resource.ownerId !== userId && userRole !== 'Admin') {
    throw new ForbiddenError('Access denied to resource');
  }
  
  return true;
}
```

**Authorization Enforcement Points**:
- API Gateway: JWT validation (authentication)
- Lambda Layer: Role-based authorization (authorization)
- DynamoDB: Resource ownership validation (data-level security)

### 5.3 IAM Boundary Model

**Principle of Least Privilege**:
- Each Lambda function has dedicated IAM role
- Roles grant only required permissions
- No wildcard permissions in production

**Lambda IAM Roles**:

**DocumentProcessingLambdaRole**:
- `s3:GetObject` on documents bucket
- `s3:PutObject` on documents bucket
- `dynamodb:PutItem` on Documents table
- `dynamodb:UpdateItem` on Documents table
- `textract:StartDocumentTextDetection`
- `textract:GetDocumentTextDetection`

**AnalysisLambdaRole**:
- `dynamodb:GetItem` on Clauses table
- `dynamodb:PutItem` on AIAnalysis table
- `bedrock:InvokeModel` on specific model ARNs
- `kms:Decrypt` for encrypted data

**AuditLambdaRole**:
- `dynamodb:PutItem` on AuditLogs table (append-only)
- `dynamodb:Query` on AuditLogs table
- No UPDATE or DELETE permissions

### 5.4 Encryption Key Strategy

**KMS Key Hierarchy**:

**Master Key**: `AttorneyCare-Master-Key`
- Used to encrypt data keys
- Automatic rotation enabled (annual)
- Multi-region replication for DR

**Data Encryption Keys**:
- `Documents-DEK`: Encrypts S3 documents
- `Database-DEK`: Encrypts DynamoDB tables
- `Logs-DEK`: Encrypts CloudWatch logs

**Encryption at Rest**:
- S3: SSE-KMS with Documents-DEK
- DynamoDB: Encryption enabled with Database-DEK
- CloudWatch Logs: Encrypted with Logs-DEK

**Encryption in Transit**:
- TLS 1.3 for all client connections
- VPC endpoints for AWS service communication
- Certificate pinning for mobile apps (future)

## 6. Failure Handling & Resilience

### 6.1 Retry Logic

**Exponential Backoff Strategy**:
```
Attempt 1: Immediate
Attempt 2: Wait 1 second
Attempt 3: Wait 2 seconds
Attempt 4: Wait 4 seconds
Attempt 5: Wait 8 seconds
Max Attempts: 5
```

**Retry Conditions**:
- Transient errors: `ThrottlingException`, `ServiceUnavailable`, `InternalServerError`
- Network timeouts
- Rate limit errors

**Non-Retryable Errors**:
- `ValidationException` (bad input)
- `AccessDeniedException` (permission issue)
- `ResourceNotFoundException` (missing resource)

### 6.2 Dead-Letter Queues

**DLQ Architecture**:
```
Lambda Function → Failure → SQS DLQ → Monitoring Alert
                                ↓
                          Manual Review Queue
                                ↓
                          Retry or Discard
```

**DLQ Configuration**:
- Maximum receive count: 3 (after 3 failures, move to DLQ)
- Message retention: 14 days
- CloudWatch alarm on DLQ depth > 10

**DLQ Processing**:
- Daily automated review of DLQ messages
- Admin dashboard shows failed operations
- Manual retry or investigation workflow

### 6.3 Partial Extraction Recovery

**Scenario**: Document processing fails mid-extraction

**Recovery Strategy**:
1. Save successfully extracted clauses to DynamoDB
2. Mark document status as `PARTIAL_COMPLETE`
3. Store failure point metadata (page number, clause index)
4. Allow user to:
   - View partial results
   - Retry extraction from failure point
   - Mark as complete and proceed

**Checkpoint Mechanism**:
- Save progress every 10 clauses
- Store checkpoint in DynamoDB
- Resume from last checkpoint on retry

### 6.4 Bedrock Fallback Strategy

**Multi-Model Fallback**:
```
Primary: Claude 3 Sonnet (high quality)
    ↓ (if unavailable or timeout)
Fallback 1: Claude 3 Haiku (faster, cheaper)
    ↓ (if unavailable)
Fallback 2: Titan Text (AWS native)
    ↓ (if all AI unavailable)
Fallback 3: Rule-based heuristics (basic risk scoring)
```

**Graceful Degradation**:
- If AI unavailable, provide basic clause extraction only
- Display notice: "AI analysis temporarily unavailable"
- Queue AI analysis for later processing
- Notify user when AI analysis completes

## 7. Scalability Design

### 7.1 Lambda Concurrency Model

**Concurrency Configuration**:
- Reserved concurrency per function: 100
- Provisioned concurrency for critical functions: 10
- Burst capacity: Up to 1,000 concurrent executions

**Function-Specific Limits**:
- `DocumentProcessingLambda`: 50 concurrent executions
- `AnalysisLambda`: 100 concurrent executions (AI-heavy)
- `AuditLambda`: 200 concurrent executions (high frequency)

**Throttling Strategy**:
- API Gateway throttling: 1,000 requests/second
- Per-user rate limit: 100 requests/minute
- Burst allowance: 200 requests

### 7.2 DynamoDB Scaling Mode

**On-Demand Capacity Mode**:
- Automatic scaling based on traffic
- No capacity planning required
- Pay per request

**Table-Specific Configuration**:
- `Users`: On-Demand (low traffic)
- `Documents`: On-Demand (variable traffic)
- `Clauses`: On-Demand (high read, variable write)
- `AuditLogs`: On-Demand (high write, append-only)
- `AIAnalysis`: On-Demand (variable traffic)

**Performance Optimization**:
- GSIs for efficient query patterns
- Composite sort keys for range queries
- DynamoDB Accelerator (DAX) for hot data (future enhancement)

### 7.3 CloudFront Caching

**Cache Strategy**:
- Static assets: Cache for 1 year (immutable)
- API responses: Cache for 5 minutes (with cache invalidation)
- Document thumbnails: Cache for 1 day
- User profile data: No cache (always fresh)

**Cache Key Configuration**:
- Include `Authorization` header in cache key for user-specific data
- Include `Accept-Language` header for future multi-language support

**Cache Invalidation**:
- Automatic invalidation on document update
- Manual invalidation via admin API
- Versioned URLs for static assets (no invalidation needed)

### 7.4 Cost Control Mechanisms

**Budget Alerts**:
- Daily spending threshold: $50
- Monthly budget: $1,000
- Alert at 80% and 100% of budget

**Cost Optimization Strategies**:
- S3 Intelligent-Tiering for document storage
- Lambda memory optimization (right-sizing)
- DynamoDB on-demand for variable traffic
- CloudFront caching to reduce origin requests
- Bedrock model selection (Haiku for simple tasks, Sonnet for complex)

**Cost Monitoring**:
- AWS Cost Explorer dashboards
- Per-feature cost allocation tags
- Weekly cost review and optimization

## 8. Human-in-the-Loop Architecture

### 8.1 AI Advisory Separation

**Design Principle**: AI outputs are recommendations, not decisions

**Implementation**:
- AI analysis results stored separately from user decisions
- Clear visual distinction in UI (AI badge, different color scheme)
- Explicit disclaimer on all AI-generated content

**Data Model Separation**:
```
AIAnalysisResult (AI recommendation)
  ├─ analysisId
  ├─ riskLevel (AI's assessment)
  ├─ explanation (AI's reasoning)
  ├─ confidenceScore
  └─ userApprovalStatus (null until user acts)

UserDecision (User action)
  ├─ decisionId
  ├─ analysisId (reference to AI recommendation)
  ├─ userAction (ACCEPTED | REJECTED | MODIFIED)
  ├─ userReasoning (optional user notes)
  └─ decidedAt (timestamp)
```

### 8.2 Explicit User Approval Workflow

**UI Workflow**:
```
1. Display AI Analysis
   ├─ Risk Level: HIGH (AI Assessment)
   ├─ Confidence: 87%
   ├─ Explanation: [AI reasoning]
   └─ Disclaimer: "This is AI-generated advice, not legal counsel"

2. User Decision Required
   ├─ [Accept AI Assessment]
   ├─ [Reject AI Assessment]
   ├─ [Modify and Save]
   └─ [Request Human Review]

3. Log User Decision
   ├─ Record user choice
   ├─ Timestamp decision
   ├─ Separate from AI recommendation
   └─ Update audit trail
```

**Approval States**:
- `PENDING`: AI analysis complete, awaiting user review
- `ACCEPTED`: User agrees with AI assessment
- `REJECTED`: User disagrees with AI assessment
- `MODIFIED`: User adjusted AI recommendation
- `DEFERRED`: User marked for later review

**Audit Trail Separation**:
```
AuditLog Entry 1:
  actionType: AI_ANALYSIS_GENERATED
  isAIGenerated: true
  aiRecommendation: { riskLevel: "HIGH", confidence: 0.87 }
  userDecision: null

AuditLog Entry 2:
  actionType: USER_DECISION_MADE
  isAIGenerated: false
  aiRecommendation: { riskLevel: "HIGH", confidence: 0.87 }
  userDecision: "ACCEPTED"
  decidedAt: "2026-02-15T10:30:00Z"
```

**Accountability Model**:
- AI provides recommendations (advisory role)
- User makes final decisions (decision authority)
- System logs both separately (audit compliance)
- Reports clearly distinguish AI vs. user actions

---

**Document Version**: 1.0  
**Last Updated**: February 2026  
**Status**: Production-Ready Architecture
