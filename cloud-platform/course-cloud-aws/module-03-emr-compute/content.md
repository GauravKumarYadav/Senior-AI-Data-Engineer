# Module 3: Amazon EMR — Managed Spark & Compute

> **Scenario**: RetailMart's data lake is growing fast. Glue ETL handles daily batch transforms well, but the data science team needs **interactive Spark notebooks** for ML model training, the platform team wants **Kubernetes-native Spark** for their microservices cluster, and finance demands **cost visibility** down to the team level. Amazon EMR gives you managed Spark with the flexibility to choose your deployment model.

---

## Screen 1: EMR Deployment Models — Choose Your Fighter

Amazon EMR offers **three deployment models**, each with distinct operational characteristics. Picking the right one is one of the highest-leverage architecture decisions you'll make.

### The Three Models at a Glance

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        Amazon EMR Deployment Models                      │
├────────────────────┬────────────────────┬────────────────────────────────┤
│   EMR on EC2       │   EMR on EKS       │   EMR Serverless               │
│   (Traditional)    │   (Kubernetes)      │   (Fully Managed)              │
├────────────────────┼────────────────────┼────────────────────────────────┤
│                    │                    │                                │
│  ┌──────────────┐  │  ┌──────────────┐  │  ┌──────────────────────────┐  │
│  │ Master Node  │  │  │  EKS Cluster │  │  │  No cluster to manage   │  │
│  │   (m5.xl)    │  │  │  (shared)    │  │  │                          │  │
│  ├──────────────┤  │  ├──────────────┤  │  │  Submit job ──► Results  │  │
│  │ Core Nodes   │  │  │ Virtual      │  │  │                          │  │
│  │  (r5.2xl x5) │  │  │ Cluster     │  │  │  Auto-provisions         │  │
│  ├──────────────┤  │  │ (namespace)  │  │  │  workers as needed       │  │
│  │ Task Nodes   │  │  └──────────────┘  │  │                          │  │
│  │ (spot x10)   │  │                    │  └──────────────────────────┘  │
│  └──────────────┘  │                    │                                │
│                    │                    │                                │
│  You manage:       │  You manage:       │  You manage:                   │
│  - Instance types  │  - EKS cluster     │  - Nothing (just submit jobs)  │
│  - Scaling rules   │  - Namespaces      │                                │
│  - Bootstrap       │  - Pod templates   │  AWS manages:                  │
│  - Security groups │                    │  - Scaling                     │
│                    │  AWS manages:       │  - Patching                    │
│  AWS manages:      │  - Spark runtime   │  - Infrastructure              │
│  - Hadoop/Spark    │  - Job scheduling  │                                │
│  - YARN            │                    │                                │
│  - Patching        │                    │                                │
└────────────────────┴────────────────────┴────────────────────────────────┘
```

### Decision Matrix

| Criteria | EMR on EC2 | EMR on EKS | EMR Serverless |
|---|---|---|---|
| Operational overhead | High | Medium | None |
| Startup time | 5-10 min | 1-2 min | 30 sec - 1 min |
| Customization | Full (bootstrap) | Pod templates | Limited |
| Cost model | Per-instance-hour | Per-pod-hour | Per vCPU-hour + GB-hour |
| Multi-framework | Spark, Hive, Presto, HBase | Spark, Hive only | Spark, Hive only |
| GPU support | ✅ (P3/P4 instances) | ✅ (GPU nodes) | ❌ |
| Persistent clusters | ✅ | ✅ | ❌ (job-scoped) |
| Best for | Complex, long-running | K8s-native orgs | Simple batch/ad-hoc |

### 💡 Interview Insight

> **Q: "RetailMart has both a data engineering team running nightly batch jobs and a data science team doing ad-hoc exploration. How do you architect EMR?"**
>
> Hybrid approach: (1) **EMR Serverless** for the nightly batch ETL — zero cluster management, auto-scales to the job, shuts down when done. Cost-efficient for predictable, scheduled workloads. (2) **EMR on EC2** with a persistent cluster for the data sciethey need interactive notebooks (EMR Studio), custom Python libraries (bootstrap actions), and sometimes GPU instances for deep learning. Use a shared cluster with YARN capacity scheduler to isolate team workloads. (3) If the org already runs EKS, consider **EMR on EKS** to consolidate the Kubernetes footprint.

---

## Screen 2: EMR on EC2 — The Fully Customizable Workhorse

EMR on EC2 is the original deployment model — you define a cluster with master, core, and task nodes, and EMR provisions EC2 instances with Hadoop, Spark, Hive, and other frameworks pre-installed.

### Cluster Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    EMR Cluster on EC2                            │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                    MASTER NODE                           │    │
│  │  - YARN ResourceManager    - Spark Driver               │    │
│  │  - HDFS NameNode           - Hive Metastore (→ Glue)    │    │
│  │  - Ganglia / CloudWatch    - Livy REST server           │    │
│  │  Instance: m5.xlarge (or m6i.xlarge)                     │    │
│  └─────────────────────────────────────────────────────────┘    │
│                              │                                   │
│           ┌──────────────────┼──────────────────┐                │
│           ▼                  ▼                  ▼                │
│  ┌────────────────┐ ┌────────────────┐ ┌────────────────┐       │
│  │  CORE NODE 1   │ │  CORE NODE 2   │ │  CORE NODE 3   │      │
│  │  - YARN NM     │ │  - YARN NM     │ │  - YARN NM     │      │
│  │  - HDFS DN     │ │  - HDFS DN     │ │  - HDFS DN     │      │
│  │  - Spark Exec  │ │  - Spark Exec  │ │  - Spark Exec  │      │
│  │  r5.2xlarge    │ │  r5.2xlarge    │ │  r5.2xlarge    │      │
│  │  (On-Demand)   │ │  (On-Demand)   │ │  (On-Demand)   │      │
│  └────────────────┘ └────────────────┘ └────────────────┘       │
│                                                                  │
│  ┌────────────────┐ ┌────────────────┐ ┌──── ─ ─ ─ ─ ──┐       │
│  │  TASK NODE 1   │ │  TASK NODE 2   │ │  TASK NODE N   │       │
│  │  - YARN NM     │ │  - YARN NM     │ │  (auto-scaled) │      │
│  │  - Spark Exec  │ │  - Spark Exec  │ │  - Spot inst.  │      │
│  │  r5.2xlarge    │ │  r5.2xlarge    │ │  r5.2xlarge    │      │
│  │  (SPOT)        │ │  (SPOT)        │ │  (SPOT)        │      │
│  └────────────────┘ └────────────────┘ └──── ─ ─ ─ ─ ──┘       │
│                                                                  │
│  Storage: S3 (primary) + HDFS (shuffle/temp) + EBS (local)      │
└─────────────────────────────────────────────────────────────────┘
```

### Node Type Roles

| Node Type | Role | HDFS? | Can Scale? | Spot Safe? |
|---|---|---|---|---|
| Master | Cluster coordination | NameNode | No (single) | ❌ Never |
| Core | Compute + Storage | DataNode | Scale out only | ⚠️ Risky |
| Task | Compute only | No | Scale in/out | ✅ Perfect |

### Creating a Cluster via AWS CLI

```bash
aws emr create-cluster \
  --name "retailmart-spark-cluster" \
  --release-label "emr-7.1.0" \
  --applications Name=Spark Name=Hive Name=JupyterEnterpriseGateway \
  --use-default-roles \
  --ec2-attributes '{
    "KeyName": "retailmart-emr-key",
    "SubnetId": "subnet-0abc123def456",
    "EmrManagedMasterSecurityGroup": "sg-master-123",
    "EmrManagedSlaveSecurityGroup": "sg-worker-456"
  }' \
  --instance-groups '[
    {
      "InstanceGroupType": "MASTER",
      "InstanceType": "m6i.xlarge",
      "InstanceCount": 1
    },
    {
      "InstanceGroupType": "CORE",
      "InstanceType": "r6i.2xlarge",
      "InstanceCount": 5,
      "EbsConfiguration": {
        "EbsBlockDeviceConfigs": [{
          "VolumeSpecification": {"VolumeType": "gp3", "SizeInGB": 200},
          "VolumesPerInstance": 2
        }]
      }
    },
    {
      "InstanceGroupType": "TASK",
      "InstanceType": "r6i.2xlarge",
      "InstanceCount": 0,
      "Market": "SPOT",
      "BidPrice": "OnDemandPrice",
      "AutoScalingPolicy": {
        "Constraints": {"MinCapacity": 0, "MaxCapacity": 20},
        "Rules": [{
          "Name": "ScaleOutOnPendingContainers",
          "Action": {
            "SimpleScalingPolicyConfiguration": {
              "AdjustmentType": "CHANGE_IN_CAPACITY",
              "ScalingAdjustment": 5,
              "CoolDown": 300
            }
          },
          "Trigger": {
            "CloudWatchAlarmDefinition": {
              "MetricName": "ContainerPendingRatio",
              "Namespace": "AWS/ElasticMapReduce",
              "Statistic": "AVERAGE",
              "Period": 300,
              "EvaluationPeriods": 1,
              "Threshold": 0.75,
              "ComparisonOperator": "GREATER_THAN_OR_EQUAL"
            }
          }
        }]
      }
    }
  ]' \
  --configurations '[
    {
      "Classification": "spark-defaults",
      "Properties": {
        "spark.dynamicAllocation.enabled": "true",
        "spark.shuffle.service.enabled": "true",
        "spark.sql.adaptive.enabled": "true",
        "spark.sql.adaptive.coalescePartitions.enabled": "true",
        "spark.serializer": "org.apache.spark.serializer.KryoSerializer",
        "spark.sql.catalog.glue_catalog": "org.apache.iceberg.spark.SparkCatalog",
        "spark.sql.catalog.glue_catalog.catalog-impl": "org.apache.iceberg.aws.glue.GlueCatalog",
        "spark.sql.catalog.glue_catalog.warehouse": "s3://retailmart-data-lake-prod/iceberg/"
      }
    },
    {
      "Classification": "hive-site",
      "Properties": {
        "hive.metastore.client.factory.class":
          "com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory"
      }
    }
  ]' \
  --log-uri "s3://retailmart-logs/emr/" \
  --auto-termination-policy '{"IdleTimeout": 3600}' \
  --tags Key=Team,Value=DataEngineering Key=Environment,Value=Production \
  --region us-east-1
```

### 💡 Interview Insight

> **Q: "Why use Instance Fleets instead of Instance Groups?"**
>
> Instance **Groups** lock you into a single instance type per group. Instance **Fleets** let you specify a list of instance types per role (e.g., r5.2xlarge, r5.4xlarge, r6i.2xlarge, m5.2xlarge) and EMR picks the best available — critical for Spot availability. With Fleets, if your preferred Spot type is unavailable, EMR automatically falls back to the next type. For RetailMart's cost-sensitive task nodes, fleets reduce Spot interruptions by 3-5x vs. single-type groups.
>
> ```bash
> # Instance Fleet example
> --instance-fleets '[
>   {
>     "InstanceFleetType": "TASK",
>     "TargetSpotCapacity": 40,
>     "InstanceTypeConfigs": [
>       {"InstanceType": "r6i.2xlarge", "WeightedCapacity": 8},
>       {"InstanceType": "r5.2xlarge",  "WeightedCapacity": 8},
>       {"InstanceType": "r5a.2xlarge", "WeightedCapacity": 8},
>       {"InstanceType": "m6i.4xlarge", "WeightedCapacity": 16}
>     ],
>     "LaunchSpecifications": {
>       "SpotSpecification": {
>         "TimeoutDurationMinutes": 10,
>         "TimeoutAction": "SWITCH_TO_ON_DEMAND",
>         "AllocationStrategy": "capacity-optimized"
>       }
>     }
>   }
> ]'
> ```

---

## Screen 3: EMR on EKS — Spark on Kubernetes

EMR on EKS runs Spark workloads on your **existing Amazon EKS cluster** as Kubernetes pods. No separate EMR cluster needed — Spark shares nodes with your other microservices.

### Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                        Amazon EKS Cluster                        │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Namespace: retailmart-spark                              │    │
│  │  (EMR Virtual Cluster)                                    │    │
│  │                                                           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    │
│  │  │  Driver   │  │ Executor │  │ Executor │  ...           │   │
│  │  │  Pod      │  │  Pod 1   │  │  Pod 2   │               │    │
│  │  │  (1 vCPU) │  │ (4 vCPU) │  │ (4 vCPU) │               │   │
│  │  └──────────┘  └──────────┘  └──────────┘               │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Namespace: retailmart-api                                │    │
│  │  (Microservices — unrelated to EMR)                       │    │
│  │                                                           │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐               │    │
│  │  │  API Pod  │  │  API Pod  │  │ Worker   │               │   │
│  │  └──────────┘  └──────────┘  └──────────┘               │    │
│  └──────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Node Group: m6i.4xlarge x 10  (shared across namespaces)        │
└──────────────────────────────────────────────────────────────────┘
```

### Setting Up EMR on EKS

```bash
# Step 1: Register the EKS cluster with EMR
aws emr-containers create-virtual-cluster \
  --name retailmart-spark-virtual \
  --container-provider '{
    "id": "retailmart-eks-cluster",
    "type": "EKS",
    "info": {
      "eksInfo": {
        "namespace": "retailmart-spark"
      }
    }
  }'

# Step 2: Submit a Spark job
aws emr-containers start-job-run \
  --virtual-cluster-id "vc-abc123def456" \
  --name "daily-sales-aggregation" \
  --execution-role-arn "arn:aws:iam::123456789:role/EMRonEKSRole" \
  --release-label "emr-7.1.0-latest" \
  --job-driver '{
    "sparkSubmitJobDriver": {
      "entryPoint": "s3://retailmart-scripts/spark/daily_sales_agg.py",
      "entryPointArguments": ["--date", "2026-04-05"],
      "sparkSubmitParameters":
        "--conf spark.executor.instances=10
         --conf spark.executor.memory=8g
         --conf spark.executor.cores=4
         --conf spark.driver.memory=4g
         --conf spark.kubernetes.executor.podTemplateFile=s3://retailmart-scripts/pod-template.yaml"
    }
  }' \
  --configuration-overrides '{
    "applicationConfiguration": [{
      "classification": "spark-defaults",
      "properties": {
        "spark.dynamicAllocation.enabled": "true",
        "spark.dynamicAllocation.minExecutors": "2",
        "spark.dynamicAllocation.maxExecutors": "50",
        "spark.sql.adaptive.enabled": "true"
      }
    }],
    "monitoringConfiguration": {
      "s3MonitoringConfiguration": {
        "logUri": "s3://retailmart-logs/emr-on-eks/"
      },
      "cloudWatchMonitoringConfiguration": {
        "logGroupName": "/emr-on-eks/retailmart",
        "logStreamNamePrefix": "daily-sales"
      }
    }
  }'
```

### Pod Template for Fine-Grained Control

```yaml
# pod-template.yaml — controls executor pod specs
apiVersion: v1
kind: Pod
metadata:
  labels:
    team: data-engineering
    cost-center: retail-analytics
spec:
  nodeSelector:
    node-type: spark-compute        # Schedule on Spark-dedicated nodes
  tolerations:
    - key: "spark-workload"
      operator: "Equal"
      value: "true"
      effect: "NoSchedule"
  containers:
    - name: spark-kubernetes-executor
      resources:
        requests:
          cpu: "4"
          memory: "8Gi"
        limits:
          cpu: "4"
          memory: "10Gi"   # headroom for off-heap
      volumeMounts:
        - name: spark-local
          mountPath: /data
  volumes:
    - name: spark-local
      emptyDir:
        sizeLimit: 50Gi     # local shuffle storage
```

### 💡 Interview Insight

> **Q: "When does EMR on EKS make more sense than EMR on EC2?"**
>
> Choose EMR on EKS when: (1) Your organization **already runs EKS** and wants to consolidate infrastructure — no separate EMR clusters to manage. (2) You need **multi-tenant isolation** via Kubernetes namespaces and resource quotas. (3) You want **fast startup** (~30 seconds if nodes are pre-provisioned via Karpenter). (4) You need to **co-locate Spark with other services** (APIs, ML serving) on the same cluster. Choose EMR on EC2 when: you need non-Spark frameworks (HBase, Presto, Flink), need HDFS for high-performance shuffle, or want the simplicity of a purpose-built Spark cluster.

---

## Screen 4: EMR Serverless — Zero Infrastructure Management

EMR Serverless is the simplest option: you submit a Spark or Hive application, and AWS provisions compute automatically. No clusters, no nodes, no scaling policies. You pay per vCPU-hour and GB-hour of memory consumed.

### How EMR Serverless Works

```
┌────────────┐     ┌─────────────────────────┐     ┌────────────┐
│            │     │    EMR Serverless        │     │            │
│  Submit    │────►│                          │────►│  S3 Output │
│  Spark job │     │  1. Provisions workers   │     │  (results) │
│            │     │  2. Runs application     │     │            │
│            │◄────│  3. Scales dynamically   │◄────│            │
│  Results   │     │  4. Releases resources   │     │  S3 Input  │
│            │     │                          │     │  (source)  │
└────────────┘     └─────────────────────────┘     └────────────┘

Billing:
  vCPU-hour:  $0.052624
  GB-hour:    $0.0057785
  Storage-GB-hour: $0.000111
  
  Example: 10 vCPU, 40 GB, 1 hour
  = (10 × $0.053) + (40 × $0.006) = $0.53 + $0.24 = $0.77
```

### Creating and Running an EMR Serverless Application

```bash
# Step 1: Create an application (reusable container for jobs)
aws emr-serverless create-application \
  --name "retailmart-batch-processing" \
  --release-label "emr-7.1.0" \
  --type "SPARK" \
  --initial-capacity '{
    "DRIVER": {
      "workerCount": 1,
      "workerConfiguration": {
        "cpu": "2vCPU",
        "memory": "4GB",
        "disk": "20GB"
      }
    },
    "EXECUTOR": {
      "workerCount": 4,
      "workerConfiguration": {
        "cpu": "4vCPU",
        "memory": "8GB",
        "disk": "40GB"
      }
    }
  }' \
  --maximum-capacity '{
    "cpu": "200vCPU",
    "memory": "400GB",
    "disk": "2000GB"
  }' \
  --auto-start-configuration '{"enabled": true}' \
  --auto-stop-configuration '{"enabled": true, "idleTimeoutMinutes": 5}'

# Step 2: Submit a job run
aws emr-serverless start-job-run \
  --application-id "app-abc123" \
  --execution-role-arn "arn:aws:iam::123456789:role/EMRServerlessRole" \
  --name "daily-sales-etl-2026-04-05" \
  --job-driver '{
    "sparkSubmit": {
      "entryPoint": "s3://retailmart-scripts/spark/daily_sales_etl.py",
      "entryPointArguments": ["--date", "2026-04-05", "--output", "s3://retailmart-data-lake-prod/curated/daily_sales/"],
      "sparkSubmitParameters":
        "--conf spark.sql.adaptive.enabled=true
         --conf spark.sql.adaptive.coalescePartitions.enabled=true
         --conf spark.hadoop.hive.metastore.client.factory.class=com.amazonaws.glue.catalog.metastore.AWSGlueDataCatalogHiveClientFactory
         --conf spark.sql.catalog.glue_catalog=org.apache.iceberg.spark.SparkCatalog
         --conf spark.sql.catalog.glue_catalog.catalog-impl=org.apache.iceberg.aws.glue.GlueCatalog"
    }
  }' \
  --configuration-overrides '{
    "monitoringConfiguration": {
      "s3MonitoringConfiguration": {
        "logUri": "s3://retailmart-logs/emr-serverless/"
      }
    }
  }'
```

### Pre-Initialized Capacity (Warm Pools)

By default, EMR Serverless cold-starts in ~30 seconds. For latency-sensitive workloads, pre-initialize workers:

```bash
# Workers kept warm and ready — you pay even when idle
"initial-capacity": {
    "EXECUTOR": {
      "workerCount": 10,          # 10 executors always warm
      "workerConfiguration": {
        "cpu": "4vCPU",
        "memory": "8GB"
      }
    }
}

# Auto-stop releases them after idle timeout
"auto-stop-configuration": {
    "enabled": true,
    "idleTimeoutMinutes": 15      # Release after 15 min idle
}
```

### 💡 Interview Insight

> **Q: "EMR Serverless vs. Glue ETL — they both run Spark serverlessly. How do you choose?"**
>
> This is a top-tier interview question. Key differences:
>
> | Factor | EMR Serverless | Glue ETL |
> |---|---|---|
> | Spark version | Latest (3.5+) | Glue runtime (may lag) |
> | Custom JARs/libs | Full control | Limited (wheel files) |
> | DynamicFrame | ❌ | ✅ (Glue-native) |
> | Data Catalog integration | Config required | Built-in |
> | Job bookmarks | ❌ (DIY) | ✅ Built-in |
> | Pricing | vCPU + memory per second | DPU-hour |
> | Custom Spark configs | Full flexibility | Limited |
> | Startup time | ~30 sec | ~1-2 min |
>
> Rule of thumb: **Glue** for simple, catalog-centric ETL with bookmarks and crawlers. **EMR Serverless** for advanced Spark jobs needing custom configs, latest Spark features, or Iceberg/Delta Lake heavy usage. At RetailMart, we use Glue for CSV-to-Parquet ingestion and EMR Serverless for the complex daily aggregation pipeline that uses Iceberg merge operations.

---

## Screen 5: EMR Studio — Interactive Notebooks

EMR Studio provides **managed JupyterLab notebooks** that connect to EMR clusters (EC2, EKS, or Serverless). This is where RetailMart's data scientists do exploratory analysis and model prototyping.

### EMR Studio Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                        EMR Studio                            │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │                 JupyterLab Interface                    │  │
│  │                                                        │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐ │  │
│  │  │  Notebook 1   │  │  Notebook 2   │  │  Notebook 3  │ │  │
│  │  │  PySpark      │  │  SparkSQL     │  │  Python      │ │  │
│  │  │  (DS Team)    │  │  (Analysts)   │  │  (ML Team)   │ │  │
│  │  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘ │  │
│  │         │                 │                 │          │  │
│  └─────────┼─────────────────┼─────────────────┼──────────┘  │
│            │                 │                 │              │
│    ┌───────▼──────┐  ┌──────▼───────┐  ┌──────▼──────────┐  │
│    │ EMR on EC2   │  │ EMR on EC2   │  │ EMR Serverless  │  │
│    │ Cluster A    │  │ Cluster A    │  │ (on-demand)     │  │
│    │ (shared)     │  │ (same)       │  │                 │  │
│    └──────────────┘  └──────────────┘  └─────────────────┘  │
│                                                              │
│  Git Integration: CodeCommit / GitHub                        │
│  Storage: S3 (notebook files persisted)                      │
└──────────────────────────────────────────────────────────────┘
```

### Setting Up EMR Studio

```bash
# Create the Studio
aws emr create-studio \
  --name "RetailMart Analytics Studio" \
  --auth-mode SSO \
  --vpc-id "vpc-0abc123" \
  --subnet-ids "subnet-0abc123" "subnet-0def456" \
  --service-role "arn:aws:iam::123456789:role/EMRStudioServiceRole" \
  --user-role "arn:aws:iam::123456789:role/EMRStudioUserRole" \
  --workspace-security-group-id "sg-studio-ws" \
  --engine-security-group-id "sg-studio-engine" \
  --default-s3-location "s3://retailmart-emr-studio/notebooks/"

# Assign a user
aws emr create-studio-session-mapping \
  --studio-id "es-abc123" \
  --identity-type USER \
  --identity-id "data-scientist@retailmart.com" \
  --session-policy-arn "arn:aws:iam::123456789:policy/EMRStudioDataSciPolicy"
```

### Notebook Example: RetailMart Sales Forecasting

```python
# ── EMR Studio PySpark Notebook ────────────────────────────────
# Cell 1: Load data from Glue catalog
from pyspark.sql import functions as F
from pyspark.sql.window import Window

df = spark.sql("""
    SELECT store_id, day, SUM(total_amount) AS daily_revenue
    FROM glue_catalog.curated.daily_sales
    WHERE year = '2026' AND month IN ('01','02','03','04')
    GROUP BY store_id, day
""")

print(f"Records: {df.count():,}")
df.show(5)

# Cell 2: Feature engineering
window_7d = Window.partitionBy("store_id").orderBy("day").rowsBetween(-6, 0)
window_30d = Window.partitionBy("store_id").orderBy("day").rowsBetween(-29, 0)

df_features = (
    df.withColumn("rolling_7d_avg", F.avg("daily_revenue").over(window_7d))
      .withColumn("rolling_30d_avg", F.avg("daily_revenue").over(window_30d))
      .withColumn("day_of_week", F.dayofweek("day"))
      .withColumn("is_weekend", F.when(F.col("day_of_week").isin(1, 7), 1).otherwise(0))
      .withColumn("revenue_lag_1", F.lag("daily_revenue", 1).over(
          Window.partitionBy("store_id").orderBy("day")))
      .withColumn("revenue_lag_7", F.lag("daily_revenue", 7).over(
          Window.partitionBy("store_id").orderBy("day")))
)

# Cell 3: Write features for ML training
df_features.write \
    .mode("overwrite") \
    .partitionBy("store_id") \
    .parquet("s3://retailmart-data-lake-prod/ml-features/sales-forecast/")
```

### 💡 Interview Insight

> **Q: "How do you manage notebook dependencies and reproducibility across team members?"**
>
> (1) **Git integration** — EMR Studio supports linking to CodeCommit or GitHub repos. Notebooks are versioned alongside code. (2) **Cluster-scoped libraries** — use `--configurations` to install packages cluster-wide via bootstrap or pip at startup. (3) **Notebook-scoped libraries** — use `%pip install` in a cell for ad-hoc packages. (4) **Environment files** — commit a `requirements.txt` alongside notebooks and install in the first cell. (5) For production, always promote notebooks to **standalone Spark scripts** submitted via Step Functions or Airflow — notebooks are for exploration, not production pipelines.

---

## Screen 6: Bootstrap Actions — Customizing Your Cluster

Bootstrap actions run **shell scripts on every node** during cluster startup, before any applications launch. Use them to install custom software, configure system settings, or download model artifacts.

### Common Bootstrap Actions for RetailMart

```bash
#!/bin/bash
# bootstrap_retailmart.sh — runs on EVERY node (master + core + task)

set -euo pipefail

# ── Install Python packages ─────────────────────────────────────
sudo pip3 install \
    pandas==2.2.1 \
    pyarrow==15.0.0 \
    scikit-learn==1.4.0 \
    boto3==1.34.50 \
    great-expectations==0.18.0 \
    delta-spark==3.1.0

# ── Install system utilities ────────────────────────────────────
sudo yum install -y jq htop tmux

# ── Download ML model artifacts (master only) ──────────────────
if grep -q isMaster /mnt/var/lib/info/instance.json; then
    aws s3 cp s3://retailmart-models/demand-forecast/latest/ \
        /opt/retailmart/models/ --recursive
    echo "Master node: model artifacts downloaded"
fi

# ── Configure Spark defaults ────────────────────────────────────
cat >> /etc/spark/conf/spark-defaults.conf << 'EOF'
spark.sql.sources.partitionOverwriteMode=dynamic
spark.sql.parquet.int96RebootLegacyFormat=true
spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version=2
EOF

# ── Set timezone ────────────────────────────────────────────────
sudo timedatectl set-timezone America/Chicago

echo "Bootstrap complete on $(hostname)"
```

### Applying Bootstrap Actions

```bash
aws emr create-cluster \
  --name "retailmart-spark-cluster" \
  --release-label "emr-7.1.0" \
  --bootstrap-actions '[
    {
      "Name": "Install RetailMart dependencies",
      "Path": "s3://retailmart-scripts/bootstrap/bootstrap_retailmart.sh",
      "Args": ["--env", "production"]
    },
    {
      "Name": "Configure CloudWatch agent",
      "Path": "s3://retailmart-scripts/bootstrap/install_cloudwatch_agent.sh"
    }
  ]' \
  ...
```

### Bootstrap vs. Custom AMI vs. Docker (EMR on EKS)

| Approach | Startup Impact | Maintenance | Flexibility |
|---|---|---|---|
| Bootstrap actions | +2-5 min per script | Update script in S3 | High (any shell cmd) |
| Custom AMI | No impact (pre-baked) | Rebuild AMI for changes | Highest |
| Docker image (EKS) | Pull time (~30 sec) | Rebuild image | High (containerized) |
| Cluster config (`--configurations`) | None | JSON change | Spark/Hadoop config only |

### 💡 Interview Insight

> **Q: "Your bootstrap script installs 20 Python packages and adds 5 minutes to cluster startup. How do you speed this up?"**
>
> Three approaches: (1) **Custom AMI** — pre-bake all packages into the AMI image. Cluster startup returns to baseline. Downside: you must rebuild the AMI when dependencies change. (2) **Cluster-scoped pip from S3** — host a private PyPI mirror on S3 and pip install from there (faster than public PyPI). (3) For EMR on EKS, use a **custom Docker image** with all dependencies — fastest startup, most portable. (4) Use `--configurations` to set `spark.jars.packages` for JAR dependencies — pulled from Maven in parallel. At RetailMart, we use a custom AMI for production clusters (stable dependencies) and bootstrap scripts for dev clusters (fast iteration).

---

## Screen 7: EMR vs. Glue — The Decision Framework

This is arguably the **most common interview question** in AWS data engineering. Here's the definitive framework:

### Decision Tree

```
                    ┌─────────────────────────┐
                    │  Do you need Spark?      │
                    └───────────┬─────────────┘
                           Yes  │  No
                    ┌───────────┘  └──────────────────┐
                    │                                  │
              ┌─────▼──────────┐              ┌───────▼────────┐
              │ Complex job?   │              │ Athena SQL     │
              │ Custom libs?   │              │ or Glue Python │
              │ Fine-tuning?   │              │ Shell          │
              └────┬───────────┘              └────────────────┘
              Yes  │  No
        ┌──────────┘  └───────────────┐
        │                             │
  ┌─────▼──────────┐          ┌──────▼─────────┐
  │ Need persistent │          │ AWS Glue ETL   │
  │ cluster or GPU? │          │ (Serverless    │
  │ HBase/Presto?   │          │  Spark)        │
  └────┬─────────┬──┘          │                │
   Yes │     No  │             │ - Bookmarks    │
       │         │             │ - DynamicFrame │
 ┌─────▼───┐ ┌──▼──────────┐  │ - Crawlers     │
 │ EMR on  │ │ EMR         │  └────────────────┘
 │ EC2     │ │ Serverless  │
 │         │ │ or EMR on   │
 │         │ │ EKS         │
 └─────────┘ └─────────────┘
```

### Detailed Comparison

| Factor | AWS Glue ETL | EMR on EC2 | EMR Serverless |
|---|---|---|---|
| **Provisioning** | Serverless | You manage cluster | Serverless |
| **Startup time** | ~1-2 min | 5-10 min | ~30 sec |
| **Spark version** | Glue runtime | You choose | You choose |
| **Custom libraries** | Wheel files, JARs | Full (bootstrap) | JARs, custom images |
| **Data Catalog** | Native integration | Config required | Config required |
| **Job bookmarks** | ✅ Built-in | ❌ DIY | ❌ DIY |
| **DynamicFrame** | ✅ | ❌ | ❌ |
| **Interactive notebooks** | Glue Studio | EMR Studio | Athena Spark |
| **Non-Spark frameworks** | ❌ | ✅ Hive, Presto, HBase | Hive only |
| **Cost (small job, 10 DPU-hr)** | ~$4.40 | ~$8-15 (+ EC2) | ~$3-5 |
| **Cost (large job, 100 DPU-hr)** | ~$44 | ~$30-50 (spot) | ~$30-50 |
| **Max concurrency** | 200 jobs/account | Cluster-limited | Application-limited |
| **GPU support** | ❌ | ✅ | ❌ |
| **Spot instances** | ❌ (auto-managed) | ✅ Full control | ❌ (auto-managed) |

### RetailMart's Hybrid Architecture

```
┌───────────────────────────────────────────────────────────────────┐
│                 RetailMart Compute Strategy                       │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  INGESTION LAYER (Glue ETL)                                 │  │
│  │  • CSV/JSON → Parquet conversion                            │  │
│  │  • Schema validation with DynamicFrame                      │  │
│  │  • Incremental with job bookmarks                           │  │
│  │  • 15 jobs, 2-10 DPU each, daily schedule                   │  │
│  │  • Monthly cost: ~$800                                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  TRANSFORM LAYER (EMR Serverless)                           │  │
│  │  • Complex aggregations, Iceberg merges                     │  │
│  │  • Custom Python ML feature pipelines                       │  │
│  │  • Latest Spark 3.5 with AQE                                │  │
│  │  • 5 jobs, 20-100 vCPU each, daily                          │  │
│  │  • Monthly cost: ~$1,200                                    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  EXPLORATION LAYER (EMR on EC2 + Studio)                    │  │
│  │  • Interactive notebooks for data science                   │  │
│  │  • ML model training (GPU instances available)              │  │
│  │  • Shared cluster with YARN queues per team                 │  │
│  │  • Persistent cluster, business hours only                  │  │
│  │  • Monthly cost: ~$3,500                                    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                              │                                    │
│                              ▼                                    │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  QUERY LAYER (Athena)                                       │  │
│  │  • Ad-hoc SQL from analysts and BI tools                    │  │
│  │  • Federated queries to DynamoDB, RDS                       │  │
│  │  • Partition projection, Iceberg time travel                │  │
│  │  • Monthly cost: ~$400                                      │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  Total monthly compute cost: ~$5,900                              │
└───────────────────────────────────────────────────────────────────┘
```

### 💡 Interview Insight

> **Q: "Your team currently uses Glue for everything. A new requirement needs Spark 3.5's MERGE INTO with Iceberg, but Glue only supports Spark 3.3. What do you do?"**
>
> This is a real scenario many teams hit. Options: (1) **EMR Serverless** — supports latest Spark, still serverless, minimal migration effort. Move the specific Iceberg-heavy jobs to EMR Serverless while keeping simple ingestion on Glue. (2) Wait for Glue to support Spark 3.5 — risky if timeline is tight. (3) EMR on EC2 transient cluster — more operational overhead but full control. Recommendation: option (1) is the pragmatic hybrid approach. At RetailMart, we migrated 3 out of 20 jobs to EMR Serverless for Iceberg support and kept the rest on Glue.

---

## Screen 8: Cost Optimization — Spot, Auto-Scaling & Right-Sizing

Cost is the #1 concern with EMR. A misconfigured cluster can cost 10x more than necessary.

### Spot Instance Strategy

```
┌───────────────────────────────────────────────────────────────────┐
│                    EMR Spot Instance Strategy                      │
│                                                                   │
│  MASTER NODE:  Always On-Demand (never Spot)                      │
│  ──────────── If master dies, the entire cluster dies.            │
│                                                                   │
│  CORE NODES:   On-Demand (with some Spot if using Instance Fleets)│
│  ─────────── Core nodes hold HDFS data — Spot termination =      │
│               potential data loss. Use EBS for HDFS if mixing.    │
│                                                                   │
│  TASK NODES:   100% Spot (ideal target)                           │
│  ──────────── Task nodes are compute-only — no HDFS data.         │
│               Spot termination = lost in-progress tasks only.     │
│               Spark retries automatically.                        │
│                                                                   │
│  Savings: Spot is typically 60-90% cheaper than On-Demand         │
│                                                                   │
│  Example:                                                         │
│    On-Demand r6i.2xlarge: $0.504/hr                               │
│    Spot r6i.2xlarge:      $0.151/hr (70% savings)                 │
│    10 task nodes × 8 hrs = $12.08 vs $40.32                       │
└───────────────────────────────────────────────────────────────────┘
```

### Managed Auto-Scaling (Recommended)

```bash
aws emr put-managed-scaling-policy \
  --cluster-id "j-abc123def456" \
  --managed-scaling-policy '{
    "ComputeLimits": {
      "UnitType": "InstanceFleetUnits",
      "MinimumCapacityUnits": 10,
      "MaximumCapacityUnits": 100,
      "MaximumOnDemandCapacityUnits": 20,
      "MaximumCoreCapacityUnits": 20
    }
  }'
```

This tells EMR: scale between 10-100 units, keep at most 20 On-Demand (rest Spot), and limit core nodes to 20 (scale tasks beyond that).

### Right-Sizing Checklist

| Metric | Target | Action if Off |
|---|---|---|
| YARN Memory Utilization | 70-85% | Scale down if < 50%, up if > 90% |
| CPU Utilization | 60-80% | Switch instance family (compute vs. memory) |
| Shuffle Spill to Disk | < 5% | Increase memory per executor |
| GC Time | < 10% of task time | Increase executor memory or tune GC |
| Task Straggler Ratio | < 1.5x median | Fix data skew, increase parallelism |
| S3 Throttling (503s) | 0 | Add prefix diversity, reduce concurrency |

### Transient vs. Persistent Clusters

```
Transient Cluster (cost-optimal for batch):
  ┌──────┐     ┌─────────────┐     ┌──────┐
  │Create│────►│  Run ETL     │────►│Termin│
  │      │     │  job(s)      │     │ -ate │
  └──────┘     └─────────────┘     └──────┘
  6:00 AM       6:10 - 7:30 AM      7:35 AM
  
  Cost: 1.5 hours × cluster cost = $$
  
Persistent Cluster (for interactive/multi-job):
  ┌──────┐     ┌────────────────────────────────────────┐     ┌──────┐
  │Create│────►│  Multiple jobs, notebooks, queries      │────►│Termin│
  │      │     │  throughout business hours               │     │ -ate │
  └──────┘     └────────────────────────────────────────┘     └──────┘
  8:00 AM       8:05 AM ──────────────────── 6:00 PM         6:05 PM
  
  Cost: 10 hours × cluster cost = $$$$$
  Optimize: auto-termination after 1 hour idle
```

```bash
# Auto-terminate after 1 hour of no activity
aws emr modify-cluster \
  --cluster-id "j-abc123def456" \
  --auto-termination-policy '{"IdleTimeout": 3600}'
```

### 💡 Interview Insight

> **Q: "Your nightly EMR batch job uses 50 r5.2xlarge On-Demand instances for 3 hours. How would you cut costs by 70%?"**
>
> Layered approach: (1) **Spot instances for task nodes** — move 40 of 50 instances to task nodes on Spot (keep 10 as core On-Demand). Spot savings: ~70% on those 40 nodes. (2) **Instance Fleets** — specify 5-6 instance types for Spot diversity and `capacity-optimized` allocation. (3) **Right-size** — check YARN utilization. If memory utilization is 40%, switch to m5.2xlarge (less memory, cheaper) or reduce instance count. (4) **Graviton instances** — r6g/m6g are 20% cheaper and 40% better price-performance than Intel equivalents. (5) **Auto-scaling** — start with 15 nodes and scale to 50 only during peak shuffle phases. Combined, these easily achieve 70%+ savings.

---

## Screen 9: Module 3 Quiz & Key Takeaways

### Quiz

**Q1.** RetailMart's data science team needs interactive PySpark notebooks with GPU support for training deep learning models on sales data. Which EMR deployment is MOST appropriate?

- A) EMR Serverless
- B) EMR on EKS
- C) EMR on EC2 with EMR Studio
- D) Glue Interactive Sessions

**Answer: C.** EMR on EC2 is the only model that supports GPU instances (P3/P4 families) natively. EMR Studio provides managed JupyterLab notebooks. EMR Serverless does not support GPUs. EMR on EKS could work with GPU-enabled nodes but adds Kubernetes complexity for a data science use case.

---

**Q2.** You're running a nightly batch job on EMR on EC2. The cluster has 5 core nodes (On-Demand) and 20 task nodes (Spot). An overnight Spot interruption terminates 15 task nodes mid-job. What happens?

- A) The entire job fails and must restart
- B) YARN reschedules the lost tasks on remaining nodes; job completes slower
- C) HDFS data is lost on the terminated nodes
- D) The master node terminates the cluster

**Answer: B.** Task nodes run compute only — no HDFS data. When Spot instances are terminated, YARN detects the lost containers and reschedules those tasks on the remaining 5 core + 5 surviving task nodes. The job completes but takes longer. If auto-scaling is enabled, new Spot nodes may launch to replace the lost ones.

---

**Q3.** A Glue ETL job running daily at 2 AM processes 50 GB of incremental data using bookmarks. The same logic could run on EMR Serverless. Which factors favor keeping it on Glue? (Select TWO)

- A) Glue's built-in job bookmarks eliminate DIY checkpointing
- B) Glue is always cheaper than EMR Serverless
- C) Glue's native Data Catalog integration simplifies reads/writes
- D) Glue supports more recent Spark versions
- E) Glue provides GPU support for ML workloads

**Answer: A and C.** Job bookmarks (A) and native catalog integration (C) are genuine Glue advantages that reduce code complexity. Glue is NOT always cheaper (B depends on job size), does NOT always have newer Spark (D — EMR typically leads), and does NOT support GPUs (E).

---

**Q4.** RetailMart wants to reduce EMR cluster startup time from 8 minutes to under 1 minute. Which approach achieves this?

- A) Use a larger master node instance type
- B) Switch to EMR Serverless with pre-initialized capacity
- C) Add more bootstrap actions
- D) Use Instance Fleets instead of Instance Groups

**Answer: B.** EMR Serverless with pre-initialized capacity keeps workers warm and ready, achieving sub-minute startup. EMR on EC2 always requires 5-10 minutes for instance provisioning + bootstrap + service startup. Larger master nodes (A) don't help. More bootstrap actions (C) make it worse. Instance Fleets (D) help with Spot availability but not startup time.

---

**Q5.** Your EMR cluster's CloudWatch metrics show YARN memory utilization at 35% and CPU at 20% across 20 r5.4xlarge instances. What should you do?

- A) Scale up to larger instances for better performance
- B) Right-size: reduce to ~8 instances or switch to smaller r5.xlarge
- C) Add more task nodes to improve parallelism
- D) Nothing — headroom is good for burst capacity

**Answer: B.** 35% memory and 20% CPU means massive over-provisioning. You're paying for 20 instances but using ~7 instances' worth of resources. Right-size by either reducing instance count to 8-10 or switching to smaller instance types (r5.xlarge or m5.xlarge). Option D wastes money unless you have known burst patterns.

---

### 🔑 Key Takeaways

1. **Three deployment models, one framework**: EMR on EC2 (maximum control), EMR on EKS (Kubernetes-native), EMR Serverless (zero ops). Pick based on operational maturity and requirements.

2. **Spot instances on task nodes are free money**: 60-90% savings with minimal risk. Never use Spot for master nodes. Use Instance Fleets for Spot diversity.

3. **EMR vs. Glue is not either/or**: Use both. Glue for catalog-integrated ingestion with bookmarks. EMR Serverless for complex transforms with latest Spark. EMR on EC2 for interactive exploration and GPU workloads.

4. **Right-sizing is the highest-ROI optimization**: Check YARN memory and CPU utilization before anything else. Most clusters are 2-3x over-provisioned.

5. **Bootstrap actions are powerful but add startup time**: For production, prefer custom AMIs. For dev, bootstrap scripts are fine.

6. **EMR Studio democratizes Spark**: Data scientists get JupyterLab without managing infrastructure. But always promote notebooks to production scripts.

7. **Auto-termination is mandatory**: Every persistent cluster should have an idle timeout. Forgotten clusters are the #1 EMR cost leak.

8. **Transient clusters for batch, persistent for interactive**: Don't keep a cluster running 24/7 for a 2-hour nightly job.
