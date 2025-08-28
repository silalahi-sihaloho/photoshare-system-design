# Security & Compliance

## Security Framework

PhotoShare implements a comprehensive security strategy based on the AWS Well-Architected Security Pillar and industry best practices for handling user-generated content and personal data.

## Authentication & Authorization

### AWS Cognito Integration

#### User Pool Configuration
```json
{
  "UserPool": {
    "PoolName": "PhotoShareUsers",
    "Policies": {
      "PasswordPolicy": {
        "MinimumLength": 12,
        "RequireUppercase": true,
        "RequireLowercase": true,
        "RequireNumbers": true,
        "RequireSymbols": true,
        "TemporaryPasswordValidityDays": 1
      }
    },
    "MfaConfiguration": "OPTIONAL",
    "EnabledMfas": ["SMS_MFA", "SOFTWARE_TOKEN_MFA"],
    "AccountRecoverySetting": {
      "RecoveryMechanisms": [
        {"Name": "verified_email", "Priority": 1},
        {"Name": "verified_phone_number", "Priority": 2}
      ]
    },
    "AutoVerifiedAttributes": ["email"],
    "AliasAttributes": ["email", "preferred_username"],
    "DeviceConfiguration": {
      "ChallengeRequiredOnNewDevice": true,
      "DeviceOnlyRememberedOnUserPrompt": false
    }
  }
}
```

#### JWT Token Management
```javascript
// Token validation middleware
const validateJWT = async (req, res, next) => {
  try {
    const token = req.headers.authorization?.replace('Bearer ', '');
    if (!token) {
      return res.status(401).json({ error: 'No token provided' });
    }

    // Verify JWT signature and expiration
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    
    // Check token blacklist (Redis)
    const isBlacklisted = await redis.get(`blacklist:${token}`);
    if (isBlacklisted) {
      return res.status(401).json({ error: 'Token revoked' });
    }

    // Validate user session
    const session = await redis.get(`session:${decoded.userId}`);
    if (!session) {
      return res.status(401).json({ error: 'Session expired' });
    }

    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Role-based access control
const requireRole = (roles) => {
  return (req, res, next) => {
    if (!roles.includes(req.user.role)) {
      return res.status(403).json({ error: 'Insufficient permissions' });
    }
    next();
  };
};

// Resource ownership validation
const validatePhotoOwnership = async (req, res, next) => {
  const photoId = req.params.photoId;
  const userId = req.user.id;
  
  const photo = await db.photos.findOne({ id: photoId });
  if (!photo) {
    return res.status(404).json({ error: 'Photo not found' });
  }
  
  if (photo.userId !== userId && req.user.role !== 'admin') {
    return res.status(403).json({ error: 'Access denied' });
  }
  
  req.photo = photo;
  next();
};
```

### Multi-Factor Authentication
```javascript
// MFA Setup
const setupMFA = async (userId, method) => {
  switch (method) {
    case 'TOTP':
      const secret = speakeasy.generateSecret({
        name: `PhotoShare (${userId})`,
        issuer: 'PhotoShare'
      });
      
      // Store secret securely
      await secretsManager.putSecretValue({
        SecretId: `mfa-secret-${userId}`,
        SecretString: secret.base32
      });
      
      return {
        qrCode: qrcode.imageSync(secret.otpauth_url, { type: 'png' }),
        backupCodes: generateBackupCodes(userId)
      };
      
    case 'SMS':
      // Send verification code via SNS
      const code = generateVerificationCode();
      await sns.publish({
        PhoneNumber: user.phoneNumber,
        Message: `PhotoShare verification code: ${code}`
      });
      
      await redis.setex(`mfa-code:${userId}`, 300, code);
      break;
  }
};

// MFA Verification
const verifyMFA = async (userId, code, method) => {
  switch (method) {
    case 'TOTP':
      const secret = await secretsManager.getSecretValue({
        SecretId: `mfa-secret-${userId}`
      });
      
      return speakeasy.totp.verify({
        secret: secret.SecretString,
        encoding: 'base32',
        token: code,
        window: 2
      });
      
    case 'SMS':
      const storedCode = await redis.get(`mfa-code:${userId}`);
      return storedCode === code;
  }
};
```

## Data Encryption

### Encryption at Rest

#### S3 Encryption Configuration
```yaml
S3BucketEncryption:
  PhotosBucket:
    BucketEncryption:
      ServerSideEncryptionConfiguration:
        - ServerSideEncryptionByDefault:
            SSEAlgorithm: aws:kms
            KMSMasterKeyID: !Ref PhotosKMSKey
          BucketKeyEnabled: true
        
  PublicAccessBlockConfiguration:
    BlockPublicAcls: true
    BlockPublicPolicy: true
    IgnorePublicAcls: true
    RestrictPublicBuckets: true

PhotosKMSKey:
  Type: AWS::KMS::Key
  Properties:
    Description: KMS key for PhotoShare photo encryption
    KeyPolicy:
      Statement:
        - Effect: Allow
          Principal:
            AWS: !Sub "arn:aws:iam::${AWS::AccountId}:root"
          Action: "kms:*"
          Resource: "*"
        - Effect: Allow
          Principal:
            Service: s3.amazonaws.com
          Action:
            - kms:Decrypt
            - kms:GenerateDataKey
          Resource: "*"
```

#### RDS Encryption
```yaml
RDSEncryption:
  DBInstance:
    StorageEncrypted: true
    KmsKeyId: !Ref DatabaseKMSKey
    
  DBCluster:
    StorageEncrypted: true
    KmsKeyId: !Ref DatabaseKMSKey
    
DatabaseKMSKey:
  Type: AWS::KMS::Key
  Properties:
    Description: KMS key for PhotoShare database encryption
    KeyRotationEnabled: true
```

#### Application-Level Encryption
```javascript
// Sensitive data encryption
const crypto = require('crypto');
const algorithm = 'aes-256-gcm';

class DataEncryption {
  constructor() {
    this.encryptionKey = process.env.ENCRYPTION_KEY;
  }
  
  encrypt(text) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipher(algorithm, this.encryptionKey);
    cipher.setAAD(Buffer.from('PhotoShare', 'utf8'));
    
    let encrypted = cipher.update(text, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const tag = cipher.getAuthTag();
    
    return {
      iv: iv.toString('hex'),
      encryptedData: encrypted,
      tag: tag.toString('hex')
    };
  }
  
  decrypt(encryptedObj) {
    const decipher = crypto.createDecipher(algorithm, this.encryptionKey);
    decipher.setAAD(Buffer.from('PhotoShare', 'utf8'));
    decipher.setAuthTag(Buffer.from(encryptedObj.tag, 'hex'));
    
    let decrypted = decipher.update(encryptedObj.encryptedData, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
}

// Usage for PII data
const encryption = new DataEncryption();

// Encrypt user email before storage
const encryptedEmail = encryption.encrypt(user.email);
await db.users.update(userId, { 
  encryptedEmail: JSON.stringify(encryptedEmail) 
});

// Decrypt when needed
const storedData = await db.users.findOne(userId);
const decryptedEmail = encryption.decrypt(JSON.parse(storedData.encryptedEmail));
```

### Encryption in Transit

#### TLS Configuration
```yaml
# Application Load Balancer
ALBListenerHTTPS:
  Type: AWS::ElasticLoadBalancingV2::Listener
  Properties:
    DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref TargetGroup
    LoadBalancerArn: !Ref ApplicationLoadBalancer
    Port: 443
    Protocol: HTTPS
    SslPolicy: ELBSecurityPolicy-TLS-1-2-2017-01
    Certificates:
      - CertificateArn: !Ref SSLCertificate

# CloudFront Distribution
CloudFrontDistribution:
  Type: AWS::CloudFront::Distribution
  Properties:
    DistributionConfig:
      ViewerProtocolPolicy: redirect-to-https
      MinimumProtocolVersion: TLSv1.2_2021
      SslSupportMethod: sni-only
```

#### Database Connection Security
```javascript
// PostgreSQL SSL connection
const pgConfig = {
  host: process.env.DB_HOST,
  database: process.env.DB_NAME,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  port: 5432,
  ssl: {
    require: true,
    rejectUnauthorized: true,
    ca: fs.readFileSync('/app/certs/rds-ca-2019-root.pem')
  }
};

// Redis TLS connection
const redisConfig = {
  host: process.env.REDIS_HOST,
  port: 6380,
  tls: {
    rejectUnauthorized: true,
    checkServerIdentity: () => undefined
  }
};
```

## Network Security

### VPC Architecture
```yaml
VPCConfiguration:
  VPC:
    CidrBlock: 10.0.0.0/16
    EnableDnsHostnames: true
    EnableDnsSupport: true
    
  PublicSubnets:
    - CidrBlock: 10.0.1.0/24  # us-east-1a
    - CidrBlock: 10.0.2.0/24  # us-east-1b
    - CidrBlock: 10.0.3.0/24  # us-east-1c
    
  PrivateSubnets:
    - CidrBlock: 10.0.11.0/24 # us-east-1a
    - CidrBlock: 10.0.12.0/24 # us-east-1b  
    - CidrBlock: 10.0.13.0/24 # us-east-1c
    
  DatabaseSubnets:
    - CidrBlock: 10.0.21.0/24 # us-east-1a
    - CidrBlock: 10.0.22.0/24 # us-east-1b
    - CidrBlock: 10.0.23.0/24 # us-east-1c
```

### Security Groups
```yaml
SecurityGroups:
  ALBSecurityGroup:
    GroupDescription: Security group for Application Load Balancer
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
        
  ECSSecurityGroup:
    GroupDescription: Security group for ECS tasks
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 3000
        ToPort: 3000
        SourceSecurityGroupId: !Ref ALBSecurityGroup
        
  DatabaseSecurityGroup:
    GroupDescription: Security group for RDS instances
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 5432
        ToPort: 5432
        SourceSecurityGroupId: !Ref ECSSecurityGroup
        
  CacheSecurityGroup:
    GroupDescription: Security group for ElastiCache
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 6379
        ToPort: 6379
        SourceSecurityGroupId: !Ref ECSSecurityGroup
```

### WAF Configuration
```yaml
WebACL:
  Type: AWS::WAFv2::WebACL
  Properties:
    DefaultAction:
      Allow: {}
    Rules:
      - Name: AWSManagedRulesCommonRuleSet
        Priority: 1
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesCommonRuleSet
        OverrideAction:
          None: {}
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: CommonRuleSetMetric
          
      - Name: RateLimitRule
        Priority: 2
        Statement:
          RateBasedStatement:
            Limit: 2000
            AggregateKeyType: IP
        Action:
          Block: {}
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: RateLimitMetric
          
      - Name: IPReputationList
        Priority: 3
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesAmazonIpReputationList
        OverrideAction:
          None: {}
```

## IAM Security

### Role-Based Access Control
```yaml
ECSTaskRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: ecs-tasks.amazonaws.com
          Action: sts:AssumeRole
    Policies:
      - PolicyName: PhotoShareTaskPolicy
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action:
                - s3:GetObject
                - s3:PutObject
                - s3:DeleteObject
              Resource: !Sub "${PhotosBucket}/*"
            - Effect: Allow
              Action:
                - kms:Decrypt
                - kms:GenerateDataKey
              Resource: !Ref PhotosKMSKey
              
LambdaExecutionRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Statement:
        - Effect: Allow
          Principal:
            Service: lambda.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
    Policies:
      - PolicyName: ImageProcessingPolicy
        PolicyDocument:
          Statement:
            - Effect: Allow
              Action:
                - s3:GetObject
                - s3:PutObject
              Resource: 
                - !Sub "${PhotosBucket}/*"
                - !Sub "${ThumbnailsBucket}/*"
```

### Secrets Management
```javascript
// AWS Secrets Manager integration
class SecretsManager {
  constructor() {
    this.client = new AWS.SecretsManager({
      region: process.env.AWS_REGION
    });
  }
  
  async getSecret(secretId) {
    try {
      const result = await this.client.getSecretValue({
        SecretId: secretId
      }).promise();
      
      return JSON.parse(result.SecretString);
    } catch (error) {
      console.error(`Failed to retrieve secret ${secretId}:`, error);
      throw error;
    }
  }
  
  async rotateSecret(secretId, newValue) {
    await this.client.putSecretValue({
      SecretId: secretId,
      SecretString: JSON.stringify(newValue),
      VersionStage: 'AWSPENDING'
    }).promise();
  }
}

// Application startup
const secrets = new SecretsManager();
const dbCredentials = await secrets.getSecret('photoshare/database');
const jwtSecret = await secrets.getSecret('photoshare/jwt-secret');
```

## Input Validation & Sanitization

### API Input Validation
```javascript
const Joi = require('joi');

// Photo upload validation
const photoUploadSchema = Joi.object({
  title: Joi.string().max(100).pattern(/^[a-zA-Z0-9\s\-_.,!?]+$/).required(),
  description: Joi.string().max(500).optional(),
  tags: Joi.array().items(
    Joi.string().max(30).pattern(/^[a-zA-Z0-9_]+$/)
  ).max(10),
  isPublic: Joi.boolean().default(true),
  location: Joi.object({
    latitude: Joi.number().min(-90).max(90),
    longitude: Joi.number().min(-180).max(180)
  }).optional()
});

// User registration validation  
const userRegistrationSchema = Joi.object({
  username: Joi.string()
    .alphanum()
    .min(3)
    .max(30)
    .pattern(/^[a-zA-Z0-9_]+$/)
    .required(),
  email: Joi.string().email().required(),
  password: Joi.string()
    .min(12)
    .pattern(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]/)
    .required(),
  fullName: Joi.string().max(100).pattern(/^[a-zA-Z\s]+$/).required()
});

// Validation middleware
const validateInput = (schema) => {
  return (req, res, next) => {
    const { error, value } = schema.validate(req.body);
    if (error) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.details.map(d => ({
          field: d.path.join('.'),
          message: d.message
        }))
      });
    }
    req.validatedData = value;
    next();
  };
};
```

### File Upload Security
```javascript
const multer = require('multer');
const { FileTypeValidator } = require('./validators');

// File upload configuration
const uploadConfig = multer({
  limits: {
    fileSize: 10 * 1024 * 1024, // 10MB limit
    files: 1
  },
  fileFilter: (req, file, cb) => {
    // Check file type
    const allowedTypes = ['image/jpeg', 'image/png', 'image/webp'];
    if (!allowedTypes.includes(file.mimetype)) {
      return cb(new Error('Invalid file type'), false);
    }
    
    // Check file extension
    const allowedExtensions = ['.jpg', '.jpeg', '.png', '.webp'];
    const ext = path.extname(file.originalname).toLowerCase();
    if (!allowedExtensions.includes(ext)) {
      return cb(new Error('Invalid file extension'), false);
    }
    
    cb(null, true);
  }
});

// Additional file validation
const validateUploadedFile = async (req, res, next) => {
  if (!req.file) {
    return res.status(400).json({ error: 'No file uploaded' });
  }
  
  try {
    // Validate file content matches extension
    const isValid = await FileTypeValidator.validate(req.file.buffer);
    if (!isValid) {
      return res.status(400).json({ error: 'File content does not match extension' });
    }
    
    // Scan for malware (using ClamAV or similar)
    const isSafe = await scanForMalware(req.file.buffer);
    if (!isSafe) {
      return res.status(400).json({ error: 'File failed security scan' });
    }
    
    next();
  } catch (error) {
    return res.status(500).json({ error: 'File validation failed' });
  }
};
```

## Rate Limiting & DDoS Protection

### API Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

// Different rate limits for different endpoints
const createRateLimit = (windowMs, max, message) => {
  return rateLimit({
    store: new RedisStore({
      client: redisClient,
      prefix: 'rl:'
    }),
    windowMs,
    max,
    message: { error: message },
    standardHeaders: true,
    legacyHeaders: false,
    keyGenerator: (req) => {
      // Rate limit by user ID if authenticated, otherwise by IP
      return req.user?.id || req.ip;
    }
  });
};

// Apply different rate limits
app.use('/api/auth/login', createRateLimit(15 * 60 * 1000, 5, 'Too many login attempts'));
app.use('/api/photos/upload', createRateLimit(60 * 1000, 10, 'Upload rate limit exceeded'));
app.use('/api/', createRateLimit(60 * 1000, 100, 'API rate limit exceeded'));

// Advanced rate limiting with Redis
class AdvancedRateLimit {
  constructor() {
    this.redis = redisClient;
  }
  
  async checkRateLimit(key, limit, window) {
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, window);
    }
    
    return {
      allowed: current <= limit,
      remaining: Math.max(0, limit - current),
      resetTime: await this.redis.ttl(key)
    };
  }
  
  async implementSlidingWindow(userId, limit, windowSize) {
    const now = Date.now();
    const key = `sliding:${userId}`;
    
    // Remove old entries
    await this.redis.zremrangebyscore(key, 0, now - windowSize);
    
    // Count current requests
    const count = await this.redis.zcard(key);
    
    if (count < limit) {
      // Add current request
      await this.redis.zadd(key, now, now);
      await this.redis.expire(key, Math.ceil(windowSize / 1000));
      return { allowed: true, remaining: limit - count - 1 };
    }
    
    return { allowed: false, remaining: 0 };
  }
}
```

## Security Monitoring & Incident Response

### Security Event Logging
```javascript
const winston = require('winston');
const CloudWatchTransport = require('winston-cloudwatch');

// Security-focused logger
const securityLogger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new CloudWatchTransport({
      logGroupName: '/aws/lambda/photoshare-security',
      logStreamName: 'security-events',
      awsOptions: {
        region: process.env.AWS_REGION
      }
    })
  ]
});

// Security event types
const SecurityEvents = {
  FAILED_LOGIN: 'FAILED_LOGIN',
  SUSPICIOUS_ACTIVITY: 'SUSPICIOUS_ACTIVITY',
  UNAUTHORIZED_ACCESS: 'UNAUTHORIZED_ACCESS',
  DATA_BREACH_ATTEMPT: 'DATA_BREACH_ATTEMPT',
  MALWARE_DETECTED: 'MALWARE_DETECTED'
};

// Log security events
const logSecurityEvent = (eventType, details, req) => {
  securityLogger.warn({
    eventType,
    timestamp: new Date().toISOString(),
    userAgent: req.get('user-agent'),
    ip: req.ip,
    userId: req.user?.id,
    details,
    requestId: req.requestId
  });
};

// Usage examples
app.use('/api/auth/login', async (req, res, next) => {
  try {
    const { email, password } = req.body;
    const user = await authenticate(email, password);
    
    if (!user) {
      logSecurityEvent(SecurityEvents.FAILED_LOGIN, {
        email,
        reason: 'Invalid credentials'
      }, req);
      
      return res.status(401).json({ error: 'Invalid credentials' });
    }
    
    // Successful login logic...
  } catch (error) {
    next(error);
  }
});
```

### Automated Threat Detection
```yaml
# CloudWatch Alarms for security events
SecurityAlarms:
  FailedLoginAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: HighFailedLoginAttempts
      MetricName: FailedLogins
      Namespace: PhotoShare/Security
      Statistic: Sum
      Period: 300
      EvaluationPeriods: 1
      Threshold: 10
      ComparisonOperator: GreaterThanThreshold
      AlarmActions:
        - !Ref SecurityNotificationTopic
        
  SuspiciousActivityAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: SuspiciousUserActivity
      MetricName: SuspiciousActivity
      Namespace: PhotoShare/Security
      Statistic: Sum
      Period: 600
      EvaluationPeriods: 2
      Threshold: 5
      ComparisonOperator: GreaterThanThreshold
```

## Compliance & Privacy

### GDPR Compliance
```javascript
// Data retention and deletion
class GDPRCompliance {
  async handleDataDeletionRequest(userId) {
    // 1. Delete user profile and posts
    await db.users.delete(userId);
    await db.photos.deleteMany({ userId });
    await db.comments.deleteMany({ userId });
    await db.likes.deleteMany({ userId });
    
    // 2. Delete from cache
    await redis.del(`user:${userId}:*`);
    
    // 3. Delete files from S3
    const userPhotos = await s3.listObjectsV2({
      Bucket: process.env.PHOTOS_BUCKET,
      Prefix: `users/${userId}/`
    }).promise();
    
    if (userPhotos.Contents.length > 0) {
      await s3.deleteObjects({
        Bucket: process.env.PHOTOS_BUCKET,
        Delete: {
          Objects: userPhotos.Contents.map(obj => ({ Key: obj.Key }))
        }
      }).promise();
    }
    
    // 4. Anonymize logs
    await this.anonymizeLogs(userId);
    
    // 5. Update search indices
    await opensearch.deleteByQuery({
      index: 'photos',
      body: {
        query: { term: { userId } }
      }
    });
  }
  
  async exportUserData(userId) {
    // Collect all user data
    const userData = {
      profile: await db.users.findOne(userId),
      photos: await db.photos.findMany({ userId }),
      comments: await db.comments.findMany({ userId }),
      likes: await db.likes.findMany({ userId }),
      followers: await db.follows.findMany({ followerId: userId }),
      following: await db.follows.findMany({ followedId: userId })
    };
    
    // Generate secure download link
    const exportData = JSON.stringify(userData, null, 2);
    const exportKey = `exports/${userId}-${Date.now()}.json`;
    
    await s3.putObject({
      Bucket: process.env.EXPORTS_BUCKET,
      Key: exportKey,
      Body: exportData,
      ServerSideEncryption: 'aws:kms',
      SSEKMSKeyId: process.env.EXPORT_KMS_KEY
    }).promise();
    
    // Create pre-signed URL valid for 24 hours
    return s3.getSignedUrl('getObject', {
      Bucket: process.env.EXPORTS_BUCKET,
      Key: exportKey,
      Expires: 86400
    });
  }
}
```

### Security Audit Trail
```javascript
// Audit logging for all data operations
const auditLogger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ 
      filename: '/var/log/audit.log',
      maxsize: 100000000, // 100MB
      maxFiles: 10
    }),
    new CloudWatchTransport({
      logGroupName: '/aws/lambda/photoshare-audit'
    })
  ]
});

const auditLog = (action, resource, userId, details = {}) => {
  auditLogger.info({
    action,
    resource,
    userId,
    timestamp: new Date().toISOString(),
    details,
    sessionId: getCurrentSessionId(),
    ipAddress: getCurrentIP()
  });
};

// Usage in API endpoints
app.post('/api/photos', async (req, res) => {
  const photo = await createPhoto(req.validatedData);
  
  auditLog('CREATE', 'PHOTO', req.user.id, {
    photoId: photo.id,
    filename: photo.filename,
    size: photo.size
  });
  
  res.json(photo);
});
```