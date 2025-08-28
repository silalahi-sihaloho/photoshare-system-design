# Scalability & Caching Strategy

## Scalability Requirements Analysis

PhotoShare must handle:
- **Peak Load**: 10,000 concurrent users
- **Daily Growth**: 100,000 daily active users
- **Photo Uploads**: 50,000 photos per day (~0.6 photos/second average, 10+ photos/second peak)
- **API Requests**: ~1M requests per day (~12 requests/second average, 100+ requests/second peak)
- **Data Storage**: 500GB+ new photo data daily

## Horizontal Scaling Strategy

### Application Layer Scaling

#### ECS Fargate Auto Scaling
```yaml
AutoScalingConfig:
  MinCapacity: 2
  MaxCapacity: 20
  TargetCPUUtilization: 70%
  TargetMemoryUtilization: 80%
  ScaleOutCooldown: 300s
  ScaleInCooldown: 600s
  
ScalingMetrics:
  - CPUUtilization
  - MemoryUtilization
  - RequestCount
  - ResponseTime
```

#### Auto Scaling Policies
```json
{
  "ScalingPolicies": [
    {
      "PolicyName": "ScaleOut",
      "ScalingAdjustment": 2,
      "Cooldown": 300,
      "MetricAggregationType": "Average",
      "StepAdjustments": [
        {
          "MetricIntervalLowerBound": 0,
          "MetricIntervalUpperBound": 50,
          "ScalingAdjustment": 1
        },
        {
          "MetricIntervalLowerBound": 50,
          "ScalingAdjustment": 2
        }
      ]
    }
  ]
}
```

### Database Scaling

#### RDS PostgreSQL Scaling
```
Primary (Write) → Read Replica 1 (Read)
                → Read Replica 2 (Read)
                → Read Replica 3 (Read, Analytics)
```

**Read/Write Separation Strategy:**
```javascript
// Connection routing logic
const getConnection = (operation) => {
  if (operation === 'READ') {
    return readReplicaPool.getConnection();
  }
  return primaryPool.getConnection();
};

// Usage examples
const userData = await getConnection('READ').query('SELECT * FROM users');
await getConnection('WRITE').query('INSERT INTO photos...');
```

#### DynamoDB Auto Scaling
```yaml
DynamoDBAutoScaling:
  Tables:
    UserFeeds:
      ReadCapacityUnits:
        Min: 10
        Max: 1000
        TargetUtilization: 70%
      WriteCapacityUnits:
        Min: 10
        Max: 1000
        TargetUtilization: 70%
    
    PhotoLikes:
      BillingMode: ON_DEMAND  # Handles unpredictable spikes
```

## Caching Architecture

### Multi-Layer Caching Strategy

```
┌─────────────────────────────────────────────────────────┐
│                    Caching Layers                       │
├─────────────────────────────────────────────────────────┤
│ L1: Browser Cache (Static Assets) - 1 year             │
│ L2: CloudFront CDN (Global) - 1 week                   │  
│ L3: API Gateway Cache (Regional) - 5 minutes           │
│ L4: Application Cache (Redis) - Variable TTL           │
│ L5: Database Query Cache - Query-specific              │
└─────────────────────────────────────────────────────────┘
```

### Redis Caching Strategy

#### Cache Cluster Configuration
```yaml
RedisCluster:
  Engine: redis
  NodeType: cache.r6g.large
  NumCacheNodes: 3
  ReplicationGroups: 2
  AutomaticFailover: true
  MultiAZ: true
  
CacheSubnetGroup:
  Subnets:
    - subnet-1a (us-east-1a)
    - subnet-1b (us-east-1b)
    - subnet-1c (us-east-1c)
```

#### Cache Patterns Implementation

**1. User Session Cache (Write-Through)**
```javascript
class SessionCache {
  async setSession(userId, sessionData) {
    // Write to cache first
    await redis.setex(`session:${userId}`, 86400, JSON.stringify(sessionData));
    
    // Then write to database
    await db.sessions.upsert({ userId, data: sessionData });
  }
  
  async getSession(userId) {
    // Try cache first
    const cached = await redis.get(`session:${userId}`);
    if (cached) return JSON.parse(cached);
    
    // Fallback to database
    const session = await db.sessions.findOne({ userId });
    if (session) {
      // Populate cache
      await redis.setex(`session:${userId}`, 86400, JSON.stringify(session.data));
    }
    return session?.data;
  }
}
```

**2. User Feed Cache (Cache-Aside)**
```javascript
class FeedCache {
  async getUserFeed(userId, page = 1, limit = 20) {
    const cacheKey = `feed:${userId}:${page}:${limit}`;
    
    // Check cache first
    const cached = await redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }
    
    // Generate feed from database
    const feed = await this.generateFeed(userId, page, limit);
    
    // Cache for 15 minutes
    await redis.setex(cacheKey, 900, JSON.stringify(feed));
    
    return feed;
  }
  
  async invalidateUserFeed(userId) {
    const pattern = `feed:${userId}:*`;
    const keys = await redis.keys(pattern);
    if (keys.length > 0) {
      await redis.del(...keys);
    }
  }
}
```

**3. Popular Content Cache (Write-Behind)**
```javascript
class PopularContentCache {
  constructor() {
    this.writeQueue = new Map();
    this.flushInterval = setInterval(() => this.flushWrites(), 30000);
  }
  
  async incrementLike(photoId) {
    // Update cache immediately
    await redis.incr(`likes:${photoId}`);
    
    // Queue database write
    this.writeQueue.set(photoId, {
      action: 'increment',
      timestamp: Date.now()
    });
  }
  
  async flushWrites() {
    const batch = Array.from(this.writeQueue.entries());
    this.writeQueue.clear();
    
    // Batch update database
    await this.batchUpdateDatabase(batch);
  }
}
```

### Cache Invalidation Strategies

#### Time-Based TTL
```javascript
const TTL_CONFIG = {
  'user:profile': 3600,      // 1 hour
  'photo:metadata': 1800,    // 30 minutes
  'feed:timeline': 900,      // 15 minutes
  'search:results': 600,     // 10 minutes
  'trending:tags': 300       // 5 minutes
};
```

#### Event-Based Invalidation
```javascript
// DynamoDB Streams trigger
exports.invalidateCache = async (event) => {
  for (const record of event.Records) {
    if (record.eventName === 'INSERT' || record.eventName === 'MODIFY') {
      const keys = extractCacheKeys(record.dynamodb.NewImage);
      await invalidateCacheKeys(keys);
    }
  }
};

const extractCacheKeys = (item) => {
  const keys = [];
  
  if (item.userId) {
    keys.push(`feed:${item.userId.S}:*`);
    keys.push(`profile:${item.userId.S}`);
  }
  
  if (item.photoId) {
    keys.push(`photo:${item.photoId.S}`);
    keys.push(`comments:${item.photoId.S}:*`);
  }
  
  return keys;
};
```

## Load Balancing Strategy

### Application Load Balancer Configuration
```yaml
LoadBalancer:
  Type: Application
  Scheme: internet-facing
  SecurityGroups:
    - ALB-SecurityGroup
  Subnets:
    - subnet-public-1a
    - subnet-public-1b
    
TargetGroup:
  Protocol: HTTP
  Port: 3000
  HealthCheck:
    Path: /health
    IntervalSeconds: 30
    TimeoutSeconds: 5
    HealthyThresholdCount: 2
    UnhealthyThresholdCount: 3
    
ListenerRules:
  - Priority: 100
    Conditions:
      - Field: path-pattern
        Values: ['/api/*']
    Actions:
      - Type: forward
        TargetGroupArn: !Ref APITargetGroup
```

### Connection Pool Management
```javascript
// Database connection pooling
const pgPool = new Pool({
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  port: 5432,
  max: 20,                    // Maximum connections
  min: 5,                     // Minimum connections
  idle: 10000,                // Close idle connections after 10s
  connectionTimeoutMillis: 2000,
  idleTimeoutMillis: 30000,
  maxUses: 7500              // Retire connections after 7500 queries
});

// Redis connection pooling
const redisPool = new GenericPool.createPool({
  create: () => redis.createClient({
    host: process.env.REDIS_HOST,
    port: 6379,
    retry_strategy: (options) => {
      return Math.min(options.attempt * 100, 3000);
    }
  }),
  destroy: (client) => client.quit()
}, {
  max: 10,
  min: 2,
  acquireTimeoutMillis: 3000,
  createTimeoutMillis: 3000,
  destroyTimeoutMillis: 5000,
  idleTimeoutMillis: 30000
});
```

## Performance Optimization Patterns

### Database Query Optimization

#### Indexing Strategy
```sql
-- User queries optimization
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
CREATE INDEX CONCURRENTLY idx_users_username ON users(username);
CREATE INDEX CONCURRENTLY idx_users_created_at ON users(created_at);

-- Photo queries optimization  
CREATE INDEX CONCURRENTLY idx_photos_user_id_created_at ON photos(user_id, created_at DESC);
CREATE INDEX CONCURRENTLY idx_photos_tags ON photos USING GIN(tags);
CREATE INDEX CONCURRENTLY idx_photos_location ON photos USING GIST(location);

-- Feed queries optimization
CREATE INDEX CONCURRENTLY idx_follows_follower_id ON follows(follower_id);
CREATE INDEX CONCURRENTLY idx_likes_photo_id_created_at ON likes(photo_id, created_at DESC);
CREATE INDEX CONCURRENTLY idx_comments_photo_id_created_at ON comments(photo_id, created_at DESC);
```

#### Query Patterns
```javascript
// Efficient feed generation
const generateFeed = async (userId, limit = 20, offset = 0) => {
  const query = `
    SELECT p.*, u.username, u.avatar_url,
           (SELECT COUNT(*) FROM likes WHERE photo_id = p.id) as like_count,
           (SELECT COUNT(*) FROM comments WHERE photo_id = p.id) as comment_count,
           EXISTS(SELECT 1 FROM likes WHERE photo_id = p.id AND user_id = $1) as user_liked
    FROM photos p
    JOIN users u ON p.user_id = u.id
    WHERE p.user_id IN (
      SELECT followed_id FROM follows WHERE follower_id = $1
      UNION 
      SELECT $1  -- Include user's own photos
    )
    ORDER BY p.created_at DESC
    LIMIT $2 OFFSET $3
  `;
  
  return await pool.query(query, [userId, limit, offset]);
};

// Batch operations for efficiency
const batchLikeUpdate = async (likes) => {
  const values = likes.map((like, index) => 
    `($${index * 3 + 1}, $${index * 3 + 2}, $${index * 3 + 3})`
  ).join(',');
  
  const query = `
    INSERT INTO likes (user_id, photo_id, created_at) 
    VALUES ${values}
    ON CONFLICT (user_id, photo_id) DO NOTHING
  `;
  
  const params = likes.flatMap(like => [like.userId, like.photoId, like.createdAt]);
  return await pool.query(query, params);
};
```

### Application Performance Patterns

#### Async/Await Optimization
```javascript
// Parallel processing for independent operations
const getPhotoDetails = async (photoId, userId) => {
  const [photo, likes, comments, userLiked] = await Promise.all([
    getPhotoMetadata(photoId),
    getLikeCount(photoId),
    getCommentCount(photoId),
    checkUserLiked(photoId, userId)
  ]);
  
  return { photo, likes, comments, userLiked };
};

// Streaming for large responses
const getPhotoFeed = async (req, res) => {
  const userId = req.user.id;
  const stream = new JSONStream();
  
  res.writeHead(200, {
    'Content-Type': 'application/json',
    'Transfer-Encoding': 'chunked'
  });
  
  stream.pipe(res);
  
  const feedQuery = generateFeedQuery(userId);
  const cursor = await db.collection('photos').find(feedQuery).stream();
  
  cursor.on('data', (photo) => {
    stream.write(photo);
  });
  
  cursor.on('end', () => {
    stream.end();
  });
};
```

#### Memory Management
```javascript
// Object pooling for heavy operations
class ImageProcessorPool {
  constructor(size = 5) {
    this.pool = [];
    this.size = size;
    this.init();
  }
  
  init() {
    for (let i = 0; i < this.size; i++) {
      this.pool.push(new ImageProcessor());
    }
  }
  
  async acquire() {
    return this.pool.pop() || new ImageProcessor();
  }
  
  release(processor) {
    processor.reset();
    if (this.pool.length < this.size) {
      this.pool.push(processor);
    }
  }
}

// Memory-efficient file processing
const processLargeFile = async (fileStream) => {
  const transform = new Transform({
    transform(chunk, encoding, callback) {
      // Process chunk
      const processed = processChunk(chunk);
      callback(null, processed);
    }
  });
  
  return pipeline(fileStream, transform, createWriteStream('output.jpg'));
};
```

## Monitoring & Alerting for Scalability

### Key Performance Indicators
```yaml
ScalabilityMetrics:
  ResponseTime:
    - P50: < 100ms
    - P95: < 200ms
    - P99: < 500ms
    
  Throughput:
    - API Requests: > 100 RPS
    - Photo Uploads: > 10 uploads/second
    - Concurrent Users: > 10,000
    
  ResourceUtilization:
    - CPU: < 70%
    - Memory: < 80%
    - Database Connections: < 80%
    - Cache Hit Ratio: > 85%
```

### Auto Scaling Triggers
```json
{
  "CloudWatchAlarms": [
    {
      "AlarmName": "HighCPUUtilization",
      "MetricName": "CPUUtilization",
      "Threshold": 70,
      "ComparisonOperator": "GreaterThanThreshold",
      "EvaluationPeriods": 2,
      "Period": 300,
      "TreatMissingData": "notBreaching"
    },
    {
      "AlarmName": "HighMemoryUtilization", 
      "MetricName": "MemoryUtilization",
      "Threshold": 80,
      "ComparisonOperator": "GreaterThanThreshold",
      "EvaluationPeriods": 2,
      "Period": 300
    },
    {
      "AlarmName": "HighRequestCount",
      "MetricName": "RequestCount",
      "Threshold": 1000,
      "ComparisonOperator": "GreaterThanThreshold",
      "EvaluationPeriods": 1,
      "Period": 60
    }
  ]
}
```

## Capacity Planning

### Growth Projections
```
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ Metric      │ Current     │ 6 Months    │ 1 Year      │ 2 Years     │
├─────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ DAU         │ 100K        │ 250K        │ 500K        │ 1M          │
│ Photos/Day  │ 50K         │ 125K        │ 250K        │ 500K        │
│ Storage     │ 500GB/day   │ 1.25TB/day  │ 2.5TB/day   │ 5TB/day     │
│ API Calls   │ 1M/day      │ 2.5M/day    │ 5M/day      │ 10M/day     │
│ Concurrent  │ 10K         │ 25K         │ 50K         │ 100K        │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

### Resource Scaling Plan
```yaml
ScalingPlan:
  ECSServices:
    Current: 4 tasks (2 vCPU, 4GB each)
    6Months: 10 tasks (2 vCPU, 4GB each)
    1Year: 20 tasks (4 vCPU, 8GB each)
    2Years: 50 tasks (4 vCPU, 8GB each)
    
  RDSInstances:
    Current: db.r6g.large (2 vCPU, 16GB)
    6Months: db.r6g.xlarge (4 vCPU, 32GB)
    1Year: db.r6g.2xlarge (8 vCPU, 64GB)
    2Years: db.r6g.4xlarge (16 vCPU, 128GB)
    
  CacheCluster:
    Current: 3 nodes cache.r6g.large
    6Months: 6 nodes cache.r6g.large
    1Year: 9 nodes cache.r6g.xlarge
    2Years: 12 nodes cache.r6g.2xlarge
```

### Cost Implications
```
┌─────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ Period      │ Compute     │ Storage     │ Database    │ Total       │
├─────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Current     │ $400        │ $300        │ $450        │ $1,150      │
│ 6 Months    │ $1,000      │ $750        │ $900        │ $2,650      │
│ 1 Year      │ $2,000      │ $1,500      │ $1,800      │ $5,300      │
│ 2 Years     │ $5,000      │ $3,000      │ $3,600      │ $11,600     │
└─────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```