# Integration Architecture Diagrams

## 1. High-Level System Integration Architecture

```mermaid
graph TB
    subgraph "Encore-Loyalty Main System (PHP)"
        GA[Global Admin Module]
        VA[Venue Admin Module]
        FM[Feedback Module]
        DB[(MySQL Database)]
        CS[Cron Services]
    end

    subgraph "AI Feedback System"
        API[encore_backend API<br/>FastAPI]
        BG[Background Jobs]
        AI[AI Service<br/>AWS Bedrock]
        EMAIL[Email Service<br/>AWS SES]
    end

    subgraph "Cloud Storage"
        DDB[(DynamoDB)]
        S3[(S3 Bucket)]
    end

    subgraph "UI Components"
        SP[Settings Popup<br/>React]
        RE[Response Editor<br/>React]
        DW[Dashboard Widget<br/>React]
    end

    GA --> API
    VA --> API
    FM --> API
    CS --> API

    API --> DDB
    API --> S3
    API --> AI
    API --> EMAIL

    BG --> DB
    BG --> API

    VA -.-> SP
    FM -.-> RE
    GA -.-> DW

    style GA fill:#e1f5fe
    style VA fill:#e1f5fe
    style FM fill:#e1f5fe
    style API fill:#c8e6c9
    style AI fill:#fff3e0
    style EMAIL fill:#fff3e0
    style DDB fill:#f3e5f5
    style S3 fill:#f3e5f5
```

## 2. Venue Provisioning Flow

```mermaid
sequenceDiagram
    participant Admin as Global Admin
    participant EL as Encore-Loyalty
    participant API as encore_backend
    participant DDB as DynamoDB
    participant S3 as S3 Storage

    Admin->>EL: Enable AI for Venue
    EL->>EL: Update ai_venue_settings table
    EL->>API: POST /onboard/{client_id}
    API->>API: Validate venue data
    API->>DDB: Migrate venue data
    API->>DDB: Create default settings
    API->>S3: Initialize profile storage
    API-->>EL: Provisioning complete
    EL-->>Admin: Show success status

    Note over API,DDB: Venue is now AI-enabled
```

## 3. Feedback Processing Flow

```mermaid
flowchart LR
    subgraph "Feedback Submission"
        CF[Customer Feedback] --> EL[Encore-Loyalty]
        EL --> MDB[(MySQL)]
    end

    subgraph "AI Processing"
        CRON[Cron Job] --> MDB
        CRON --> CHK{AI Enabled?}
        CHK -->|Yes| API[encore_backend]
        CHK -->|No| END1[Skip]

        API --> AI[AWS Bedrock]
        AI --> GEN[Generate Response]
        GEN --> DDB[(DynamoDB)]
        GEN --> ALERT{Alert Needed?}
        ALERT -->|Yes| EMAIL[Send Alert Email]
        ALERT -->|No| STORE[Store Response]
        EMAIL --> STORE
    end

    subgraph "Response Delivery"
        SYNC[Sync Service] --> DDB
        SYNC --> MDB2[(MySQL)]
        AUTO{Auto-Send?} --> SYNC
        AUTO -->|Yes| SEND[Send to Customer]
        AUTO -->|No| WAIT[Wait for Approval]
        WAIT --> MANUAL[Manual Review]
        MANUAL --> SEND
    end
```

## 4. Component Interaction Diagram

```mermaid
graph TB
    subgraph "Encore-Loyalty UI Layer"
        UI1[Venue Settings Page]
        UI2[Feedback List]
        UI3[AI Dashboard]
    end

    subgraph "API Gateway"
        APIG[API Routes]
    end

    subgraph "Business Logic"
        SET[Settings Service]
        FEED[Feedback Service]
        ANAL[Analytics Service]
    end

    subgraph "External Components"
        POP1[Settings Popup<br/>iframe/modal]
        POP2[Response Editor<br/>iframe/modal]
        POP3[Dashboard Widget<br/>iframe/embed]
    end

    UI1 --> POP1
    UI2 --> POP2
    UI3 --> POP3

    POP1 --> APIG
    POP2 --> APIG
    POP3 --> APIG

    APIG --> SET
    APIG --> FEED
    APIG --> ANAL

    style UI1 fill:#e3f2fd
    style UI2 fill:#e3f2fd
    style UI3 fill:#e3f2fd
    style POP1 fill:#f3e5f5
    style POP2 fill:#f3e5f5
    style POP3 fill:#f3e5f5
```

## 5. Data Flow Diagram

```mermaid
graph TD
    subgraph "Data Sources"
        CF[Customer Feedback]
        VS[Venue Settings]
        AR[AI Responses]
    end

    subgraph "Storage Layer"
        MySQL[(MySQL)]
        DDB[(DynamoDB)]
        S3[(S3)]
    end

    subgraph "Processing Layer"
        SYNC[Data Sync Service]
        AI[AI Engine]
        QUEUE[Job Queue]
    end

    CF --> MySQL
    VS --> MySQL

    MySQL --> SYNC
    SYNC --> DDB

    DDB --> AI
    AI --> AR
    AR --> DDB
    AR --> S3

    QUEUE --> SYNC
    QUEUE --> AI
```

## 6. Security Architecture

```mermaid
graph TB
    subgraph "External Access"
        USER[Venue Admin]
        ADMIN[Global Admin]
    end

    subgraph "Authentication Layer"
        AUTH1[Encore-Loyalty Auth]
        AUTH2[API Key Auth]
    end

    subgraph "API Layer"
        ELAPI[Encore-Loyalty API]
        AIAPI[AI Backend API]
    end

    subgraph "Security Controls"
        RBAC[Role-Based Access]
        AUDIT[Audit Logging]
        ENCRYPT[Encryption]
    end

    USER --> AUTH1
    ADMIN --> AUTH1
    AUTH1 --> ELAPI

    ELAPI --> AUTH2
    AUTH2 --> AIAPI

    AIAPI --> RBAC
    AIAPI --> AUDIT
    AIAPI --> ENCRYPT

    style AUTH1 fill:#ffebee
    style AUTH2 fill:#ffebee
    style RBAC fill:#e8f5e9
    style AUDIT fill:#e8f5e9
    style ENCRYPT fill:#e8f5e9
```

## 7. Deployment Architecture

```mermaid
graph TB
    subgraph "Encore-Loyalty Infrastructure"
        ELB1[Load Balancer]
        PHP1[PHP Server 1]
        PHP2[PHP Server 2]
        MYSQL[(MySQL Cluster)]
    end

    subgraph "AI Backend Infrastructure"
        ELB2[API Gateway]
        API1[FastAPI Instance 1]
        API2[FastAPI Instance 2]
        WORKER[Background Workers]
    end

    subgraph "AWS Services"
        DDB[(DynamoDB)]
        S3[(S3)]
        SES[SES Email]
        BEDROCK[Bedrock AI]
        CW[CloudWatch]
    end

    ELB1 --> PHP1
    ELB1 --> PHP2
    PHP1 --> MYSQL
    PHP2 --> MYSQL

    PHP1 --> ELB2
    PHP2 --> ELB2

    ELB2 --> API1
    ELB2 --> API2

    API1 --> DDB
    API1 --> S3
    API1 --> SES
    API1 --> BEDROCK

    API2 --> DDB
    API2 --> S3
    API2 --> SES
    API2 --> BEDROCK

    WORKER --> DDB
    WORKER --> MYSQL

    API1 --> CW
    API2 --> CW
    WORKER --> CW
```

## 8. State Diagram for Feedback Processing

```mermaid
stateDiagram-v2
    [*] --> New: Customer submits feedback
    New --> Pending: AI processing queued
    Pending --> Processing: AI analysis started
    Processing --> Generated: Response created
    Processing --> Failed: AI error

    Generated --> AutoSendCheck: Check settings
    AutoSendCheck --> Sent: Auto-send enabled
    AutoSendCheck --> Awaiting: Manual approval required

    Awaiting --> Sent: Manager approves
    Awaiting --> Regenerating: Manager requests new response

    Regenerating --> Generated: New response created

    Failed --> Retry: Retry logic
    Retry --> Processing: Retry attempt
    Retry --> ManualRequired: Max retries exceeded

    Sent --> [*]: Process complete
    ManualRequired --> [*]: Manual intervention
```

## 9. Entity Relationship Diagram

```mermaid
erDiagram
    VENUE ||--o{ FEEDBACK : receives
    VENUE ||--|| AI_SETTINGS : has
    VENUE ||--o{ CUSTOMER : serves

    CUSTOMER ||--o{ FEEDBACK : submits
    FEEDBACK ||--o| AI_RESPONSE : generates
    AI_RESPONSE ||--o{ RESPONSE_HISTORY : tracks

    VENUE {
        int client_id PK
        string name
        boolean ai_enabled
        datetime provisioned_at
    }

    AI_SETTINGS {
        int venue_id FK
        json notification_settings
        json auto_reply_settings
        json ai_behavior_settings
    }

    FEEDBACK {
        int id PK
        int venue_id FK
        int customer_id FK
        string comment_text
        int rating
        datetime created_at
        string ai_status
    }

    AI_RESPONSE {
        string response_id PK
        int feedback_id FK
        string response_text
        string sentiment
        string alert_level
        datetime generated_at
        datetime sent_at
    }
```

## 10. UI Component Architecture

```mermaid
graph TB
    subgraph "Encore-Loyalty Pages"
        PAGE1[Global Admin Page]
        PAGE2[Venue Admin Page]
        PAGE3[Feedback List Page]
    end

    subgraph "Integration Points"
        IFRAME1[Settings IFrame]
        MODAL1[Response Modal]
        WIDGET1[Dashboard Widget]
    end

    subgraph "React Components"
        COMP1[VenueSettings.jsx]
        COMP2[ResponseEditor.jsx]
        COMP3[AIDashboard.jsx]
    end

    subgraph "API Communication"
        API[REST API]
        WS[WebSocket<br/>Real-time updates]
    end

    PAGE1 --> WIDGET1
    PAGE2 --> IFRAME1
    PAGE3 --> MODAL1

    IFRAME1 --> COMP1
    MODAL1 --> COMP2
    WIDGET1 --> COMP3

    COMP1 --> API
    COMP2 --> API
    COMP3 --> API
    COMP3 --> WS

    style PAGE1 fill:#e1f5fe
    style PAGE2 fill:#e1f5fe
    style PAGE3 fill:#e1f5fe
    style COMP1 fill:#c8e6c9
    style COMP2 fill:#c8e6c9
    style COMP3 fill:#c8e6c9
```

## Diagram Usage Guide

1. **High-Level Architecture**: Show to stakeholders for overall system
   understanding
2. **Provisioning Flow**: Use when explaining venue onboarding process
3. **Feedback Processing**: Reference for understanding the complete feedback
   lifecycle
4. **Component Interaction**: For developers implementing UI integrations
5. **Data Flow**: For database and data architects
6. **Security Architecture**: For security reviews and compliance
7. **Deployment Architecture**: For DevOps and infrastructure teams
8. **State Diagram**: For understanding feedback status transitions
9. **ERD**: For database design and relationships
10. **UI Architecture**: For frontend developers and UX designers

### Rendering Instructions

These diagrams use Mermaid syntax and can be rendered using:

- GitHub/GitLab markdown preview
- Mermaid Live Editor (https://mermaid.live)
- VS Code with Mermaid extension
- Any markdown viewer with Mermaid support
