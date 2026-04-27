=== BUIDL TITLE ===
Production RDS Aurora Serverless v2 + pgvector for AI-Native SaaS

=== DETAILS FIELD (the actual prompt) ===

You are an AWS infrastructure architect helping me deploy a production-ready PostgreSQL database optimized for AI/ML workloads using vector embeddings.

## OBJECTIVE
Deploy Amazon RDS Aurora Serverless v2 with pgvector extension for semantic search and RAG (Retrieval-Augmented Generation) applications. This setup auto-scales from 0.5 ACU to 16 ACU, supports 1536-dimensional embeddings (OpenAI ada-002 / AWS Titan compatible), and includes backup/recovery, monitoring, and cost controls.

## ARCHITECTURE DECISIONS

**Database Engine**: Aurora PostgreSQL 15.4 (pgvector 0.5.1 included)
**Compute**: Serverless v2 (min 0.5 ACU, max 16 ACU)
**Storage**: Aurora I/O-Optimized (better for high-throughput vector queries)
**Deployment**: Multi-AZ with 1 reader instance for HA
**Security**: Private subnet, TLS 1.2+, IAM authentication enabled

## STEP-BY-STEP DEPLOYMENT

### 1. Create VPC and Subnet Group (if not exists)

Create a DB subnet group spanning 3 Availability Zones:

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name ai-saas-db-subnet \
  --db-subnet-group-description "Private subnets for Aurora AI database" \
  --subnet-ids subnet-xxx subnet-yyy subnet-zzz \
  --tags Key=Project,Value=AI-SaaS Key=Environment,Value=Production
```

**Decision Point**: Use existing VPC with private subnets (10.0.32.0/20, 10.0.48.0/20, 10.0.64.0/20) or create new VPC? Existing VPC recommended if you have application servers already deployed.

### 2. Create Security Group with Least-Privilege Access

```bash
aws ec2 create-security-group \
  --group-name aurora-ai-sg \
  --description "Security group for Aurora pgvector database" \
  --vpc-id vpc-xxxxx

aws ec2 authorize-security-group-ingress \
  --group-id sg-xxxxx \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app-servers \
  --group-owner-id 123456789012
```

**Security Note**: NEVER allow 0.0.0.0/0 ingress on port 5432. Only allow traffic from application security group.

### 3. Create IAM Role for Enhanced Monitoring

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "monitoring.rds.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
```

Attach policy: `arn:aws:iam::aws:policy/service-role/AmazonRDSEnhancedMonitoringRole`

### 4. Create Aurora Serverless v2 Cluster

```bash
aws rds create-db-cluster \
  --db-cluster-identifier ai-saas-aurora-cluster \
  --engine aurora-postgresql \
  --engine-version 15.4 \
  --master-username postgres \
  --master-user-password $(openssl rand -base64 32) \
  --db-subnet-group-name ai-saas-db-subnet \
  --vpc-security-group-ids sg-xxxxx \
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "mon:04:00-mon:05:00" \
  --enable-cloudwatch-logs-exports '["postgresql"]' \
  --storage-encrypted \
  --kms-key-id arn:aws:kms:us-east-1:123456789012:key/xxxxx \
  --serverless-v2-scaling-configuration MinCapacity=0.5,MaxCapacity=16 \
  --database-name vector_db \
  --enable-iam-database-authentication \
  --tags Key=Project,Value=AI-SaaS Key=CostCenter,Value=Engineering
```

**Cost Optimization**: Set MinCapacity to 0.5 ACU ($0.12/hour) for dev environments. Production should use 1-2 ACU minimum to avoid cold-start latency.

### 5. Create Writer and Reader Instances

```bash
# Writer instance
aws rds create-db-instance \
  --db-instance-identifier ai-saas-aurora-writer \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --db-cluster-identifier ai-saas-aurora-cluster \
  --monitoring-interval 60 \
  --monitoring-role-arn arn:aws:iam::123456789012:role/RDSEnhancedMonitoring \
  --performance-insights-enabled \
  --performance-insights-retention-period 7

# Reader instance (for read-heavy vector searches)
aws rds create-db-instance \
  --db-instance-identifier ai-saas-aurora-reader \
  --db-instance-class db.serverless \
  --engine aurora-postgresql \
  --db-cluster-identifier ai-saas-aurora-cluster \
  --promotion-tier 1
```

**Wait for cluster to become available** (~10 minutes):

```bash
aws rds wait db-cluster-available --db-cluster-identifier ai-saas-aurora-cluster
```

### 6. Enable pgvector Extension

Connect to the database using psql:

```bash
psql "host=ai-saas-aurora-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com \
      port=5432 dbname=vector_db user=postgres sslmode=require"
```

Run these SQL commands:

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create vector index using HNSW (fast approximate nearest neighbor search)
CREATE TABLE embeddings (
  id BIGSERIAL PRIMARY KEY,
  content TEXT NOT NULL,
  embedding vector(1536),  -- OpenAI ada-002 / AWS Titan dimensions
  metadata JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Create HNSW index for fast similarity search (optimized for < 1M vectors)
CREATE INDEX ON embeddings USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- For > 1M vectors, use IVFFlat:
-- CREATE INDEX ON embeddings USING ivfflat (embedding vector_cosine_ops)
-- WITH (lists = 100);

-- Grant read-write access to application user
CREATE USER app_user WITH PASSWORD '{{SECURE_PASSWORD}}';
GRANT SELECT, INSERT, UPDATE, DELETE ON embeddings TO app_user;
GRANT USAGE, SELECT ON SEQUENCE embeddings_id_seq TO app_user;
```

**Index Choice Decision**:
- HNSW: Better recall, slower inserts. Use for < 1M vectors.
- IVFFlat: Faster inserts, needs tuning. Use for > 1M vectors.

### 7. Configure CloudWatch Alarms

```bash
# High CPU utilization alarm
aws cloudwatch put-metric-alarm \
  --alarm-name aurora-ai-high-cpu \
  --alarm-description "Alert if Aurora CPU > 80% for 5 minutes" \
  --metric-name CPUUtilization \
  --namespace AWS/RDS \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DBClusterIdentifier,Value=ai-saas-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts

# Serverless scaling alarm (hitting max capacity)
aws cloudwatch put-metric-alarm \
  --alarm-name aurora-ai-max-capacity \
  --alarm-description "Alert if Aurora ACU reaches max for 10+ minutes" \
  --metric-name ServerlessDatabaseCapacity \
  --namespace AWS/RDS \
  --statistic Average \
  --period 600 \
  --threshold 15 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DBClusterIdentifier,Value=ai-saas-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts

# Storage alarm (when to upgrade plan)
aws cloudwatch put-metric-alarm \
  --alarm-name aurora-ai-storage-high \
  --alarm-description "Alert if storage > 800 GB" \
  --metric-name VolumeBytesUsed \
  --namespace AWS/RDS \
  --statistic Average \
  --period 3600 \
  --threshold 858993459200 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --dimensions Name=DBClusterIdentifier,Value=ai-saas-aurora-cluster \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:ops-alerts
```

### 8. Enable Automated Backups and Point-in-Time Recovery

Already enabled with `--backup-retention-period 7`. To restore to a point in time:

```bash
aws rds restore-db-cluster-to-point-in-time \
  --source-db-cluster-identifier ai-saas-aurora-cluster \
  --db-cluster-identifier ai-saas-aurora-restored \
  --restore-to-time 2026-04-26T14:30:00Z \
  --use-latest-restorable-time
```

### 9. Set Up Budget Alerts

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget file://budget-config.json
```

**budget-config.json**:
```json
{
  "BudgetName": "Aurora-AI-Monthly",
  "BudgetLimit": {
    "Amount": "500",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST",
  "CostFilters": {
    "TagKeyValue": ["user:Project$AI-SaaS"]
  }
}
```

### 10. Validation Steps

Run these checks to confirm everything works:

```sql
-- Test pgvector is working
SELECT vector_dims('[1,2,3]'::vector);  -- Should return 3

-- Insert test embedding
INSERT INTO embeddings (content, embedding) VALUES
('test document', ARRAY_FILL(0, ARRAY[1536])::vector);

-- Test similarity search (cosine distance)
SELECT content, embedding <=> '[0.1, 0.2, ...]'::vector AS distance
FROM embeddings
ORDER BY distance LIMIT 5;
```

**Connection String for Application**:
```
postgresql://app_user:{{PASSWORD}}@ai-saas-aurora-cluster.cluster-xxxxx.us-east-1.rds.amazonaws.com:5432/vector_db?sslmode=require
```

=== CONTEXT & DOCUMENTATION FIELD ===

**Prerequisites**:
- AWS CLI v2.15+ configured with Administrator or PowerUser permissions
- Existing VPC with 3 private subnets across different AZs
- KMS key created for encryption at rest
- SNS topic for CloudWatch alarm notifications
- Understanding of vector embeddings and semantic search concepts

**Use Case**:
This prompt is for engineering teams building AI-powered SaaS applications that need:
- Semantic search over user documents
- RAG (Retrieval-Augmented Generation) chatbots with knowledge bases
- Recommendation engines using vector similarity
- Content deduplication using embedding distance

Typical customers: B2B SaaS startups with 10K-1M users, processing 100K-10M documents, running 1000-100K searches/day.

**Expected Outcome**:
- Aurora PostgreSQL cluster with pgvector extension enabled
- Auto-scaling from 0.5 to 16 ACU based on load
- Multi-AZ deployment with 1 writer + 1 reader instance
- TLS-encrypted connections with IAM authentication
- 7-day automated backups with point-in-time recovery
- CloudWatch alarms for CPU, capacity, and storage
- Estimated monthly cost: $54-$480/month depending on usage (0.5 ACU idle → 16 ACU sustained)

**Troubleshooting Tips**:

1. **"Extension pgvector does not exist"**
   - Ensure engine version is 15.4+ (pgvector included by default)
   - Run `SELECT * FROM pg_available_extensions WHERE name = 'vector';` to verify

2. **Slow vector queries (> 500ms)**
   - Check if HNSW/IVFFlat index is being used: `EXPLAIN ANALYZE SELECT ... ORDER BY embedding <=> ...`
   - Increase `m` parameter in HNSW index (higher = better recall, more memory)
   - For IVFFlat, tune `lists` parameter: `lists = SQRT(num_rows)`

3. **Connection timeouts from application**
   - Verify security group allows traffic from application SG
   - Check if database is in private subnet and application has VPC access
   - Enable VPC Flow Logs to debug network path

4. **High costs (> $500/month)**
   - Review ServerlessDatabaseCapacity metric — are you hitting max ACU constantly?
   - Consider reducing MinCapacity during low-traffic hours (use AWS Lambda scheduler)
   - Switch to Aurora I/O-Optimized if I/O costs exceed $100/month

5. **"Too many connections" error**
   - Aurora Serverless v2 max connections = (ACU × 1000). At 0.5 ACU = 500 connections
   - Implement connection pooling (PgBouncer on ECS or RDS Proxy)
   - Increase MinCapacity if application needs > 500 concurrent connections

=== AWS SERVICES & BEST PRACTICES FIELD ===

**Services used**:
- Amazon RDS Aurora Serverless v2 (database engine)
- Amazon VPC (network isolation)
- AWS KMS (encryption at rest)
- Amazon CloudWatch (monitoring and alarms)
- AWS Budgets (cost controls)
- Amazon SNS (alarm notifications)
- IAM (authentication and authorization)

**Well-Architected pillars addressed**:

1. **Security**:
   - Encryption at rest using AWS KMS
   - Encryption in transit with TLS 1.2+
   - IAM database authentication enabled
   - Private subnet deployment (no internet access)
   - Least-privilege security group rules
   - Automated security patching during maintenance windows

2. **Reliability**:
   - Multi-AZ deployment with automatic failover
   - Automated backups with 7-day retention
   - Point-in-time recovery capability
   - Read replica for high availability
   - CloudWatch alarms for proactive monitoring

3. **Performance Efficiency**:
   - Serverless auto-scaling (0.5-16 ACU)
   - Aurora I/O-Optimized for high-throughput workloads
   - HNSW index for fast vector searches (< 100ms)
   - Reader instance for read-heavy semantic search queries
   - Performance Insights enabled for query optimization

4. **Cost Optimization**:
   - Serverless v2 scales to 0.5 ACU during idle ($54/month minimum)
   - Aurora I/O-Optimized eliminates per-request I/O charges
   - 7-day backup retention (vs default 1-day)
   - Budget alerts at $500/month threshold
   - Right-sized MaxCapacity prevents runaway costs

5. **Operational Excellence**:
   - Infrastructure-as-code with AWS CLI commands
   - CloudWatch Logs integration for query monitoring
   - Enhanced Monitoring for OS-level metrics
   - Tagging strategy for cost allocation
   - Documented troubleshooting procedures

**Security controls included**:
- VPC isolation with private subnets
- Security group ingress restricted to application tier
- IAM database authentication
- TLS 1.2+ enforced
- KMS encryption at rest
- Automated security patching
- CloudWatch Logs for audit trail
- No public endpoints exposed

**Cost optimization measures**:
- Serverless v2 auto-scaling (pay only for ACU used)
- MinCapacity set to 0.5 ACU (lowest possible)
- Aurora I/O-Optimized pricing model
- 7-day backup retention (not 35-day)
- Budget alerts for cost anomalies
- Tagged resources for cost allocation
- Reader instance only when HA required (can remove for dev)

**Estimated Monthly Cost**:
- Idle (0.5 ACU, 24/7): ~$54
- Light usage (2 ACU average): ~$216
- Moderate (8 ACU average): ~$864 (exceeds budget, consider RDS PostgreSQL instead)
- Storage (100 GB): ~$10
- Backups (7 days × 100 GB): ~$2
- Total for typical startup: $70-$250/month
