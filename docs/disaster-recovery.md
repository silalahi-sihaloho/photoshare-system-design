# Disaster Recovery Plan

## Overview

PhotoShare's disaster recovery (DR) plan ensures business continuity and rapid recovery from catastrophic events. Our DR strategy is designed to meet:
- **Recovery Time Objective (RTO)**: 15 minutes for critical services
- **Recovery Point Objective (RPO)**: 5 minutes maximum data loss
- **Business Continuity**: 99.9% annual uptime SLA

## Risk Assessment

### Potential Disaster Scenarios

| Risk Type | Probability | Impact | Mitigation Priority |
|-----------|-------------|---------|-------------------|
| AWS Region Outage | Low | High | Critical |
| Database Corruption | Medium | High | Critical |
| Application Bugs | High | Medium | High |
| Security Breach | Medium | High | Critical |
| Human Error | High | Medium | High |
| Natural Disasters | Low | High | Medium |
| DDoS Attack | Medium | Medium | High |
| Third-party Dependencies | Medium | Medium | Medium |

### Business Impact Analysis

```
┌─────────────────────┬─────────────┬─────────────┬─────────────┐
│ Service Component   │ Criticality │ Max Downtime│ Dependencies│
├─────────────────────┼─────────────┼─────────────┼─────────────┤
│ User Authentication │ Critical    │ 5 minutes   │ AWS Cognito │
│ Photo Upload        │ Critical    │ 15 minutes  │ S3, Lambda  │
│ Feed Generation     │ High        │ 30 minutes  │ RDS, Cache  │
│ Search              │ Medium      │ 1 hour      │ OpenSearch  │
│ Social Features     │ Medium      │ 1 hour      │ DynamoDB    │
│ Admin Dashboard     │ Low         │ 4 hours     │ All Services│
└─────────────────────┴─────────────┴─────────────┴─────────────┘
```

## Multi-Region Architecture

### Primary Region: us-east-1
- **Production Environment**: Complete infrastructure
- **Real-time Replication**: To us-west-2
- **Auto-failover**: Enabled for critical components

### Secondary Region: us-west-2
- **Warm Standby**: Scaled-down infrastructure
- **Data Replication**: Real-time for critical data
- **Auto-scaling**: Configured to scale up on failover

### Cross-Region Setup

```yaml
# terraform/disaster-recovery.tf
# Primary Region (us-east-1)
provider "aws" {
  alias  = "primary"
  region = "us-east-1"
}

# Secondary Region (us-west-2)
provider "aws" {
  alias  = "secondary"
  region = "us-west-2"
}

# RDS Cross-Region Backup
resource "aws_db_instance" "primary" {
  provider = aws.primary
  
  identifier = "photoshare-primary"
  engine     = "postgres"
  engine_version = "14.9"
  instance_class = "db.r6g.large"
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  # Enable automated backups
  copy_tags_to_snapshot = true
  
  # Cross-region automated backups
  manage_master_user_password = true
  
  tags = {
    Environment = "production"
    Purpose     = "primary-database"
  }
}

# Read Replica in Secondary Region
resource "aws_db_instance" "cross_region_replica" {
  provider = aws.secondary
  
  identifier = "photoshare-replica-west"
  
  # Create from primary instance
  replicate_source_db = aws_db_instance.primary.identifier
  
  instance_class = "db.r6g.large"
  
  # Can be promoted to standalone
  auto_minor_version_upgrade = false
  
  tags = {
    Environment = "production"
    Purpose     = "disaster-recovery"
  }
}

# S3 Cross-Region Replication
resource "aws_s3_bucket" "primary_photos" {
  provider = aws.primary
  bucket   = "photoshare-photos-primary"
}

resource "aws_s3_bucket" "secondary_photos" {
  provider = aws.secondary
  bucket   = "photoshare-photos-secondary"
}

resource "aws_s3_bucket_replication_configuration" "photos_replication" {
  provider = aws.primary
  
  role   = aws_iam_role.replication.arn
  bucket = aws_s3_bucket.primary_photos.id
  
  rule {
    id     = "photos_replication"
    status = "Enabled"
    
    destination {
      bucket        = aws_s3_bucket.secondary_photos.arn
      storage_class = "STANDARD_IA"
    }
  }
  
  depends_on = [aws_s3_bucket_versioning.primary_photos]
}

# ElastiCache Global Replication Group
resource "aws_elasticache_global_replication_group" "photoshare_cache" {
  global_replication_group_id = "photoshare-global-cache"
  description                = "Global cache for PhotoShare"
  
  primary_replication_group_id = aws_elasticache_replication_group.primary.id
}

resource "aws_elasticache_replication_group" "primary" {
  provider = aws.primary
  
  replication_group_id       = "photoshare-cache-primary"
  description                = "Primary cache cluster"
  
  node_type                  = "cache.r6g.large"
  port                      = 6379
  parameter_group_name      = "default.redis7"
  
  num_cache_clusters        = 2
  automatic_failover_enabled = true
  multi_az_enabled          = true
  
  global_replication_group_id = aws_elasticache_global_replication_group.photoshare_cache.global_replication_group_id
}

resource "aws_elasticache_replication_group" "secondary" {
  provider = aws.secondary
  
  replication_group_id       = "photoshare-cache-secondary"
  description                = "Secondary cache cluster"
  
  node_type                  = "cache.r6g.large"
  port                      = 6379
  parameter_group_name      = "default.redis7"
  
  num_cache_clusters        = 1
  automatic_failover_enabled = false
  
  global_replication_group_id = aws_elasticache_global_replication_group.photoshare_cache.global_replication_group_id
}
```

## Backup Strategy

### Database Backups

#### Automated Backups
```sql
-- RDS Automated Backup Configuration
-- Retention: 7 days for production, 1 day for staging
-- Backup Window: 03:00-04:00 UTC (low traffic period)
-- Point-in-time Recovery: Enabled

-- Manual Snapshot Creation
CREATE OR REPLACE FUNCTION create_manual_backup()
RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
    -- Create manual snapshot via AWS CLI
    PERFORM aws_s3.query_export_to_s3(
        'SELECT pg_start_backup(''manual_backup_'' || current_timestamp)',
        aws_commons.create_s3_uri(
            'photoshare-db-backups',
            'manual/' || to_char(current_timestamp, 'YYYY/MM/DD/') || 'backup.sql',
            'us-east-1'
        )
    );
END;
$$;

-- Schedule manual backups
SELECT cron.schedule(
    'manual-backup',
    '0 2 * * 0',  -- Weekly on Sunday at 2 AM
    'SELECT create_manual_backup();'
);
```

#### Cross-Region Backup Script
```bash
#!/bin/bash
# scripts/cross-region-backup.sh

set -e

TIMESTAMP=$(date +%Y%m%d_%H%M%S)
PRIMARY_REGION="us-east-1"
SECONDARY_REGION="us-west-2"
DB_INSTANCE_ID="photoshare-primary"

echo "Starting cross-region backup process..."

# Create snapshot in primary region
SNAPSHOT_ID="photoshare-manual-${TIMESTAMP}"

aws rds create-db-snapshot \
    --db-instance-identifier $DB_INSTANCE_ID \
    --db-snapshot-identifier $SNAPSHOT_ID \
    --region $PRIMARY_REGION

echo "Waiting for snapshot to complete..."
aws rds wait db-snapshot-completed \
    --db-snapshot-identifier $SNAPSHOT_ID \
    --region $PRIMARY_REGION

# Copy snapshot to secondary region
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier "arn:aws:rds:${PRIMARY_REGION}:$(aws sts get-caller-identity --query Account --output text):snapshot:${SNAPSHOT_ID}" \
    --target-db-snapshot-identifier $SNAPSHOT_ID \
    --region $SECONDARY_REGION

echo "Snapshot copied to secondary region"

# Clean up old snapshots (keep last 7)
OLD_SNAPSHOTS=$(aws rds describe-db-snapshots \
    --db-instance-identifier $DB_INSTANCE_ID \
    --snapshot-type manual \
    --query "reverse(sort_by(DBSnapshots[?starts_with(DBSnapshotIdentifier, 'photoshare-manual-')], &SnapshotCreateTime))[7:].DBSnapshotIdentifier" \
    --output text \
    --region $PRIMARY_REGION)

for snapshot in $OLD_SNAPSHOTS; do
    if [ ! -z "$snapshot" ]; then
        echo "Deleting old snapshot: $snapshot"
        aws rds delete-db-snapshot \
            --db-snapshot-identifier $snapshot \
            --region $PRIMARY_REGION
    fi
done

echo "Cross-region backup completed successfully"
```

### Application Data Backups

#### Configuration Backup
```javascript
// scripts/backup-configuration.js
const AWS = require('aws-sdk');
const s3 = new AWS.S3({ region: 'us-east-1' });
const ssm = new AWS.SSM({ region: 'us-east-1' });

class ConfigurationBackup {
  constructor() {
    this.backupBucket = 'photoshare-config-backups';
    this.timestamp = new Date().toISOString().replace(/[:.]/g, '-');
  }
  
  async backupSSMParameters() {
    console.log('Backing up SSM parameters...');
    
    const parameters = await ssm.getParametersByPath({
      Path: '/photoshare/',
      Recursive: true,
      WithDecryption: true
    }).promise();
    
    const backupData = {
      timestamp: this.timestamp,
      parameters: parameters.Parameters.map(param => ({
        name: param.Name,
        value: param.Value,
        type: param.Type,
        description: param.Description
      }))
    };
    
    await s3.putObject({
      Bucket: this.backupBucket,
      Key: `ssm-parameters/${this.timestamp}/parameters.json`,
      Body: JSON.stringify(backupData, null, 2),
      ServerSideEncryption: 'aws:kms'
    }).promise();
    
    console.log('SSM parameters backed up successfully');
  }
  
  async backupSecrets() {
    console.log('Backing up secrets...');
    
    const secretsManager = new AWS.SecretsManager({ region: 'us-east-1' });
    
    const secrets = await secretsManager.listSecrets({
      Filters: [
        {
          Key: 'name',
          Values: ['photoshare/']
        }
      ]
    }).promise();
    
    const backupData = {
      timestamp: this.timestamp,
      secrets: []
    };
    
    for (const secret of secrets.SecretList) {
      try {
        const secretValue = await secretsManager.getSecretValue({
          SecretId: secret.ARN
        }).promise();
        
        backupData.secrets.push({
          name: secret.Name,
          value: secretValue.SecretString,
          description: secret.Description
        });
      } catch (error) {
        console.error(`Failed to backup secret ${secret.Name}:`, error);
      }
    }
    
    await s3.putObject({
      Bucket: this.backupBucket,
      Key: `secrets/${this.timestamp}/secrets.json`,
      Body: JSON.stringify(backupData, null, 2),
      ServerSideEncryption: 'aws:kms'
    }).promise();
    
    console.log('Secrets backed up successfully');
  }
  
  async backupInfrastructure() {
    console.log('Backing up infrastructure state...');
    
    // Export Terraform state
    const { execSync } = require('child_process');
    
    try {
      // Pull latest state
      execSync('terraform refresh', { cwd: './terraform' });
      
      // Export state
      const stateOutput = execSync('terraform show -json', { 
        cwd: './terraform',
        encoding: 'utf8'
      });
      
      await s3.putObject({
        Bucket: this.backupBucket,
        Key: `terraform-state/${this.timestamp}/terraform.tfstate.json`,
        Body: stateOutput,
        ServerSideEncryption: 'aws:kms'
      }).promise();
      
      console.log('Infrastructure state backed up successfully');
    } catch (error) {
      console.error('Failed to backup infrastructure state:', error);
    }
  }
  
  async run() {
    try {
      await this.backupSSMParameters();
      await this.backupSecrets();
      await this.backupInfrastructure();
      
      console.log('Configuration backup completed successfully');
    } catch (error) {
      console.error('Configuration backup failed:', error);
      process.exit(1);
    }
  }
}

// Run backup
const backup = new ConfigurationBackup();
backup.run();
```

## Failover Procedures

### Automated Failover

#### Route 53 Health Checks
```yaml
# cloudformation/health-checks.yaml
Resources:
  PrimaryHealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      Type: HTTPS
      ResourcePath: /health
      FullyQualifiedDomainName: api.photoshare.com
      Port: 443
      RequestInterval: 30
      FailureThreshold: 3
      
  SecondaryHealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      Type: HTTPS
      ResourcePath: /health
      FullyQualifiedDomainName: api-west.photoshare.com
      Port: 443
      RequestInterval: 30
      FailureThreshold: 3
      
  DNSRecordSet:
    Type: AWS::Route53::RecordSetGroup
    Properties:
      HostedZoneId: !Ref PhotoShareHostedZone
      RecordSets:
        - Name: api.photoshare.com
          Type: A
          SetIdentifier: "primary"
          Failover: PRIMARY
          TTL: 60
          ResourceRecords:
            - !GetAtt PrimaryLoadBalancer.DNSName
          HealthCheckId: !Ref PrimaryHealthCheck
            
        - Name: api.photoshare.com
          Type: A
          SetIdentifier: "secondary"
          Failover: SECONDARY
          TTL: 60
          ResourceRecords:
            - !GetAtt SecondaryLoadBalancer.DNSName
          HealthCheckId: !Ref SecondaryHealthCheck
```

#### Lambda Failover Function
```javascript
// lambda/failover-handler.js
const AWS = require('aws-sdk');

exports.handler = async (event) => {
    console.log('Failover event received:', JSON.stringify(event, null, 2));
    
    const region = process.env.AWS_REGION;
    const targetRegion = region === 'us-east-1' ? 'us-west-2' : 'us-east-1';
    
    try {
        // Step 1: Promote RDS read replica to primary
        await promoteReadReplica(targetRegion);
        
        // Step 2: Scale up ECS services in target region
        await scaleUpECSServices(targetRegion);
        
        // Step 3: Update DNS records for immediate failover
        await updateDNSRecords(targetRegion);
        
        // Step 4: Notify operations team
        await sendFailoverNotification('SUCCESS', targetRegion);
        
        return {
            statusCode: 200,
            body: JSON.stringify({
                message: 'Failover completed successfully',
                targetRegion,
                timestamp: new Date().toISOString()
            })
        };
        
    } catch (error) {
        console.error('Failover failed:', error);
        
        await sendFailoverNotification('FAILED', targetRegion, error.message);
        
        return {
            statusCode: 500,
            body: JSON.stringify({
                message: 'Failover failed',
                error: error.message
            })
        };
    }
};

async function promoteReadReplica(targetRegion) {
    const rds = new AWS.RDS({ region: targetRegion });
    
    console.log(`Promoting read replica in ${targetRegion}...`);
    
    await rds.promoteReadReplica({
        DBInstanceIdentifier: 'photoshare-replica-west'
    }).promise();
    
    // Wait for promotion to complete
    await rds.waitFor('dBInstanceAvailable', {
        DBInstanceIdentifier: 'photoshare-replica-west'
    }).promise();
    
    console.log('Read replica promoted successfully');
}

async function scaleUpECSServices(targetRegion) {
    const ecs = new AWS.ECS({ region: targetRegion });
    
    console.log(`Scaling up ECS services in ${targetRegion}...`);
    
    // Scale up API service
    await ecs.updateService({
        cluster: 'photoshare-cluster-secondary',
        service: 'photoshare-api-secondary',
        desiredCount: 4  // Scale up from warm standby
    }).promise();
    
    // Wait for services to be stable
    await ecs.waitFor('servicesStable', {
        cluster: 'photoshare-cluster-secondary',
        services: ['photoshare-api-secondary']
    }).promise();
    
    console.log('ECS services scaled up successfully');
}

async function updateDNSRecords(targetRegion) {
    const route53 = new AWS.Route53();
    
    console.log('Updating DNS records for immediate failover...');
    
    // Force failover by updating health check
    await route53.changeResourceRecordSets({
        HostedZoneId: process.env.HOSTED_ZONE_ID,
        ChangeBatch: {
            Changes: [
                {
                    Action: 'UPSERT',
                    ResourceRecordSet: {
                        Name: 'api.photoshare.com',
                        Type: 'A',
                        SetIdentifier: 'failover-immediate',
                        TTL: 60,
                        ResourceRecords: [
                            { Value: getRegionLoadBalancerIP(targetRegion) }
                        ]
                    }
                }
            ]
        }
    }).promise();
    
    console.log('DNS records updated successfully');
}

async function sendFailoverNotification(status, targetRegion, error = null) {
    const sns = new AWS.SNS({ region: 'us-east-1' });
    
    const message = {
        timestamp: new Date().toISOString(),
        status,
        targetRegion,
        sourceRegion: process.env.AWS_REGION,
        error
    };
    
    await sns.publish({
        TopicArn: process.env.FAILOVER_NOTIFICATION_TOPIC,
        Subject: `PhotoShare Failover ${status}`,
        Message: JSON.stringify(message, null, 2)
    }).promise();
}

function getRegionLoadBalancerIP(region) {
    const loadBalancerIPs = {
        'us-east-1': process.env.PRIMARY_LB_IP,
        'us-west-2': process.env.SECONDARY_LB_IP
    };
    
    return loadBalancerIPs[region];
}
```

### Manual Failover Procedures

#### Primary to Secondary Region Failover
```bash
#!/bin/bash
# scripts/manual-failover.sh

set -e

SECONDARY_REGION="us-west-2"
PRIMARY_REGION="us-east-1"

echo "=== PhotoShare Manual Failover Procedure ==="
echo "Failing over from $PRIMARY_REGION to $SECONDARY_REGION"

# Confirm failover decision
read -p "Are you sure you want to proceed with failover? (yes/no): " confirmation
if [ "$confirmation" != "yes" ]; then
    echo "Failover cancelled"
    exit 1
fi

echo "Starting failover process..."

# Step 1: Stop writes to primary database
echo "1. Stopping writes to primary database..."
aws rds modify-db-instance \
    --db-instance-identifier photoshare-primary \
    --region $PRIMARY_REGION \
    --no-deletion-protection \
    --apply-immediately

# Step 2: Promote read replica
echo "2. Promoting read replica to primary..."
aws rds promote-read-replica \
    --db-instance-identifier photoshare-replica-west \
    --region $SECONDARY_REGION

# Wait for promotion
echo "Waiting for database promotion to complete..."
aws rds wait db-instance-available \
    --db-instance-identifier photoshare-replica-west \
    --region $SECONDARY_REGION

# Step 3: Update application configuration
echo "3. Updating application configuration..."

# Update database endpoint in SSM
NEW_DB_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier photoshare-replica-west \
    --region $SECONDARY_REGION \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

aws ssm put-parameter \
    --name "/photoshare/database/endpoint" \
    --value "$NEW_DB_ENDPOINT" \
    --overwrite \
    --region $SECONDARY_REGION

# Step 4: Scale up secondary region services
echo "4. Scaling up services in secondary region..."

# Scale ECS services
aws ecs update-service \
    --cluster photoshare-cluster-secondary \
    --service photoshare-api-secondary \
    --desired-count 4 \
    --region $SECONDARY_REGION

# Wait for services to stabilize
echo "Waiting for services to stabilize..."
aws ecs wait services-stable \
    --cluster photoshare-cluster-secondary \
    --services photoshare-api-secondary \
    --region $SECONDARY_REGION

# Step 5: Update Route 53 records
echo "5. Updating DNS records..."
./scripts/update-dns-failover.sh $SECONDARY_REGION

# Step 6: Verify failover
echo "6. Verifying failover..."
sleep 30  # Wait for DNS propagation

HEALTH_CHECK=$(curl -s -o /dev/null -w "%{http_code}" https://api.photoshare.com/health)
if [ "$HEALTH_CHECK" = "200" ]; then
    echo "✅ Failover completed successfully"
    echo "API is responding from secondary region"
else
    echo "❌ Failover verification failed"
    echo "API health check returned: $HEALTH_CHECK"
    exit 1
fi

# Step 7: Send notifications
echo "7. Sending notifications..."
aws sns publish \
    --topic-arn "$FAILOVER_NOTIFICATION_TOPIC" \
    --subject "PhotoShare Failover Completed" \
    --message "Manual failover from $PRIMARY_REGION to $SECONDARY_REGION completed successfully at $(date)" \
    --region $SECONDARY_REGION

echo "=== Failover Completed ==="
echo "Services are now running in $SECONDARY_REGION"
echo "Monitor the application closely for the next 24 hours"
```

## Recovery Procedures

### Database Recovery

#### Point-in-Time Recovery
```bash
#!/bin/bash
# scripts/database-recovery.sh

set -e

RECOVERY_TIME=$1
DB_INSTANCE_ID="photoshare-primary"
RECOVERY_INSTANCE_ID="photoshare-recovery-$(date +%Y%m%d-%H%M%S)"

if [ -z "$RECOVERY_TIME" ]; then
    echo "Usage: $0 <recovery-time-utc>"
    echo "Example: $0 '2024-01-15 14:30:00'"
    exit 1
fi

echo "Starting point-in-time recovery to: $RECOVERY_TIME"

# Create new instance from point-in-time backup
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier $DB_INSTANCE_ID \
    --target-db-instance-identifier $RECOVERY_INSTANCE_ID \
    --restore-time "$RECOVERY_TIME" \
    --db-instance-class db.r6g.large \
    --multi-az \
    --storage-encrypted

echo "Waiting for recovery instance to be available..."
aws rds wait db-instance-available \
    --db-instance-identifier $RECOVERY_INSTANCE_ID

# Get new endpoint
NEW_ENDPOINT=$(aws rds describe-db-instances \
    --db-instance-identifier $RECOVERY_INSTANCE_ID \
    --query 'DBInstances[0].Endpoint.Address' \
    --output text)

echo "Recovery instance created: $RECOVERY_INSTANCE_ID"
echo "New endpoint: $NEW_ENDPOINT"

# Verify data integrity
echo "Verifying data integrity..."
psql -h $NEW_ENDPOINT -U photoshare_admin -d photoshare -c "
SELECT 
    (SELECT COUNT(*) FROM users) as user_count,
    (SELECT COUNT(*) FROM photos) as photo_count,
    (SELECT MAX(created_at) FROM photos) as latest_photo;
"

echo "Recovery completed. New instance: $RECOVERY_INSTANCE_ID"
echo "Manual verification required before switching traffic"
```

#### Application Data Recovery
```javascript
// scripts/application-recovery.js
const AWS = require('aws-sdk');

class ApplicationRecovery {
  constructor() {
    this.s3 = new AWS.S3({ region: 'us-east-1' });
    this.ssm = new AWS.SSM({ region: 'us-east-1' });
    this.secretsManager = new AWS.SecretsManager({ region: 'us-east-1' });
  }
  
  async restoreFromBackup(backupTimestamp) {
    console.log(`Starting application recovery from backup: ${backupTimestamp}`);
    
    try {
      await this.restoreSSMParameters(backupTimestamp);
      await this.restoreSecrets(backupTimestamp);
      await this.verifyConfiguration();
      
      console.log('Application recovery completed successfully');
    } catch (error) {
      console.error('Application recovery failed:', error);
      throw error;
    }
  }
  
  async restoreSSMParameters(backupTimestamp) {
    console.log('Restoring SSM parameters...');
    
    const backupObject = await this.s3.getObject({
      Bucket: 'photoshare-config-backups',
      Key: `ssm-parameters/${backupTimestamp}/parameters.json`
    }).promise();
    
    const backupData = JSON.parse(backupObject.Body.toString());
    
    for (const param of backupData.parameters) {
      try {
        await this.ssm.putParameter({
          Name: param.name,
          Value: param.value,
          Type: param.type,
          Description: param.description,
          Overwrite: true
        }).promise();
        
        console.log(`Restored parameter: ${param.name}`);
      } catch (error) {
        console.error(`Failed to restore parameter ${param.name}:`, error);
      }
    }
  }
  
  async restoreSecrets(backupTimestamp) {
    console.log('Restoring secrets...');
    
    const backupObject = await this.s3.getObject({
      Bucket: 'photoshare-config-backups',
      Key: `secrets/${backupTimestamp}/secrets.json`
    }).promise();
    
    const backupData = JSON.parse(backupObject.Body.toString());
    
    for (const secret of backupData.secrets) {
      try {
        // Try to update existing secret
        await this.secretsManager.updateSecret({
          SecretId: secret.name,
          SecretString: secret.value,
          Description: secret.description
        }).promise();
        
        console.log(`Restored secret: ${secret.name}`);
      } catch (error) {
        if (error.code === 'ResourceNotFoundException') {
          // Create new secret if it doesn't exist
          await this.secretsManager.createSecret({
            Name: secret.name,
            SecretString: secret.value,
            Description: secret.description
          }).promise();
          
          console.log(`Created secret: ${secret.name}`);
        } else {
          console.error(`Failed to restore secret ${secret.name}:`, error);
        }
      }
    }
  }
  
  async verifyConfiguration() {
    console.log('Verifying restored configuration...');
    
    // Verify critical parameters exist
    const criticalParams = [
      '/photoshare/database/endpoint',
      '/photoshare/redis/endpoint',
      '/photoshare/s3/bucket'
    ];
    
    for (const paramName of criticalParams) {
      try {
        await this.ssm.getParameter({ Name: paramName }).promise();
        console.log(`✅ Parameter verified: ${paramName}`);
      } catch (error) {
        console.error(`❌ Missing parameter: ${paramName}`);
        throw new Error(`Critical parameter missing: ${paramName}`);
      }
    }
    
    // Verify critical secrets exist
    const criticalSecrets = [
      'photoshare/database/credentials',
      'photoshare/jwt/secret'
    ];
    
    for (const secretName of criticalSecrets) {
      try {
        await this.secretsManager.getSecretValue({ SecretId: secretName }).promise();
        console.log(`✅ Secret verified: ${secretName}`);
      } catch (error) {
        console.error(`❌ Missing secret: ${secretName}`);
        throw new Error(`Critical secret missing: ${secretName}`);
      }
    }
    
    console.log('Configuration verification completed');
  }
}

// Usage
const recovery = new ApplicationRecovery();
const backupTimestamp = process.argv[2];

if (!backupTimestamp) {
  console.error('Usage: node application-recovery.js <backup-timestamp>');
  process.exit(1);
}

recovery.restoreFromBackup(backupTimestamp)
  .then(() => {
    console.log('Recovery process completed');
    process.exit(0);
  })
  .catch((error) => {
    console.error('Recovery process failed:', error);
    process.exit(1);
  });
```

## Communication Plan

### Incident Response Team

```
┌─────────────────────┬─────────────────────┬─────────────────────┐
│ Role                │ Primary Contact     │ Backup Contact      │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ Incident Commander  │ DevOps Lead         │ Engineering Manager │
│ Technical Lead      │ Senior Developer    │ Platform Engineer   │
│ Communications Lead │ Product Manager     │ Customer Success    │
│ Executive Sponsor   │ CTO                 │ VP Engineering      │
└─────────────────────┴─────────────────────┴─────────────────────┘
```

### Communication Templates

#### Internal Notification
```markdown
# Incident Alert: [SEVERITY] - [BRIEF DESCRIPTION]

**Incident ID**: INC-{timestamp}
**Severity**: Critical/High/Medium/Low
**Started**: {timestamp}
**Status**: Investigating/Identified/Monitoring/Resolved

## Impact
- Services Affected: {list}
- Users Affected: {estimated number}
- Business Impact: {revenue/reputation impact}

## Current Actions
- {action 1}
- {action 2}

## Next Update
Scheduled for: {time}

**Incident Commander**: {name}
**War Room**: {link/location}
```

#### Customer Communication
```markdown
# Service Status Update

We are currently experiencing issues with our photo sharing service that may impact your ability to upload and view photos.

**What's happening**: {brief technical explanation}
**Impact**: {user-facing impact}
**Timeline**: Started at {time}
**Resolution**: We are actively working to resolve this issue and expect service to be restored by {estimated time}.

We will provide updates every 30 minutes until resolution.

We apologize for any inconvenience.

- The PhotoShare Team
```

### Escalation Matrix

```
Level 1 (0-15 minutes)
├─ On-call Engineer
├─ Team Lead
└─ Automated monitoring alerts

Level 2 (15-30 minutes)
├─ Engineering Manager
├─ DevOps Team
└─ Product Manager

Level 3 (30-60 minutes)
├─ VP Engineering
├─ Customer Success Lead
└─ External vendor escalation

Level 4 (60+ minutes)
├─ CTO
├─ CEO
└─ PR/Legal team if needed
```

## Testing & Validation

### DR Testing Schedule

```
┌─────────────────┬─────────────┬─────────────┬─────────────────┐
│ Test Type       │ Frequency   │ Duration    │ Success Criteria│
├─────────────────┼─────────────┼─────────────┼─────────────────┤
│ Backup Restore  │ Monthly     │ 2 hours     │ 100% data       │
│ Failover Test   │ Quarterly   │ 4 hours     │ <15 min RTO     │
│ Full DR Exercise│ Annually    │ 8 hours     │ <5 min RPO      │
│ Runbook Review  │ Monthly     │ 1 hour      │ All steps valid │
└─────────────────┴─────────────┴─────────────┴─────────────────┘
```

### DR Test Script
```bash
#!/bin/bash
# scripts/dr-test.sh

set -e

TEST_TYPE=${1:-"backup-restore"}
TEST_ENV="dr-test"

echo "Starting DR test: $TEST_TYPE"

case $TEST_TYPE in
    "backup-restore")
        ./scripts/test-backup-restore.sh
        ;;
    "failover")
        ./scripts/test-failover.sh
        ;;
    "full-dr")
        ./scripts/test-full-dr.sh
        ;;
    *)
        echo "Unknown test type: $TEST_TYPE"
        exit 1
        ;;
esac

echo "DR test completed: $TEST_TYPE"
```

This comprehensive disaster recovery plan ensures PhotoShare can maintain business continuity and quickly recover from various disaster scenarios while meeting our strict RTO and RPO requirements.