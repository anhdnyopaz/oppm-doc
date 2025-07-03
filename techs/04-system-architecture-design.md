# Thiết Kế Kiến Trúc Hệ Thống (System Architecture Design)

## Tổng Quan Kiến Trúc

### High-Level Architecture
```
[Frontend Web/Mobile] ↔ [API Gateway] ↔ [Microservices] ↔ [Databases]
                                     ↓
                              [External Services]
```

### Architecture Principles
1. **Microservices-based** - Tách biệt services độc lập
2. **API-First** - RESTful APIs với GraphQL cho complex queries
3. **Event-Driven** - Message queues cho async processing
4. **Cloud-Native** - Container-based deployment
5. **Security-First** - Zero-trust architecture

## Microservices Architecture

### Core Services

#### 1. User Service
```go
// User management và authentication
type UserService struct {
    Repository UserRepository
    JWT        JWTManager
    Email      EmailService
}

// Core functions:
// - User registration/login
// - Profile management  
// - Role-based permissions
// - OAuth integration
```

#### 2. Project Service
```go
// Project lifecycle management
type ProjectService struct {
    Repository ProjectRepository
    Template   TemplateService
    Validator  ProjectValidator
}

// Core functions:
// - Project CRUD operations
// - Template management
// - Project settings
// - Member management
```

#### 3. Task Service
```go
// Task và work management
type TaskService struct {
    Repository TaskRepository
    Scheduler  SchedulingEngine
    Dependency DependencyManager
}

// Core functions:
// - Task CRUD operations
// - Dependencies management
// - Time tracking
// - Progress monitoring
```

#### 4. Communication Service
```go
// Real-time communication
type CommunicationService struct {
    WebSocket WebSocketManager
    Chat      ChatRepository
    Notify    NotificationService
}

// Core functions:
// - Real-time messaging
// - File sharing
// - Notifications
// - Video call integration
```

#### 5. Reporting Service
```go
// Analytics và reporting
type ReportingService struct {
    Analytics AnalyticsEngine
    Export    ExportService
    Dashboard DashboardService
}

// Core functions:
// - Dashboard generation
// - Report creation
// - Data analytics
// - Export functionality
```

## Database Design

### Primary Database (PostgreSQL)

#### Core Tables
```sql
-- User Management
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    avatar_url TEXT,
    role_id UUID REFERENCES roles(id),
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Project Management
CREATE TABLE projects (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    start_date DATE,
    end_date DATE,
    budget DECIMAL(15,2),
    status VARCHAR(50) DEFAULT 'planning',
    template_id UUID REFERENCES project_templates(id),
    owner_id UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Task Management
CREATE TABLE tasks (
    id UUID PRIMARY KEY,
    project_id UUID REFERENCES projects(id),
    parent_task_id UUID REFERENCES tasks(id),
    title VARCHAR(255) NOT NULL,
    description TEXT,
    status VARCHAR(50) DEFAULT 'todo',
    priority VARCHAR(20) DEFAULT 'medium',
    start_date TIMESTAMP,
    due_date TIMESTAMP,
    estimated_hours INTEGER,
    actual_hours INTEGER,
    completion_percentage INTEGER DEFAULT 0,
    assignee_id UUID REFERENCES users(id),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Communication
CREATE TABLE messages (
    id UUID PRIMARY KEY,
    project_id UUID REFERENCES projects(id),
    channel_id UUID REFERENCES channels(id),
    sender_id UUID REFERENCES users(id),
    content TEXT NOT NULL,
    message_type VARCHAR(20) DEFAULT 'text',
    reply_to_id UUID REFERENCES messages(id),
    created_at TIMESTAMP DEFAULT NOW()
);
```

### Caching Strategy (Redis)
```go
// Session management
"session:{user_id}" -> session_data

// Project caching
"project:{project_id}" -> project_data
"project_members:{project_id}" -> member_list

// Real-time data
"active_users" -> set of online users
"typing_users:{channel_id}" -> set of typing users
```

## API Design

### RESTful Endpoints

#### Authentication
```
POST /api/auth/register
POST /api/auth/login
POST /api/auth/logout
POST /api/auth/refresh
GET  /api/auth/profile
```

#### Projects
```
GET    /api/projects
POST   /api/projects
GET    /api/projects/{id}
PUT    /api/projects/{id}
DELETE /api/projects/{id}
GET    /api/projects/{id}/members
POST   /api/projects/{id}/members
```

#### Tasks
```
GET    /api/projects/{project_id}/tasks
POST   /api/projects/{project_id}/tasks
GET    /api/tasks/{id}
PUT    /api/tasks/{id}
DELETE /api/tasks/{id}
POST   /api/tasks/{id}/comments
```

### WebSocket Events
```javascript
// Client → Server
{
  "type": "join_project",
  "payload": { "project_id": "uuid" }
}

{
  "type": "send_message", 
  "payload": { "channel_id": "uuid", "content": "message" }
}

// Server → Client
{
  "type": "new_message",
  "payload": { "message": {...}, "sender": {...} }
}

{
  "type": "task_updated",
  "payload": { "task": {...}, "updated_by": {...} }
}
```

## Frontend Architecture

### React/Next.js Structure
```
src/
├── components/
│   ├── common/          # Shared components
│   ├── auth/           # Authentication components
│   ├── project/        # Project-specific components
│   ├── task/           # Task management components
│   └── chat/           # Communication components
├── pages/
│   ├── auth/           # Auth pages
│   ├── dashboard/      # Main dashboard
│   ├── projects/       # Project pages
│   └── settings/       # Settings pages
├── hooks/              # Custom React hooks
├── services/           # API services
├── utils/              # Utility functions
└── store/              # State management
```

### State Management (Zustand)
```typescript
// Auth store
interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (credentials: LoginData) => Promise<void>;
  logout: () => void;
}

// Project store
interface ProjectState {
  currentProject: Project | null;
  projects: Project[];
  setCurrentProject: (project: Project) => void;
  updateProject: (project: Project) => void;
}

// Real-time store
interface RealtimeState {
  socket: Socket | null;
  connected: boolean;
  activeUsers: User[];
  connect: () => void;
  disconnect: () => void;
}
```

## Security Architecture

### Authentication Flow
```
1. User login → JWT token issued
2. Token stored in httpOnly cookie
3. API requests include token in Authorization header
4. Server validates token on each request
5. Refresh token for session extension
```

### Security Measures
```go
// Rate limiting
middleware.RateLimit(requests: 100, window: "1m")

// CORS configuration
middleware.CORS(origins: ["https://app.domain.com"])

// Input validation
middleware.ValidateJSON(schema: requestSchema)

// SQL injection protection
db.QueryRow("SELECT * FROM users WHERE id = $1", userID)
```

## Deployment Architecture

### Container Strategy
```dockerfile
# Multi-stage build cho production
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/node_modules ./node_modules
COPY . .
EXPOSE 3000
CMD ["npm", "start"]
```

### Kubernetes Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: project-management-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: pm-api
  template:
    metadata:
      labels:
        app: pm-api
    spec:
      containers:
      - name: api
        image: pm-api:latest
        ports:
        - containerPort: 8080
        env:
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
```

## Monitoring và Observability

### Health Checks
```go
// Health check endpoint
func HealthCheck(c *gin.Context) {
    status := map[string]interface{}{
        "status": "healthy",
        "timestamp": time.Now(),
        "version": os.Getenv("APP_VERSION"),
        "database": checkDatabase(),
        "redis": checkRedis(),
    }
    c.JSON(200, status)
}
```

### Metrics Collection
```go
// Prometheus metrics
var (
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Help: "HTTP request duration",
        },
        []string{"method", "endpoint", "status"},
    )
    
    activeConnections = prometheus.NewGauge(
        prometheus.GaugeOpts{
            Name: "websocket_active_connections",
            Help: "Number of active WebSocket connections",
        },
    )
)
```

### Logging Strategy
```go
// Structured logging với logrus
log.WithFields(log.Fields{
    "user_id":    userID,
    "project_id": projectID,
    "action":     "create_task",
    "duration":   duration,
}).Info("Task created successfully")
```

## Performance Optimization

### Database Optimization
```sql
-- Indexes cho performance
CREATE INDEX idx_tasks_project_id ON tasks(project_id);
CREATE INDEX idx_tasks_assignee_id ON tasks(assignee_id);
CREATE INDEX idx_messages_channel_id ON messages(channel_id);

-- Partitioning cho large tables
CREATE TABLE messages_2024 PARTITION OF messages
FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

### Caching Strategy
```go
// Multi-level caching
func GetProject(id string) (*Project, error) {
    // Level 1: In-memory cache
    if project := memoryCache.Get(id); project != nil {
        return project, nil
    }
    
    // Level 2: Redis cache
    if project := redisCache.Get(id); project != nil {
        memoryCache.Set(id, project)
        return project, nil
    }
    
    // Level 3: Database
    project, err := db.GetProject(id)
    if err != nil {
        return nil, err
    }
    
    redisCache.Set(id, project)
    memoryCache.Set(id, project)
    return project, nil
}
```

## Scalability Considerations

### Horizontal Scaling
- **Stateless services** - Cho easy scaling
- **Load balancer** - Distribute traffic
- **Database sharding** - Based on project_id
- **CDN** - Cho static assets

### Auto-scaling Rules
```yaml
# Kubernetes HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: pm-api-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: project-management-api
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## Disaster Recovery

### Backup Strategy
- **Database backups** - Daily snapshots
- **File storage backups** - Geo-replicated
- **Configuration backups** - Version controlled
- **Recovery testing** - Monthly drills

### High Availability
- **Multi-region deployment** - Geographic redundancy
- **Circuit breakers** - Prevent cascade failures
- **Graceful degradation** - Core features availability
- **Failover automation** - Minimize downtime

## Development Workflow

### CI/CD Pipeline
```yaml
# GitHub Actions workflow
name: Deploy to Production
on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        run: |
          go test ./...
          npm test
  
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to K8s
        run: |
          kubectl apply -f k8s/
          kubectl rollout status deployment/pm-api
```

### Code Quality
- **Linting** - ESLint, golangci-lint
- **Testing** - Unit tests với >80% coverage
- **Code review** - Mandatory PR reviews
- **Security scanning** - SAST/DAST tools

## Conclusion

Kiến trúc này được thiết kế để:
1. **Scalable** - Hỗ trợ millions of users
2. **Maintainable** - Clean code và separation of concerns  
3. **Secure** - Defense in depth strategy
4. **Observable** - Comprehensive monitoring
5. **Resilient** - High availability và disaster recovery

Success factors:
- Team expertise trong microservices
- Proper DevOps practices
- Continuous performance monitoring
- Regular security audits 