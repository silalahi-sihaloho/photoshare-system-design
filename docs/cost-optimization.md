# Cost Optimization Strategy

## Cost Management Philosophy

PhotoShare follows a comprehensive cost optimization approach focusing on:
- **Right-sizing**: Matching resources to actual usage patterns
- **Cost Visibility**: Detailed tracking and attribution of expenses
- **Automated Optimization**: Using automation to reduce waste
- **Reserved Capacity**: Strategic use of reserved instances for predictable workloads
- **Continuous Monitoring**: Ongoing cost analysis and optimization

## Current Cost Breakdown

### Monthly Cost Analysis (Production Environment)

```
┌─────────────────────┬─────────────┬─────────────┬─────────────┬─────────────┐
│ Service Category    │ Current     │ Optimized   │ Savings     │ % Reduction │
├─────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ Compute (ECS)       │ $400        │ $280        │ $120        │ 30%         │
│ Database (RDS)      │ $300        │ $210        │ $90         │ 30%         │
│ Cache (ElastiCache) │ $150        │ $105        │ $45         │ 30%         │
│ Storage (S3)        │ $500        │ $350        │ $150        │ 30%         │
│ CDN (CloudFront)    │ $100        │ $80         │ $20         │ 20%         │
│ Load Balancer       │ $25         │ $25         │ $0          │ 0%          │
│ Data Transfer       │ $75         │ $60         │ $15         │ 20%         │
│ Lambda Functions    │ $50         │ $40         │ $10         │ 20%         │
│ Other Services      │ $50         │ $40         │ $10         │ 20%         │
├─────────────────────┼─────────────┼─────────────┼─────────────┼─────────────┤
│ TOTAL              │ $1,650      │ $1,190      │ $460        │ 28%         │
└─────────────────────┴─────────────┴─────────────┴─────────────┴─────────────┘
```

## Compute Cost Optimization

### ECS Fargate Optimization

#### Spot Instances Strategy
```yaml
# terraform/ecs-spot.tf
resource "aws_ecs_capacity_provider" "fargate_spot" {
  name = "${var.name_prefix}-fargate-spot"
  
  auto_scaling_group_provider {
    managed_scaling {
      maximum_scaling_step_size = 10
      minimum_scaling_step_size = 1
      status                   = "ENABLED"
      target_capacity          = 100
    }
    managed_termination_protection = "DISABLED"
  }
  
  tags = {
    Name = "${var.name_prefix}-fargate-spot"
  }
}

# ECS Service with mixed capacity providers
resource "aws_ecs_service" "app" {
  name            = "${var.name_prefix}-service"
  cluster         = aws_ecs_cluster.main.id
  task_definition = aws_ecs_task_definition.app.arn
  desired_count   = var.desired_count
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE"
    weight           = 20
    base             = 1  # Always have 1 on-demand instance
  }
  
  capacity_provider_strategy {
    capacity_provider = "FARGATE_SPOT"
    weight           = 80  # 80% of additional capacity on Spot
  }
}
```

#### Right-sizing Analysis
```javascript
// scripts/rightsizing-analysis.js
const AWS = require('aws-sdk');
const cloudwatch = new AWS.CloudWatch({ region: 'us-east-1' });

class RightSizingAnalyzer {
  constructor() {
    this.ecs = new AWS.ECS({ region: 'us-east-1' });
  }
  
  async analyzeECSUtilization(clusterName, serviceName, days = 30) {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - (days * 24 * 60 * 60 * 1000));
    
    // Get CPU and Memory utilization
    const cpuMetrics = await this.getMetrics('AWS/ECS', 'CPUUtilization', {
      ClusterName: clusterName,
      ServiceName: serviceName
    }, startTime, endTime);
    
    const memoryMetrics = await this.getMetrics('AWS/ECS', 'MemoryUtilization', {
      ClusterName: clusterName,
      ServiceName: serviceName
    }, startTime, endTime);
    
    const analysis = this.analyzeUtilization(cpuMetrics, memoryMetrics);
    
    return {
      currentConfiguration: await this.getCurrentTaskDefinition(serviceName),
      utilizationAnalysis: analysis,
      recommendations: this.generateRecommendations(analysis)
    };
  }
  
  async getMetrics(namespace, metricName, dimensions, startTime, endTime) {
    const params = {
      Namespace: namespace,
      MetricName: metricName,
      Dimensions: Object.entries(dimensions).map(([key, value]) => ({
        Name: key,
        Value: value
      })),
      StartTime: startTime,
      EndTime: endTime,
      Period: 3600, // 1 hour intervals
      Statistics: ['Average', 'Maximum']
    };
    
    const result = await cloudwatch.getMetricStatistics(params).promise();
    return result.Datapoints.sort((a, b) => a.Timestamp - b.Timestamp);
  }
  
  analyzeUtilization(cpuMetrics, memoryMetrics) {
    const cpuStats = this.calculateStats(cpuMetrics.map(m => m.Average));
    const memoryStats = this.calculateStats(memoryMetrics.map(m => m.Average));
    
    return {
      cpu: {
        ...cpuStats,
        peak: Math.max(...cpuMetrics.map(m => m.Maximum))
      },
      memory: {
        ...memoryStats,
        peak: Math.max(...memoryMetrics.map(m => m.Maximum))
      }
    };
  }
  
  calculateStats(values) {
    const sorted = values.sort((a, b) => a - b);
    return {
      average: values.reduce((a, b) => a + b, 0) / values.length,
      p50: sorted[Math.floor(sorted.length * 0.5)],
      p95: sorted[Math.floor(sorted.length * 0.95)],
      p99: sorted[Math.floor(sorted.length * 0.99)],
      min: Math.min(...values),
      max: Math.max(...values)
    };
  }
  
  generateRecommendations(analysis) {
    const recommendations = [];
    
    // CPU recommendations
    if (analysis.cpu.p95 < 30) {
      recommendations.push({
        type: 'cpu_downsize',
        message: 'CPU utilization is consistently low. Consider reducing CPU allocation.',
        savings: 'Up to 25% compute cost reduction'
      });
    } else if (analysis.cpu.p95 > 80) {
      recommendations.push({
        type: 'cpu_upsize',
        message: 'CPU utilization is high. Consider increasing CPU allocation.',
        impact: 'Improved performance and response times'
      });
    }
    
    // Memory recommendations
    if (analysis.memory.p95 < 40) {
      recommendations.push({
        type: 'memory_downsize',
        message: 'Memory utilization is low. Consider reducing memory allocation.',
        savings: 'Up to 20% compute cost reduction'
      });
    } else if (analysis.memory.p95 > 85) {
      recommendations.push({
        type: 'memory_upsize',
        message: 'Memory utilization is high. Consider increasing memory allocation.',
        impact: 'Reduced risk of OOM errors'
      });
    }
    
    return recommendations;
  }
}

// Generate monthly right-sizing report
const generateRightSizingReport = async () => {
  const analyzer = new RightSizingAnalyzer();
  
  const services = [
    { cluster: 'photoshare-prod', service: 'photoshare-api' },
    { cluster: 'photoshare-staging', service: 'photoshare-api' }
  ];
  
  const report = {
    generatedAt: new Date().toISOString(),
    analysis: []
  };
  
  for (const service of services) {
    try {
      const analysis = await analyzer.analyzeECSUtilization(
        service.cluster, 
        service.service
      );
      
      report.analysis.push({
        service: `${service.cluster}/${service.service}`,
        ...analysis
      });
    } catch (error) {
      console.error(`Failed to analyze ${service.cluster}/${service.service}:`, error);
    }
  }
  
  console.log(JSON.stringify(report, null, 2));
};
```

### Auto Scaling Optimization
```yaml
# Auto Scaling Configuration
AutoScalingPolicy:
  ScaleOutPolicy:
    MetricType: "CPUUtilization"
    TargetValue: 70
    ScaleOutCooldown: 300s
    StepScaling:
      - MetricIntervalLowerBound: 0
        MetricIntervalUpperBound: 50
        ScalingAdjustment: 1
      - MetricIntervalLowerBound: 50
        ScalingAdjustment: 2
        
  ScaleInPolicy:
    MetricType: "CPUUtilization" 
    TargetValue: 30
    ScaleInCooldown: 600s  # Longer cooldown for scale-in
    ScalingAdjustment: -1
    
  ScheduledScaling:
    # Scale up during peak hours (9 AM - 6 PM UTC)
    - ScheduleName: "peak-hours-scale-up"
      Recurrence: "0 9 * * *"
      MinCapacity: 4
      MaxCapacity: 20
      DesiredCapacity: 6
      
    # Scale down during off-peak hours
    - ScheduleName: "off-peak-scale-down"
      Recurrence: "0 18 * * *"
      MinCapacity: 2
      MaxCapacity: 10
      DesiredCapacity: 2
```

## Database Cost Optimization

### RDS Reserved Instances
```yaml
# Reserved Instance Strategy
RDSReservedInstances:
  ProductionPrimary:
    InstanceClass: "db.r6g.large"
    TermLength: "1yr"
    PaymentOption: "Partial Upfront"
    ExpectedSavings: "30-40%"
    
  ProductionReadReplicas:
    InstanceClass: "db.r6g.medium"
    TermLength: "1yr" 
    PaymentOption: "All Upfront"
    ExpectedSavings: "35-45%"
    
  StagingEnvironment:
    InstanceClass: "db.t3.medium"
    TermLength: "On-Demand"
    Reason: "Variable usage patterns"
```

### Database Performance Optimization
```javascript
// database/optimization.js
class DatabaseOptimizer {
  constructor() {
    this.rds = new AWS.RDS({ region: process.env.AWS_REGION });
    this.cloudwatch = new AWS.CloudWatch({ region: process.env.AWS_REGION });
  }
  
  async analyzeSlowQueries() {
    // Enable Performance Insights if not already enabled
    const dbInstances = await this.rds.describeDBInstances().promise();
    
    for (const instance of dbInstances.DBInstances) {
      if (!instance.PerformanceInsightsEnabled) {
        console.log(`Enabling Performance Insights for ${instance.DBInstanceIdentifier}`);
        await this.rds.modifyDBInstance({
          DBInstanceIdentifier: instance.DBInstanceIdentifier,
          EnablePerformanceInsights: true,
          PerformanceInsightsRetentionPeriod: 7, // Free tier
          ApplyImmediately: false
        }).promise();
      }
    }
  }
  
  async optimizeConnectionPool() {
    // Calculate optimal connection pool size
    const maxConnections = await this.getMaxConnections();
    const averageConcurrentRequests = await this.getAverageConcurrentRequests();
    
    // Rule of thumb: pool size = concurrent requests * 1.2
    const recommendedPoolSize = Math.ceil(averageConcurrentRequests * 1.2);
    const maxPoolSize = Math.min(recommendedPoolSize, maxConnections * 0.8);
    
    return {
      current: process.env.DB_POOL_MAX || 20,
      recommended: maxPoolSize,
      maxConnections,
      reasoning: `Based on ${averageConcurrentRequests} average concurrent requests`
    };
  }
  
  async getMaxConnections() {
    // Query RDS parameter group for max_connections
    const params = {
      DBInstanceIdentifier: process.env.RDS_INSTANCE_ID
    };
    
    const instance = await this.rds.describeDBInstances(params).promise();
    const parameterGroup = instance.DBInstances[0].DBParameterGroups[0].DBParameterGroupName;
    
    const parameters = await this.rds.describeDBParameters({
      DBParameterGroupName: parameterGroup,
      Source: 'engine-default'
    }).promise();
    
    const maxConnectionsParam = parameters.Parameters.find(p => p.ParameterName === 'max_connections');
    return parseInt(maxConnectionsParam?.ParameterValue || '100');
  }
  
  async getAverageConcurrentRequests() {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - (7 * 24 * 60 * 60 * 1000)); // 7 days
    
    const metrics = await this.cloudwatch.getMetricStatistics({
      Namespace: 'AWS/RDS',
      MetricName: 'DatabaseConnections',
      Dimensions: [
        {
          Name: 'DBInstanceIdentifier',
          Value: process.env.RDS_INSTANCE_ID
        }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 3600,
      Statistics: ['Average']
    }).promise();
    
    const avgConnections = metrics.Datapoints.reduce((sum, point) => sum + point.Average, 0) / metrics.Datapoints.length;
    return Math.ceil(avgConnections);
  }
}
```

## Storage Cost Optimization

### S3 Intelligent Tiering
```yaml
# S3 Lifecycle Policies
S3LifecyclePolicies:
  PhotosBucket:
    Rules:
      - Id: "IntelligentTiering"
        Status: "Enabled"
        Filter:
          Prefix: "photos/"
        Transitions:
          - Days: 0
            StorageClass: "INTELLIGENT_TIERING"
            
      - Id: "ThumbnailTransitions"
        Status: "Enabled"
        Filter:
          Prefix: "thumbnails/"
        Transitions:
          - Days: 30
            StorageClass: "STANDARD_IA"
          - Days: 90
            StorageClass: "GLACIER"
            
      - Id: "OldPhotoArchiving"
        Status: "Enabled"
        Filter:
          Prefix: "photos/"
        Transitions:
          - Days: 365
            StorageClass: "GLACIER"
          - Days: 1095  # 3 years
            StorageClass: "DEEP_ARCHIVE"
            
      - Id: "IncompleteMultipartUploads"
        Status: "Enabled"
        AbortIncompleteMultipartUpload:
          DaysAfterInitiation: 7
```

### Storage Analysis & Optimization
```javascript
// scripts/storage-analysis.js
const AWS = require('aws-sdk');
const s3 = new AWS.S3({ region: process.env.AWS_REGION });

class StorageAnalyzer {
  constructor() {
    this.bucketName = process.env.PHOTOS_BUCKET;
  }
  
  async analyzeStorageUsage() {
    const analysis = {
      totalObjects: 0,
      totalSize: 0,
      storageClasses: {},
      prefixAnalysis: {},
      costBreakdown: {}
    };
    
    // Get storage metrics from CloudWatch
    const cloudwatch = new AWS.CloudWatch({ region: process.env.AWS_REGION });
    
    const storageMetrics = await cloudwatch.getMetricStatistics({
      Namespace: 'AWS/S3',
      MetricName: 'BucketSizeBytes',
      Dimensions: [
        { Name: 'BucketName', Value: this.bucketName },
        { Name: 'StorageType', Value: 'StandardStorage' }
      ],
      StartTime: new Date(Date.now() - 24 * 60 * 60 * 1000),
      EndTime: new Date(),
      Period: 86400,
      Statistics: ['Average']
    }).promise();
    
    if (storageMetrics.Datapoints.length > 0) {
      analysis.totalSize = storageMetrics.Datapoints[0].Average;
    }
    
    // Analyze by prefix (photos/, thumbnails/, etc.)
    await this.analyzeByPrefix(analysis, 'photos/');
    await this.analyzeByPrefix(analysis, 'thumbnails/');
    await this.analyzeByPrefix(analysis, 'temp/');
    
    // Calculate cost breakdown
    analysis.costBreakdown = this.calculateCosts(analysis);
    
    return analysis;
  }
  
  async analyzeByPrefix(analysis, prefix) {
    let continuationToken;
    let totalSize = 0;
    let totalObjects = 0;
    
    do {
      const params = {
        Bucket: this.bucketName,
        Prefix: prefix,
        ContinuationToken: continuationToken
      };
      
      const result = await s3.listObjectsV2(params).promise();
      
      for (const object of result.Contents || []) {
        totalSize += object.Size;
        totalObjects++;
        
        const storageClass = object.StorageClass || 'STANDARD';
        if (!analysis.storageClasses[storageClass]) {
          analysis.storageClasses[storageClass] = { size: 0, count: 0 };
        }
        analysis.storageClasses[storageClass].size += object.Size;
        analysis.storageClasses[storageClass].count++;
      }
      
      continuationToken = result.NextContinuationToken;
    } while (continuationToken);
    
    analysis.prefixAnalysis[prefix] = {
      totalSize,
      totalObjects,
      averageSize: totalObjects > 0 ? totalSize / totalObjects : 0
    };
  }
  
  calculateCosts(analysis) {
    // S3 pricing per GB/month (US East 1)
    const pricing = {
      'STANDARD': 0.023,
      'STANDARD_IA': 0.0125,
      'INTELLIGENT_TIERING': 0.0228, // includes monitoring fee
      'GLACIER': 0.004,
      'DEEP_ARCHIVE': 0.00099
    };
    
    const costs = {};
    let totalCost = 0;
    
    for (const [storageClass, data] of Object.entries(analysis.storageClasses)) {
      const sizeGB = data.size / (1024 * 1024 * 1024);
      const monthlyCost = sizeGB * (pricing[storageClass] || pricing.STANDARD);
      
      costs[storageClass] = {
        sizeGB: sizeGB.toFixed(2),
        monthlyCost: monthlyCost.toFixed(2)
      };
      
      totalCost += monthlyCost;
    }
    
    costs.total = totalCost.toFixed(2);
    return costs;
  }
  
  async generateOptimizationRecommendations() {
    const analysis = await this.analyzeStorageUsage();
    const recommendations = [];
    
    // Check for objects that could be moved to cheaper storage
    if (analysis.storageClasses.STANDARD) {
      const standardSizeGB = analysis.storageClasses.STANDARD.size / (1024 * 1024 * 1024);
      const potentialSavings = standardSizeGB * (0.023 - 0.0125); // Standard to IA
      
      if (potentialSavings > 10) {
        recommendations.push({
          type: 'storage_class_optimization',
          description: 'Move older photos to Standard-IA storage class',
          potentialSavings: `$${potentialSavings.toFixed(2)}/month`,
          implementation: 'Update lifecycle policy to transition objects after 30 days'
        });
      }
    }
    
    // Check for temp files that should be cleaned up
    if (analysis.prefixAnalysis['temp/']?.totalSize > 1024 * 1024 * 1024) { // > 1GB
      recommendations.push({
        type: 'cleanup_temp_files',
        description: 'Clean up temporary files to reduce storage costs',
        potentialSavings: `$${(analysis.prefixAnalysis['temp/'].totalSize / (1024 * 1024 * 1024) * 0.023).toFixed(2)}/month`,
        implementation: 'Add lifecycle policy to delete temp/ objects after 1 day'
      });
    }
    
    return {
      analysis,
      recommendations
    };
  }
}

// Monthly storage optimization report
const generateStorageReport = async () => {
  const analyzer = new StorageAnalyzer();
  const report = await analyzer.generateOptimizationRecommendations();
  
  console.log('=== S3 Storage Optimization Report ===');
  console.log(JSON.stringify(report, null, 2));
};
```

## CDN Cost Optimization

### CloudFront Pricing Classes
```yaml
# CloudFront Distribution Configuration
CloudFrontDistribution:
  PriceClass: "PriceClass_100"  # Use only North America and Europe
  # Alternative: "PriceClass_200" (add Asia) or "PriceClass_All"
  
  CacheBehaviors:
    StaticAssets:
      PathPattern: "*.{jpg,jpeg,png,gif,css,js,woff,woff2}"
      TTL:
        DefaultTTL: 86400      # 1 day
        MaxTTL: 31536000       # 1 year
        MinTTL: 0
      Compress: true
      
    APIResponses:
      PathPattern: "/api/*"
      TTL:
        DefaultTTL: 0          # No caching for API
        MaxTTL: 0
        MinTTL: 0
      ForwardHeaders: ["Authorization", "Content-Type"]
      
    PhotoThumbnails:
      PathPattern: "/thumbnails/*"
      TTL:
        DefaultTTL: 604800     # 1 week
        MaxTTL: 31536000       # 1 year
        MinTTL: 86400          # 1 day
      Compress: true
```

### CDN Cache Optimization
```javascript
// utils/cdn-optimizer.js
class CDNOptimizer {
  constructor() {
    this.cloudfront = new AWS.CloudFront({ region: 'us-east-1' });
  }
  
  async analyzeCachePerformance(distributionId) {
    // Get cache statistics
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - (30 * 24 * 60 * 60 * 1000)); // 30 days
    
    const cloudwatch = new AWS.CloudWatch({ region: 'us-east-1' });
    
    const cacheHitRate = await cloudwatch.getMetricStatistics({
      Namespace: 'AWS/CloudFront',
      MetricName: 'CacheHitRate',
      Dimensions: [
        { Name: 'DistributionId', Value: distributionId }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 86400, // Daily
      Statistics: ['Average']
    }).promise();
    
    const originRequests = await cloudwatch.getMetricStatistics({
      Namespace: 'AWS/CloudFront',
      MetricName: 'Requests',
      Dimensions: [
        { Name: 'DistributionId', Value: distributionId }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 86400,
      Statistics: ['Sum']
    }).promise();
    
    return this.generateCacheRecommendations(cacheHitRate, originRequests);
  }
  
  generateCacheRecommendations(cacheHitRate, originRequests) {
    const avgCacheHitRate = cacheHitRate.Datapoints.reduce((sum, point) => sum + point.Average, 0) / cacheHitRate.Datapoints.length;
    const totalRequests = originRequests.Datapoints.reduce((sum, point) => sum + point.Sum, 0);
    
    const recommendations = [];
    
    if (avgCacheHitRate < 85) {
      recommendations.push({
        type: 'improve_cache_hit_rate',
        currentRate: `${avgCacheHitRate.toFixed(1)}%`,
        target: '90%+',
        suggestions: [
          'Increase TTL values for static content',
          'Add Cache-Control headers to responses',
          'Review cache behaviors for API endpoints'
        ]
      });
    }
    
    if (totalRequests > 1000000) { // > 1M requests per month
      const estimatedSavings = (totalRequests * 0.0001 * (90 - avgCacheHitRate) / 100).toFixed(2);
      recommendations.push({
        type: 'cost_optimization',
        description: 'Improving cache hit rate can reduce origin requests',
        estimatedSavings: `$${estimatedSavings}/month`,
        implementation: 'Optimize cache policies and TTL values'
      });
    }
    
    return {
      currentPerformance: {
        avgCacheHitRate: `${avgCacheHitRate.toFixed(1)}%`,
        totalRequests: totalRequests.toLocaleString()
      },
      recommendations
    };
  }
}
```

## Lambda Cost Optimization

### Memory and Timeout Optimization
```javascript
// scripts/lambda-optimizer.js
class LambdaOptimizer {
  constructor() {
    this.lambda = new AWS.Lambda({ region: process.env.AWS_REGION });
    this.cloudwatch = new AWS.CloudWatch({ region: process.env.AWS_REGION });
  }
  
  async optimizeFunctionConfiguration(functionName) {
    // Get function configuration
    const functionConfig = await this.lambda.getFunctionConfiguration({
      FunctionName: functionName
    }).promise();
    
    // Analyze CloudWatch metrics
    const metrics = await this.getFunctionMetrics(functionName);
    
    const recommendations = this.generateRecommendations(functionConfig, metrics);
    
    return {
      currentConfig: {
        memorySize: functionConfig.MemorySize,
        timeout: functionConfig.Timeout,
        runtime: functionConfig.Runtime
      },
      metrics,
      recommendations
    };
  }
  
  async getFunctionMetrics(functionName, days = 30) {
    const endTime = new Date();
    const startTime = new Date(endTime.getTime() - (days * 24 * 60 * 60 * 1000));
    
    const [duration, memoryUsed, invocations, errors] = await Promise.all([
      this.getMetric('Duration', functionName, startTime, endTime),
      this.getMetric('MemoryUtilization', functionName, startTime, endTime),
      this.getMetric('Invocations', functionName, startTime, endTime),
      this.getMetric('Errors', functionName, startTime, endTime)
    ]);
    
    return {
      avgDuration: this.calculateAverage(duration),
      maxDuration: this.calculateMax(duration),
      avgMemoryUsed: this.calculateAverage(memoryUsed),
      maxMemoryUsed: this.calculateMax(memoryUsed),
      totalInvocations: this.calculateSum(invocations),
      totalErrors: this.calculateSum(errors)
    };
  }
  
  async getMetric(metricName, functionName, startTime, endTime) {
    const params = {
      Namespace: 'AWS/Lambda',
      MetricName: metricName,
      Dimensions: [
        { Name: 'FunctionName', Value: functionName }
      ],
      StartTime: startTime,
      EndTime: endTime,
      Period: 3600,
      Statistics: ['Average', 'Maximum', 'Sum']
    };
    
    const result = await this.cloudwatch.getMetricStatistics(params).promise();
    return result.Datapoints;
  }
  
  generateRecommendations(config, metrics) {
    const recommendations = [];
    
    // Memory optimization
    const memoryEfficiency = (metrics.avgMemoryUsed / config.MemorySize) * 100;
    
    if (memoryEfficiency < 60) {
      const recommendedMemory = Math.ceil(metrics.maxMemoryUsed * 1.2 / 64) * 64; // Round to nearest 64MB
      const savings = this.calculateMonthlySavings(
        config.MemorySize, 
        recommendedMemory, 
        metrics.totalInvocations,
        metrics.avgDuration
      );
      
      recommendations.push({
        type: 'memory_optimization',
        current: `${config.MemorySize}MB`,
        recommended: `${recommendedMemory}MB`,
        reason: `Memory efficiency: ${memoryEfficiency.toFixed(1)}%`,
        estimatedSavings: `$${savings.toFixed(2)}/month`
      });
    }
    
    // Timeout optimization
    const timeoutEfficiency = (metrics.maxDuration / (config.Timeout * 1000)) * 100;
    
    if (timeoutEfficiency < 50) {
      const recommendedTimeout = Math.ceil(metrics.maxDuration / 1000) + 10; // Add 10s buffer
      
      recommendations.push({
        type: 'timeout_optimization',
        current: `${config.Timeout}s`,
        recommended: `${recommendedTimeout}s`,
        reason: `Max duration: ${(metrics.maxDuration / 1000).toFixed(1)}s`,
        benefit: 'Faster failure detection and reduced costs'
      });
    }
    
    return recommendations;
  }
  
  calculateMonthlySavings(currentMemory, recommendedMemory, invocations, avgDuration) {
    // Lambda pricing: $0.0000166667 per GB-second
    const pricePerGBSecond = 0.0000166667;
    
    const currentCost = (currentMemory / 1024) * (avgDuration / 1000) * invocations * pricePerGBSecond;
    const newCost = (recommendedMemory / 1024) * (avgDuration / 1000) * invocations * pricePerGBSecond;
    
    return Math.max(0, currentCost - newCost) * 30; // Monthly
  }
  
  calculateAverage(datapoints) {
    if (datapoints.length === 0) return 0;
    return datapoints.reduce((sum, point) => sum + point.Average, 0) / datapoints.length;
  }
  
  calculateMax(datapoints) {
    if (datapoints.length === 0) return 0;
    return Math.max(...datapoints.map(point => point.Maximum));
  }
  
  calculateSum(datapoints) {
    return datapoints.reduce((sum, point) => sum + point.Sum, 0);
  }
}
```

## Cost Monitoring & Alerting

### AWS Budgets Configuration
```yaml
# cloudformation/budgets.yaml
Resources:
  MonthlyBudget:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetName: PhotoShare-Monthly-Budget
        BudgetLimit:
          Amount: 2000
          Unit: USD
        TimeUnit: MONTHLY
        BudgetType: COST
        CostFilters:
          Service:
            - Amazon Elastic Compute Cloud - Compute
            - Amazon Relational Database Service
            - Amazon Simple Storage Service
            - Amazon CloudFront
            - AWS Lambda
        NotificationsWithSubscribers:
          - Notification:
              NotificationType: ACTUAL
              ComparisonOperator: GREATER_THAN
              Threshold: 80
              ThresholdType: PERCENTAGE
            Subscribers:
              - SubscriptionType: EMAIL
                Address: devops@photoshare.com
          - Notification:
              NotificationType: FORECASTED
              ComparisonOperator: GREATER_THAN
              Threshold: 100
              ThresholdType: PERCENTAGE
            Subscribers:
              - SubscriptionType: EMAIL
                Address: devops@photoshare.com
                
  ServiceSpecificBudgets:
    Type: AWS::Budgets::Budget
    Properties:
      Budget:
        BudgetName: PhotoShare-ECS-Budget
        BudgetLimit:
          Amount: 600
          Unit: USD
        TimeUnit: MONTHLY
        BudgetType: COST
        CostFilters:
          Service:
            - Amazon Elastic Container Service
        NotificationsWithSubscribers:
          - Notification:
              NotificationType: ACTUAL
              ComparisonOperator: GREATER_THAN
              Threshold: 90
              ThresholdType: PERCENTAGE
            Subscribers:
              - SubscriptionType: EMAIL
                Address: devops@photoshare.com
```

### Cost Analysis Dashboard
```javascript
// utils/cost-analyzer.js
const AWS = require('aws-sdk');

class CostAnalyzer {
  constructor() {
    this.costExplorer = new AWS.CostExplorer({ region: 'us-east-1' });
  }
  
  async generateMonthlyCostReport() {
    const endDate = new Date().toISOString().split('T')[0];
    const startDate = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000).toISOString().split('T')[0];
    
    // Get cost by service
    const costByService = await this.costExplorer.getCostAndUsage({
      TimePeriod: {
        Start: startDate,
        End: endDate
      },
      Granularity: 'MONTHLY',
      Metrics: ['BlendedCost'],
      GroupBy: [
        {
          Type: 'DIMENSION',
          Key: 'SERVICE'
        }
      ]
    }).promise();
    
    // Get cost by resource
    const costByResource = await this.costExplorer.getCostAndUsage({
      TimePeriod: {
        Start: startDate,
        End: endDate
      },
      Granularity: 'MONTHLY',
      Metrics: ['BlendedCost'],
      GroupBy: [
        {
          Type: 'DIMENSION',
          Key: 'RESOURCE_ID'
        }
      ]
    }).promise();
    
    return {
      period: { start: startDate, end: endDate },
      totalCost: this.extractTotalCost(costByService),
      costByService: this.processServiceCosts(costByService),
      costByResource: this.processResourceCosts(costByResource),
      recommendations: await this.generateCostRecommendations()
    };
  }
  
  extractTotalCost(costData) {
    return costData.ResultsByTime[0]?.Total?.BlendedCost?.Amount || '0';
  }
  
  processServiceCosts(costData) {
    const services = costData.ResultsByTime[0]?.Groups || [];
    
    return services
      .map(group => ({
        service: group.Keys[0],
        cost: parseFloat(group.Metrics.BlendedCost.Amount),
        unit: group.Metrics.BlendedCost.Unit
      }))
      .sort((a, b) => b.cost - a.cost)
      .slice(0, 10); // Top 10 services
  }
  
  processResourceCosts(costData) {
    const resources = costData.ResultsByTime[0]?.Groups || [];
    
    return resources
      .map(group => ({
        resource: group.Keys[0],
        cost: parseFloat(group.Metrics.BlendedCost.Amount),
        unit: group.Metrics.BlendedCost.Unit
      }))
      .sort((a, b) => b.cost - a.cost)
      .slice(0, 20); // Top 20 resources
  }
  
  async generateCostRecommendations() {
    // Get Right Sizing recommendations
    const rightSizing = await this.costExplorer.getRightSizingRecommendation({
      Service: 'AmazonEC2'
    }).promise();
    
    // Get Reserved Instance recommendations
    const reservedInstances = await this.costExplorer.getReservationPurchaseRecommendation({
      Service: 'AmazonEC2'
    }).promise();
    
    return {
      rightSizing: rightSizing.RightSizingRecommendations?.slice(0, 5) || [],
      reservedInstances: reservedInstances.Recommendations?.slice(0, 5) || []
    };
  }
  
  async trackCostTrends(days = 90) {
    const endDate = new Date().toISOString().split('T')[0];
    const startDate = new Date(Date.now() - days * 24 * 60 * 60 * 1000).toISOString().split('T')[0];
    
    const trendData = await this.costExplorer.getCostAndUsage({
      TimePeriod: {
        Start: startDate,
        End: endDate
      },
      Granularity: 'DAILY',
      Metrics: ['BlendedCost']
    }).promise();
    
    return trendData.ResultsByTime.map(result => ({
      date: result.TimePeriod.Start,
      cost: parseFloat(result.Total.BlendedCost.Amount)
    }));
  }
}

// Schedule monthly cost report
const generateCostReport = async () => {
  const analyzer = new CostAnalyzer();
  const report = await analyzer.generateMonthlyCostReport();
  
  console.log('=== Monthly Cost Report ===');
  console.log(`Total Cost: $${report.totalCost}`);
  console.log('\nTop Services:');
  report.costByService.forEach(service => {
    console.log(`${service.service}: $${service.cost.toFixed(2)}`);
  });
  
  console.log('\nCost Optimization Recommendations:');
  console.log(JSON.stringify(report.recommendations, null, 2));
};
```

## Reserved Instance Strategy

### RI Purchase Recommendations
```javascript
// scripts/ri-analyzer.js
class ReservedInstanceAnalyzer {
  constructor() {
    this.ec2 = new AWS.EC2({ region: process.env.AWS_REGION });
    this.rds = new AWS.RDS({ region: process.env.AWS_REGION });
    this.elasticache = new AWS.ElastiCache({ region: process.env.AWS_REGION });
  }
  
  async analyzeRIOpportunities() {
    const analysis = {
      ec2: await this.analyzeEC2Instances(),
      rds: await this.analyzeRDSInstances(),
      elasticache: await this.analyzeElastiCacheInstances()
    };
    
    return {
      ...analysis,
      totalPotentialSavings: this.calculateTotalSavings(analysis)
    };
  }
  
  async analyzeRDSInstances() {
    const instances = await this.rds.describeDBInstances().promise();
    const recommendations = [];
    
    for (const instance of instances.DBInstances) {
      if (instance.DBInstanceStatus === 'available') {
        const utilization = await this.getRDSUtilization(instance.DBInstanceIdentifier);
        
        if (utilization.averageUptime > 70) { // Running >70% of time
          const onDemandCost = this.calculateRDSOnDemandCost(instance.DBInstanceClass);
          const riCost = this.calculateRDSReservedCost(instance.DBInstanceClass, '1yr', 'Partial Upfront');
          
          recommendations.push({
            instanceId: instance.DBInstanceIdentifier,
            instanceClass: instance.DBInstanceClass,
            currentMonthlyCost: onDemandCost,
            riMonthlyCost: riCost.monthly,
            upfrontCost: riCost.upfront,
            monthlySavings: onDemandCost - riCost.monthly,
            annualSavings: (onDemandCost - riCost.monthly) * 12,
            paybackPeriod: riCost.upfront / (onDemandCost - riCost.monthly),
            recommendation: 'Purchase 1-year Partial Upfront RI'
          });
        }
      }
    }
    
    return recommendations;
  }
  
  calculateRDSOnDemandCost(instanceClass) {
    // RDS pricing (simplified - actual pricing varies by region)
    const pricing = {
      'db.t3.micro': 0.017,
      'db.t3.small': 0.034,
      'db.t3.medium': 0.068,
      'db.r6g.large': 0.24,
      'db.r6g.xlarge': 0.48,
      'db.r6g.2xlarge': 0.96
    };
    
    const hourlyRate = pricing[instanceClass] || 0.1;
    return hourlyRate * 24 * 30; // Monthly cost
  }
  
  calculateRDSReservedCost(instanceClass, term, paymentOption) {
    // Reserved Instance pricing (simplified)
    const discounts = {
      '1yr': {
        'No Upfront': 0.25,
        'Partial Upfront': 0.35,
        'All Upfront': 0.40
      },
      '3yr': {
        'No Upfront': 0.35,
        'Partial Upfront': 0.45,
        'All Upfront': 0.50
      }
    };
    
    const onDemandHourly = this.calculateRDSOnDemandCost(instanceClass) / (24 * 30);
    const discount = discounts[term][paymentOption];
    const riHourly = onDemandHourly * (1 - discount);
    
    const termMonths = term === '1yr' ? 12 : 36;
    const totalRICost = riHourly * 24 * 30 * termMonths;
    
    let upfront = 0;
    let monthly = riHourly * 24 * 30;
    
    if (paymentOption === 'Partial Upfront') {
      upfront = totalRICost * 0.5;
      monthly = totalRICost * 0.5 / termMonths;
    } else if (paymentOption === 'All Upfront') {
      upfront = totalRICost;
      monthly = 0;
    }
    
    return { upfront, monthly };
  }
  
  async getRDSUtilization(instanceId) {
    // Mock utilization data - in real implementation, query CloudWatch
    return {
      averageUptime: 85,
      peakUsagePeriods: ['9AM-6PM'],
      recommendedSize: 'current'
    };
  }
  
  calculateTotalSavings(analysis) {
    let totalAnnualSavings = 0;
    
    ['ec2', 'rds', 'elasticache'].forEach(service => {
      if (analysis[service]) {
        totalAnnualSavings += analysis[service].reduce((sum, rec) => sum + rec.annualSavings, 0);
      }
    });
    
    return totalAnnualSavings;
  }
}

// Generate RI recommendations report
const generateRIReport = async () => {
  const analyzer = new ReservedInstanceAnalyzer();
  const opportunities = await analyzer.analyzeRIOpportunities();
  
  console.log('=== Reserved Instance Opportunities ===');
  console.log(`Total Potential Annual Savings: $${opportunities.totalPotentialSavings.toFixed(2)}`);
  
  console.log('\nRDS Recommendations:');
  opportunities.rds.forEach(rec => {
    console.log(`${rec.instanceId} (${rec.instanceClass}): Save $${rec.annualSavings.toFixed(2)}/year`);
  });
};
```

## Automated Cost Optimization

### Cost Optimization Lambda
```javascript
// lambda/cost-optimizer.js
const AWS = require('aws-sdk');

exports.handler = async (event) => {
  const optimizations = [];
  
  try {
    // 1. Clean up unused EBS snapshots
    const ebsOptimizations = await cleanupEBSSnapshots();
    optimizations.push(...ebsOptimizations);
    
    // 2. Remove unused Elastic IPs
    const eipOptimizations = await cleanupUnusedEIPs();
    optimizations.push(...eipOptimizations);
    
    // 3. Delete old CloudWatch logs
    const logOptimizations = await cleanupOldLogs();
    optimizations.push(...logOptimizations);
    
    // 4. Optimize S3 storage classes
    const s3Optimizations = await optimizeS3Storage();
    optimizations.push(...s3Optimizations);
    
    // 5. Send optimization report
    await sendOptimizationReport(optimizations);
    
    return {
      statusCode: 200,
      body: JSON.stringify({
        message: 'Cost optimization completed',
        optimizations: optimizations.length,
        estimatedSavings: optimizations.reduce((sum, opt) => sum + opt.savings, 0)
      })
    };
    
  } catch (error) {
    console.error('Cost optimization failed:', error);
    
    return {
      statusCode: 500,
      body: JSON.stringify({
        message: 'Cost optimization failed',
        error: error.message
      })
    };
  }
};

async function cleanupEBSSnapshots() {
  const ec2 = new AWS.EC2({ region: process.env.AWS_REGION });
  const optimizations = [];
  
  // Get all snapshots owned by this account
  const snapshots = await ec2.describeSnapshots({
    OwnerIds: ['self']
  }).promise();
  
  const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
  
  for (const snapshot of snapshots.Snapshots) {
    const snapshotDate = new Date(snapshot.StartTime);
    
    // Check if snapshot is older than 30 days and not associated with any AMI
    if (snapshotDate < thirtyDaysAgo) {
      const isUsed = await isSnapshotUsedByAMI(snapshot.SnapshotId);
      
      if (!isUsed) {
        try {
          await ec2.deleteSnapshot({ SnapshotId: snapshot.SnapshotId }).promise();
          
          optimizations.push({
            type: 'ebs_snapshot_cleanup',
            resourceId: snapshot.SnapshotId,
            action: 'deleted',
            savings: 0.05 * snapshot.VolumeSize // $0.05 per GB/month
          });
        } catch (error) {
          console.error(`Failed to delete snapshot ${snapshot.SnapshotId}:`, error);
        }
      }
    }
  }
  
  return optimizations;
}

async function cleanupUnusedEIPs() {
  const ec2 = new AWS.EC2({ region: process.env.AWS_REGION });
  const optimizations = [];
  
  const addresses = await ec2.describeAddresses().promise();
  
  for (const address of addresses.Addresses) {
    // Release unassociated Elastic IPs
    if (!address.InstanceId && !address.NetworkInterfaceId) {
      try {
        await ec2.releaseAddress({ AllocationId: address.AllocationId }).promise();
        
        optimizations.push({
          type: 'eip_cleanup',
          resourceId: address.AllocationId,
          action: 'released',
          savings: 3.65 // $3.65 per month for unused EIP
        });
      } catch (error) {
        console.error(`Failed to release EIP ${address.AllocationId}:`, error);
      }
    }
  }
  
  return optimizations;
}

async function cleanupOldLogs() {
  const cloudwatchlogs = new AWS.CloudWatchLogs({ region: process.env.AWS_REGION });
  const optimizations = [];
  
  const logGroups = await cloudwatchlogs.describeLogGroups().promise();
  
  for (const logGroup of logGroups.logGroups) {
    // Set retention policy for log groups without one
    if (!logGroup.retentionInDays) {
      try {
        await cloudwatchlogs.putRetentionPolicy({
          logGroupName: logGroup.logGroupName,
          retentionInDays: 30 // 30 days retention
        }).promise();
        
        optimizations.push({
          type: 'log_retention_policy',
          resourceId: logGroup.logGroupName,
          action: 'set_retention_30_days',
          savings: 1.0 // Estimated savings
        });
      } catch (error) {
        console.error(`Failed to set retention for ${logGroup.logGroupName}:`, error);
      }
    }
  }
  
  return optimizations;
}

async function optimizeS3Storage() {
  // Implementation similar to storage analyzer
  // Move objects to appropriate storage classes
  return [];
}

async function sendOptimizationReport(optimizations) {
  const sns = new AWS.SNS({ region: process.env.AWS_REGION });
  
  const totalSavings = optimizations.reduce((sum, opt) => sum + opt.savings, 0);
  
  const message = `
PhotoShare Cost Optimization Report

Total Optimizations: ${optimizations.length}
Estimated Monthly Savings: $${totalSavings.toFixed(2)}

Optimizations Performed:
${optimizations.map(opt => `- ${opt.type}: ${opt.action} (${opt.resourceId})`).join('\n')}
  `;
  
  await sns.publish({
    TopicArn: process.env.COST_OPTIMIZATION_TOPIC_ARN,
    Subject: 'PhotoShare Cost Optimization Report',
    Message: message
  }).promise();
}
```

This comprehensive cost optimization strategy covers all major AWS services used by PhotoShare and provides automated tools for ongoing cost management. The strategy focuses on right-sizing resources, using reserved capacity strategically, implementing lifecycle policies, and continuous monitoring to achieve the target 28% cost reduction.