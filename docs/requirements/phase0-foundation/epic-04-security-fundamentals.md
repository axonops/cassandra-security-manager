# Phase 0 Epic 04: Security Fundamentals

## Overview
Core security features that protect both the application and its data must be established before any certificate operations begin. This epic implements the fundamental security controls that ensure the system itself cannot be compromised, as a compromised certificate management system would undermine all security guarantees.

## User Stories
1. **A4.1 - API Security Hardening**: Implement comprehensive API security against common attacks
2. **A4.2 - Secrets Management Foundation**: Establish secure storage for sensitive cryptographic materials

## Dependencies
- A1 (API Framework) - Security middleware requires API framework
- A2 (Authentication) - Security controls build on auth infrastructure
- A3 (Data Persistence) - Secrets storage requires database

## Success Metrics
- Zero security vulnerabilities in OWASP Top 10 categories
- All inputs validated against strict schemas
- 100% of secrets encrypted at rest
- Rate limiting preventing brute force attacks
- Security scan passing with no critical/high findings

## Technical Considerations
- Use OWASP security headers
- Implement Content Security Policy
- Use parameterized queries exclusively
- Implement security event monitoring
- Plan for future HSM integration
- Consider implementing WAF rules

## Workflow Diagram

```mermaid
graph TB
    subgraph "API Security Workflow"
        A[Incoming Request] --> B{Rate Limit Check}
        B -->|Exceeded| C[429 Too Many Requests]
        B -->|OK| D{Authentication Required?}
        D -->|Yes| E{Valid Token?}
        D -->|No| F[Process Public Endpoint]
        E -->|No| G[401 Unauthorised]
        E -->|Yes| H{Input Validation}
        H -->|Invalid| I[422 Validation Error]
        H -->|Valid| J{SQL Injection Check}
        J -->|Detected| K[400 Bad Request + Log]
        J -->|Clean| L{CORS Check}
        L -->|Invalid Origin| M[403 Forbidden]
        L -->|Valid| N[Process Request]
        N --> O[Apply Security Headers]
        O --> P[Return Response]
    end
    
    subgraph "Secrets Management Workflow"
        Q[Secret to Store] --> R{Secret Type}
        R -->|Private Key| S[Generate DEK]
        R -->|API Token| S
        R -->|Password| T[Hash with Argon2]
        S --> U[Encrypt with KEK]
        U --> V[Store in Database]
        T --> V
        V --> W[Audit Log Entry]
        
        X[Retrieve Secret] --> Y{Authorised?}
        Y -->|No| Z[403 Forbidden + Alert]
        Y -->|Yes| AA[Decrypt Secret]
        AA --> AB[Audit Access]
        AB --> AC[Return Secret]
    end
```