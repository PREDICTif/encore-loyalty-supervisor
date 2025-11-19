# Encore-Loyalty AI Feedback Integration Overview

## Executive Summary

This document outlines the integration plan between **Encore-Loyalty Main
System** (PHP-based) and the **AI Feedback Manager** system (Node.js/React +
Python FastAPI backend). The integration will enable automated AI-powered
customer feedback responses while maintaining the existing Encore-Loyalty system
architecture.

## Key Integration Principles

1. **No Direct Code Changes in Encore-Loyalty**: The Indian team will implement
   all changes in the Encore-Loyalty Main System
2. **API-First Integration**: All communication between systems via REST APIs
3. **Progressive Enhancement**: Add AI capabilities without disrupting existing
   workflows
4. **Shared Terminology**: Common naming conventions for both teams

## System Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     Encore-Loyalty Main System (PHP)                     │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐   │
│  │  Global Admin   │  │  Venue Admin     │  │  Feedback Module    │   │
│  │  - AI Settings  │  │  - Notifications │  │  - List View        │   │
│  │  - Dashboard    │  │  - Auto-Reply    │  │  - AI Responses     │   │
│  └────────┬────────┘  └────────┬─────────┘  └──────────┬──────────┘   │
│           │                    │                         │               │
└───────────┼────────────────────┼─────────────────────────┼───────────────┘
            │                    │                         │
            │              API Gateway                    │
            │                    │                         │
┌───────────▼────────────────────▼─────────────────────────▼───────────────┐
│                          encore_backend (FastAPI)                         │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐   │
│  │  Onboarding     │  │  Settings API    │  │  Feedback API       │   │
│  │  Service        │  │  (NEW)           │  │  (ENHANCED)         │   │
│  └─────────────────┘  └──────────────────┘  └─────────────────────┘   │
│                                                                          │
│  ┌─────────────────┐  ┌──────────────────┐  ┌─────────────────────┐   │
│  │  AI Analysis    │  │  Email Service   │  │  Background Jobs    │   │
│  │  (Bedrock)      │  │  (SES)           │  │  (NEW)              │   │
│  └─────────────────┘  └──────────────────┘  └─────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
            │                    │                         │
┌───────────▼────────────────────▼─────────────────────────▼───────────────┐
│                              AWS Services                                 │
│         DynamoDB              S3                    CloudWatch            │
└──────────────────────────────────────────────────────────────────────────┘
```

## Integration Components

### 1. Global Admin Integration (Encore-Loyalty)

- **AI Provisioning Page**: Enable/disable AI for venues
- **AI Dashboard**: Overview of AI-enabled venues and statistics
- **System Settings**: Global AI configuration

### 2. Venue Admin Integration (Encore-Loyalty)

- **Notification Settings**: Configure alert thresholds
- **Auto-Reply Settings**: Enable/disable automatic responses
- **AI Configuration**: Venue-specific AI behavior settings

### 3. Feedback Module Enhancement (Encore-Loyalty)

- **AI Response Display**: Show generated responses in feedback list
- **Response Actions**: Send, regenerate, or edit AI responses
- **Response History**: Track AI-generated vs manual responses

### 4. API Endpoints (encore_backend)

- **Settings Management**: Store and retrieve venue AI settings
- **Feedback Processing**: Fetch and process pending feedback
- **Response Management**: Generate, regenerate, and track responses
- **Dashboard Data**: Provide statistics and analytics

### 5. UI Components (Separate React Apps)

- **Settings Popup**: Venue AI configuration interface
- **Response Editor**: Edit and send AI-generated responses
- **Dashboard Widgets**: Analytics and statistics displays

## Data Flow

### Provisioning Flow

1. Global admin enables AI for a venue in Encore-Loyalty
2. Encore-Loyalty calls `POST /api/v1/onboard/{client_id}`
3. encore_backend migrates venue data to DynamoDB
4. Settings are initialized with defaults

### Feedback Processing Flow

1. Customer submits feedback through existing Encore-Loyalty channels
2. Background job in encore_backend polls for new feedback
3. AI analyzes feedback and generates response
4. Manager receives alert email if needed
5. Response is available in Encore-Loyalty for review/sending

### Response Sending Flow

1. Venue admin reviews AI-generated response
2. Admin can edit, regenerate, or approve response
3. On approval, response is sent to customer
4. System logs the action and updates statistics

## Security Considerations

1. **API Authentication**: Secure API keys for system-to-system communication
2. **Data Encryption**: All data in transit encrypted via HTTPS
3. **Access Control**: Role-based permissions for AI features
4. **Audit Logging**: Track all AI-related actions

## Common Terminology

| Term               | Description                               | Used By    |
| ------------------ | ----------------------------------------- | ---------- |
| Venue Provisioning | Process of enabling AI for a restaurant   | Both teams |
| AI Settings Bundle | Configuration for venue AI behavior       | Both teams |
| Response Pipeline  | Flow from feedback to AI response         | Both teams |
| Alert Threshold    | Rating/keyword triggers for notifications | Both teams |
| Auto-Send Mode     | Automatic vs manual response sending      | Both teams |

## Success Metrics

1. **Technical Metrics**

   - API response times < 2 seconds
   - AI response generation < 5 seconds
   - System uptime > 99.9%

2. **Business Metrics**
   - Response time reduction > 80%
   - Customer satisfaction improvement
   - Manager workload reduction

## Next Steps

1. Review and approve integration approach
2. Indian team implements Encore-Loyalty changes
3. Our team enhances encore_backend APIs
4. Develop UI components
5. Integration testing
6. Pilot deployment
7. Full rollout
