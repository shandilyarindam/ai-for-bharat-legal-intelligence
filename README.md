# AI for Bharat – Legal Intelligence Platform

## 🏆 AWS AI for Bharat Hackathon Submission

An AI-powered clause-level legal document intelligence system designed to improve transparency, accountability, and accessibility in contract review for citizens, startups, and public institutions.

---

## 📌 Problem Statement

Contract review today is:
- Manual and fragmented
- Opaque in negotiation history
- Difficult for non-legal users
- Lacking accountability and defensible audit trails

Farmers, gig workers, startup founders, and government departments often sign agreements without fully understanding hidden liabilities or unfair clauses.

---

## 💡 Our Solution

This platform transforms contracts into structured, clause-level units and provides:

- 🔍 AI-powered clause extraction
- ⚠️ Risk flagging AI (Low / Medium / High / Critical)
- 🧠 Explainable Legal AI (plain-language interpretation)
- 🛡 AI-generated draft detection
- 👥 Role-based access control
- 📜 Immutable audit trail
- 📤 Compliance-ready export reports

The system ensures **AI assists decision-making, but never replaces human authority.**

---

## 🏗 Architecture Overview

Frontend:
- React.js
- TypeScript
- Tailwind CSS

Backend (AWS Serverless):
- Amazon API Gateway
- AWS Lambda
- Amazon DynamoDB
- Amazon S3
- Amazon Cognito
- AWS CloudTrail
- AWS KMS

AI Layer:
- Amazon Bedrock (LLM)
- Amazon Textract (OCR)

---

## 🔐 Security & Compliance

- TLS 1.3 encryption in transit
- AWS KMS encryption at rest
- JWT-based authentication
- Role-Based Access Control (RBAC)
- Immutable audit logging
- Human-in-the-loop safeguard
- GDPR-aligned architecture

---

## ⚙️ Key Features

### Clause Review Panel
- Clause-wise navigation
- Status indicators:
  - 🟢 Agreed
  - 🟡 Negotiation
  - 🔴 Rejected
- Comment tracking
- Timestamped actions

### AI Subsystems
- Clause segmentation
- Risk scoring
- Plain-language explanation
- AI-draft detection
- Confidence scoring

### Audit & Compliance
- Append-only logs
- Full action traceability
- Exportable compliance reports

---

## 📊 Impact Goals

- 50% reduction in contract review time
- Increased transparency for citizens
- Defensible audit trail for enterprises & government
- Democratized legal understanding

---

## 🚀 Future Enhancements

- Real-time collaborative review
- E-signature integration
- AI-powered contract drafting
- Predictive legal risk modeling

---

## 📄 Documentation

- requirements.md – Functional & Non-Functional Requirements
- design.md – System Architecture & Technical Design

---

## 👨‍💻 Built For

AWS AI for Bharat Hackathon 2026

---

## ⚖ Disclaimer

This system provides AI-assisted legal analysis and does not replace licensed legal advice.
