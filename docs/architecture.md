# PhotoShare System Architecture

## High-Level Architecture

PhotoShare follows a modern, cloud-native architecture designed for scalability, reliability, and performance. The system is built using a microservices approach with clear separation of concerns.

## Architecture Principles

1. **Scalability First**: Designed to handle 100K daily active users
2. **Performance Oriented**: <200ms API response times
3. **High Availability**: 99.9% uptime SLA
4. **Cost Effective**: Optimized AWS resource usage
5. **Security by Design**: Zero-trust security model

## Component Architecture

### Frontend Layer

```
┌─────────────────────────────────────────────────┐
│                  Frontend Layer                 │
├─────────────────────────────────────────────────┤
│  React SPA (TypeScript)                         │
│  ├─ State Management: Redux Toolkit             │
│  ├─ Routing: React Router                       │
│  ├─ UI: Material-UI or Chakra UI                │
│  ├─ HTTP Client: Axios with interceptors        │
│  └─ WebSocket: Socket.io-client                 │
└─────────────────────────────────────────────────┘
```

**Hosting Strategy**: S3 + CloudFront
- Static assets served from S3
- CloudFront for global CDN
- Route 53 for DNS management
- Custom domain with SSL certificate

### API Gateway Layer

```
┌─────────────────────────────────────────────────┐
│                API Gateway Layer                │
├─────────────────────────────────────────────────┤
│  AWS API Gateway                                │
│  ├─ REST APIs for CRUD operations               │
│  ├─ WebSocket APIs for real-time features       │
│  ├─ Rate limiting and throttling                │
│  ├─ Request/response transformation              │
│  └─ Integration with AWS Cognito                │
└─────────────────────────────────────────────────┘
```

### Application Layer

```
┌─────────────────────────────────────────────────┐
│               Application Layer                 │
├─────────────────────────────────────────────────┤
│  ECS Fargate Cluster                            │
│  ├─ Express.js Application (Node.js 18+)        │
│  ├─ Auto Scaling Group (2-20 instances)         │
│  ├─ Application Load Balancer                   │
│  ├─ Health checks and monitoring                │
│  └─ Blue-green deployment support               │
│                                                 │
│  Lambda Functions                               │
│  ├─ Image processing pipeline                   │
│  ├─ Feed generation workers                     │
│  ├─ Notification handlers                       │
│  └─ Background job processors                   │
└─────────────────────────────────────────────────┘
```

### Data Layer

```
┌─────────────────────────────────────────────────┐
│                  Data Layer                     │
├─────────────────────────────────────────────────┤
│  Primary Database: RDS PostgreSQL               │
│  ├─ Multi-AZ deployment                         │
│  ├─ Read replicas (2-3 instances)               │
│  ├─ Automated backups                           │
│  └─ Connection pooling                          │
│                                                 │
│  NoSQL Database: DynamoDB                       │
│  ├─ User feeds and timelines                    │
│  ├─ Like/comment counters                       │
│  ├─ Session data                                │
│  └─ Auto-scaling enabled                        │
│                                                 │
│  Object Storage: S3                             │
│  ├─ Original photos (Standard)                  │
│  ├─ Thumbnails (Standard-IA)                    │
│  ├─ Archive storage (Glacier)                   │
│  └─ CDN integration                             │
│                                                 │
│  Cache Layer: ElastiCache Redis                 │
│  ├─ Session caching                             │
│  ├─ Feed caching                                │
│  ├─ Popular content caching                     │
│  └─ Rate limiting counters                      │
└─────────────────────────────────────────────────┘
```

## Data Flow Diagrams

### Photo Upload Flow

```
User → CloudFront → S3 → Lambda → Thumbnail Generation → Database Update → Search Index
                    ↓
               S3 Event Trigger
                    ↓
            Image Processing Pipeline
                    ↓
            Multiple Resolution Generation
                    ↓
                 Metadata Extraction
                    ↓
              Database & Search Update
```

### Feed Generation Flow

```
User Action → API Gateway → Express.js → DynamoDB → DynamoDB Streams → Lambda
                                ↓                                        ↓
                          Update Counters                        Update Search Index
                                ↓                                        ↓
                           Cache Update ← Redis ← Feed Service ← OpenSearch
```

### Real-time Notifications

```
User Action → API Gateway → Lambda → DynamoDB → WebSocket Broadcast
                              ↓
                          SNS Topic
                              ↓
                        Mobile Push (FCM)
                              ↓
                         Email (SES)
```

## Service Communication

### Synchronous Communication
- **Frontend ↔ API Gateway**: HTTPS/REST
- **API Gateway ↔ Express.js**: HTTP/JSON
- **Express.js ↔ Databases**: Connection pooling

### Asynchronous Communication
- **S3 Events**: Lambda triggers
- **DynamoDB Streams**: Change data capture
- **SQS Queues**: Background job processing
- **SNS Topics**: Event broadcasting

## Scalability Patterns

### Horizontal Scaling
- ECS services with auto-scaling
- RDS read replicas
- DynamoDB partition scaling
- Lambda concurrent executions

### Vertical Scaling
- RDS instance size adjustment
- ElastiCache cluster scaling
- ECS task resource allocation

### Caching Strategy
```
L1: Browser Cache (Static assets)
L2: CloudFront Cache (Global CDN)  
L3: API Gateway Cache (Regional)
L4: Application Cache (Redis)
L5: Database Query Cache (RDS)
```

## High Availability Design

### Multi-AZ Deployment
- ECS tasks across multiple AZs
- RDS Multi-AZ for automatic failover
- ElastiCache replication groups
- S3 cross-region replication

### Disaster Recovery
- **RTO**: 15 minutes
- **RPO**: 5 minutes
- Cross-region backup strategy
- Infrastructure as Code for quick recovery

## Performance Optimizations

### Frontend Optimizations
- Code splitting and lazy loading
- Image optimization and WebP support
- Service workers for offline capability
- Progressive Web App (PWA) features

### Backend Optimizations
- Connection pooling for databases
- Async/await for non-blocking operations
- Batch operations for bulk updates
- Streaming for large responses

### Database Optimizations
- Proper indexing strategy
- Query optimization
- Connection pooling
- Read/write separation

## Technology Stack

### Frontend
- **Framework**: React 18+ with TypeScript
- **State Management**: Redux Toolkit
- **Styling**: Styled Components or Emotion
- **Testing**: Jest + React Testing Library
- **Build Tool**: Vite or Create React App

### Backend
- **Runtime**: Node.js 18+ LTS
- **Framework**: Express.js with TypeScript
- **ORM**: Prisma or TypeORM
- **Validation**: Joi or Zod
- **Testing**: Jest + Supertest
- **Documentation**: Swagger/OpenAPI

### Infrastructure
- **IaC**: Terraform
- **Containerization**: Docker
- **Orchestration**: ECS Fargate
- **Monitoring**: CloudWatch + X-Ray
- **CI/CD**: GitHub Actions

## API Design

### RESTful Endpoints
```
GET    /api/v1/photos              # Get paginated photos
POST   /api/v1/photos              # Upload new photo
GET    /api/v1/photos/:id          # Get specific photo
PUT    /api/v1/photos/:id          # Update photo metadata
DELETE /api/v1/photos/:id          # Delete photo

GET    /api/v1/users/:id/feed      # Get user's personalized feed
POST   /api/v1/photos/:id/like     # Like a photo
DELETE /api/v1/photos/:id/like     # Unlike a photo
POST   /api/v1/photos/:id/comments # Add comment
GET    /api/v1/photos/:id/comments # Get comments

GET    /api/v1/search              # Search photos, users, tags
GET    /api/v1/trending            # Get trending content
```

### WebSocket Events
```
photo:liked         # Real-time like notifications
photo:commented     # Real-time comment notifications
user:followed       # Follow notifications
feed:updated        # Feed refresh notifications
```

## Error Handling Strategy

### HTTP Status Codes
- **200**: Success
- **201**: Created
- **400**: Bad Request
- **401**: Unauthorized
- **403**: Forbidden
- **404**: Not Found
- **429**: Too Many Requests
- **500**: Internal Server Error

### Error Response Format
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input parameters",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ],
    "requestId": "req_123456789"
  }
}
```

## Configuration Management

### Environment-based Configuration
- **Development**: Local development settings
- **Staging**: Production-like environment
- **Production**: Live environment

### Secrets Management
- AWS Secrets Manager for database credentials
- AWS Parameter Store for configuration
- Environment variables for deployment-specific settings

## Migration Strategy

### Database Migrations
- Versioned migration scripts
- Zero-downtime deployment strategies
- Rollback procedures

### API Versioning
- URL-based versioning (/api/v1/, /api/v2/)
- Backward compatibility maintenance
- Deprecation notices and timelines