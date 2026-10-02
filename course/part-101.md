# Part 101: Cloud Deployment (AWS/GCP)

## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- Deploy Python apps ไปยัง AWS ECS และ GCP Cloud Run
- ใช้ Managed Databases (RDS, Cloud SQL)
- จัดการ File Storage ด้วย S3 และ GCS
- จัดการ Environment Configuration อย่างปลอดภัย
- ใช้ Terraform สำหรับ Infrastructure as Code

---

## 1. AWS Deployment

### Dockerfile สำหรับ Production

```dockerfile
# Dockerfile.production
FROM python:3.11-slim AS builder

WORKDIR /app

# ติดตั้ง build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# Production stage
FROM python:3.11-slim

WORKDIR /app

# ติดตั้ง runtime dependencies เท่านั้น
RUN apt-get update && apt-get install -y \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Copy installed packages จาก builder
COPY --from=builder /root/.local /root/.local

# Copy application code
COPY . .

# สร้าง non-root user
RUN useradd -m -u 1000 appuser && chown -R appuser:appuser /app
USER appuser

ENV PATH=/root/.local/bin:$PATH
ENV PYTHONPATH=/app
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "main:app", "-k", "uvicorn.workers.UvicornWorker", \
     "-w", "4", "-b", "0.0.0.0:8000", "--timeout", "60", \
     "--access-logfile", "-", "--error-logfile", "-"]
```

### AWS ECS Task Definition

```python
# aws/ecs_deployment.py
"""
AWS ECS (Elastic Container Service) Deployment
สำหรับ containerized Python applications
"""
import boto3
import json
import os
from typing import Optional


class ECSDeployer:
    """Deploy Python app ไปยัง AWS ECS"""
    
    def __init__(self, region: str = "ap-southeast-1"):
        self.region = region
        self.ecs = boto3.client("ecs", region_name=region)
        self.ecr = boto3.client("ecr", region_name=region)
        self.logs = boto3.client("logs", region_name=region)
    
    def create_task_definition(
        self,
        family: str,
        image_uri: str,
        cpu: int = 256,
        memory: int = 512,
        env_vars: dict = None,
        secrets: dict = None,
        log_group: str = None
    ) -> dict:
        """สร้าง ECS Task Definition"""
        
        container_def = {
            "name": family,
            "image": image_uri,
            "essential": True,
            "portMappings": [
                {
                    "containerPort": 8000,
                    "protocol": "tcp"
                }
            ],
            "environment": [
                {"name": k, "value": v}
                for k, v in (env_vars or {}).items()
            ],
            "secrets": [
                {
                    "name": k,
                    "valueFrom": v  # ARN ของ Secrets Manager
                }
                for k, v in (secrets or {}).items()
            ],
            "healthCheck": {
                "command": ["CMD-SHELL", "curl -f http://localhost:8000/health || exit 1"],
                "interval": 30,
                "timeout": 10,
                "retries": 3,
                "startPeriod": 60
            },
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": log_group or f"/ecs/{family}",
                    "awslogs-region": self.region,
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
        
        response = self.ecs.register_task_definition(
            family=family,
            networkMode="awsvpc",
            requiresCompatibilities=["FARGATE"],
            cpu=str(cpu),
            memory=str(memory),
            executionRoleArn=f"arn:aws:iam::{self._get_account_id()}:role/ecsTaskExecutionRole",
            taskRoleArn=f"arn:aws:iam::{self._get_account_id()}:role/ecsTaskRole",
            containerDefinitions=[container_def]
        )
        
        return response["taskDefinition"]
    
    def deploy_service(
        self,
        cluster: str,
        service_name: str,
        task_definition: str,
        desired_count: int = 2,
        subnet_ids: list = None,
        security_group_ids: list = None,
        target_group_arn: str = None
    ) -> dict:
        """Deploy service ไปยัง ECS cluster"""
        
        network_config = {
            "awsvpcConfiguration": {
                "subnets": subnet_ids or [],
                "securityGroups": security_group_ids or [],
                "assignPublicIp": "DISABLED"
            }
        }
        
        load_balancers = []
        if target_group_arn:
            load_balancers = [{
                "targetGroupArn": target_group_arn,
                "containerName": service_name,
                "containerPort": 8000
            }]
        
        # ตรวจสอบว่า service มีอยู่แล้วหรือไม่
        try:
            existing = self.ecs.describe_services(
                cluster=cluster,
                services=[service_name]
            )
            
            if existing["services"] and existing["services"][0]["status"] != "INACTIVE":
                # อัพเดท existing service
                response = self.ecs.update_service(
                    cluster=cluster,
                    service=service_name,
                    taskDefinition=task_definition,
                    desiredCount=desired_count,
                    networkConfiguration=network_config,
                    forceNewDeployment=True,
                    deploymentConfiguration={
                        "maximumPercent": 200,
                        "minimumHealthyPercent": 100,
                        "deploymentCircuitBreaker": {
                            "enable": True,
                            "rollback": True
                        }
                    }
                )
                print(f"Updated service: {service_name}")
                return response["service"]
        except Exception:
            pass
        
        # สร้าง service ใหม่
        response = self.ecs.create_service(
            cluster=cluster,
            serviceName=service_name,
            taskDefinition=task_definition,
            desiredCount=desired_count,
            launchType="FARGATE",
            networkConfiguration=network_config,
            loadBalancers=load_balancers,
            deploymentConfiguration={
                "maximumPercent": 200,
                "minimumHealthyPercent": 100,
                "deploymentCircuitBreaker": {
                    "enable": True,
                    "rollback": True
                }
            },
            enableExecuteCommand=True  # สำหรับ debugging
        )
        
        print(f"Created service: {service_name}")
        return response["service"]
    
    def _get_account_id(self) -> str:
        """ดึง AWS Account ID"""
        sts = boto3.client("sts")
        return sts.get_caller_identity()["Account"]
    
    def wait_for_deployment(self, cluster: str, service_name: str, timeout: int = 300):
        """รอให้ deployment เสร็จ"""
        print(f"Waiting for deployment of {service_name}...")
        
        waiter = self.ecs.get_waiter("services_stable")
        waiter.wait(
            cluster=cluster,
            services=[service_name],
            WaiterConfig={"Delay": 15, "MaxAttempts": timeout // 15}
        )
        
        print(f"Deployment of {service_name} completed!")


# GitHub Actions CI/CD Pipeline
GITHUB_ACTIONS_WORKFLOW = """
# .github/workflows/deploy.yml
name: Deploy to AWS ECS

on:
  push:
    branches: [main]

env:
  AWS_REGION: ap-southeast-1
  ECR_REGISTRY: ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-southeast-1.amazonaws.com
  ECR_REPOSITORY: python-app
  ECS_CLUSTER: production-cluster
  ECS_SERVICE: python-app-service

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/ -v --cov=. --cov-report=xml
      - uses: codecov/codecov-action@v3

  deploy:
    needs: test
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}
      
      - name: Login to Amazon ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2
      
      - name: Build, tag, and push Docker image
        run: |
          IMAGE_TAG=${{ github.sha }}
          docker build -f Dockerfile.production -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "IMAGE_URI=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT
        id: build-image
      
      - name: Deploy to ECS
        run: |
          python aws/deploy.py \\
            --cluster $ECS_CLUSTER \\
            --service $ECS_SERVICE \\
            --image ${{ steps.build-image.outputs.IMAGE_URI }}
"""
```

### AWS RDS (Managed PostgreSQL)

```python
# aws/rds_manager.py
import boto3
import os
from typing import Optional


class RDSManager:
    """จัดการ AWS RDS instances"""
    
    def __init__(self, region: str = "ap-southeast-1"):
        self.rds = boto3.client("rds", region_name=region)
        self.region = region
    
    def create_db_instance(
        self,
        db_identifier: str,
        db_name: str,
        username: str,
        password: str,
        instance_class: str = "db.t3.micro",
        engine: str = "postgres",
        engine_version: str = "15.4",
        storage_gb: int = 20,
        subnet_group: str = None,
        security_groups: list = None,
        multi_az: bool = True
    ) -> dict:
        """สร้าง RDS instance"""
        
        response = self.rds.create_db_instance(
            DBInstanceIdentifier=db_identifier,
            DBName=db_name,
            MasterUsername=username,
            MasterUserPassword=password,
            DBInstanceClass=instance_class,
            Engine=engine,
            EngineVersion=engine_version,
            AllocatedStorage=storage_gb,
            StorageType="gp3",
            StorageEncrypted=True,
            MultiAZ=multi_az,
            AutoMinorVersionUpgrade=True,
            BackupRetentionPeriod=7,  # 7 วัน backup
            PreferredBackupWindow="03:00-04:00",
            PreferredMaintenanceWindow="sun:04:00-sun:05:00",
            DeletionProtection=True,
            EnablePerformanceInsights=True,
            PerformanceInsightsRetentionPeriod=7,
            EnableCloudwatchLogsExports=["postgresql", "upgrade"],
            DBSubnetGroupName=subnet_group,
            VpcSecurityGroupIds=security_groups or [],
            Tags=[
                {"Key": "Environment", "Value": "production"},
                {"Key": "Application", "Value": "python-app"}
            ]
        )
        
        return response["DBInstance"]
    
    def get_connection_string(self, db_identifier: str, db_name: str) -> str:
        """ดึง connection string จาก RDS"""
        response = self.rds.describe_db_instances(
            DBInstanceIdentifier=db_identifier
        )
        
        instance = response["DBInstances"][0]
        endpoint = instance["Endpoint"]
        
        # ดึง credentials จาก Secrets Manager
        secrets = boto3.client("secretsmanager", region_name=self.region)
        secret_value = secrets.get_secret_value(
            SecretId=f"rds/{db_identifier}/credentials"
        )
        
        import json
        creds = json.loads(secret_value["SecretString"])
        
        return (
            f"postgresql://{creds['username']}:{creds['password']}"
            f"@{endpoint['Address']}:{endpoint['Port']}/{db_name}"
        )


# Database Migration จาก Local ไปยัง RDS
def migrate_to_rds():
    """Script สำหรับ migrate database ไปยัง RDS"""
    local_url = os.getenv("LOCAL_DATABASE_URL")
    rds_url = os.getenv("RDS_DATABASE_URL")
    
    # ใช้ Alembic สำหรับ schema migration
    from alembic.config import Config
    from alembic import command
    
    alembic_cfg = Config("alembic.ini")
    alembic_cfg.set_main_option("sqlalchemy.url", rds_url)
    
    # Apply migrations
    command.upgrade(alembic_cfg, "head")
    print("Schema migration completed!")
```

---

## 2. GCP Cloud Run

```python
# gcp/cloud_run_deployment.py
"""
GCP Cloud Run - Serverless container platform
เหมาะสำหรับ Python apps ที่ traffic ไม่สม่ำเสมอ
"""
from google.cloud import run_v2
from google.api_core.exceptions import NotFound
import os


class CloudRunDeployer:
    """Deploy Python app ไปยัง GCP Cloud Run"""
    
    def __init__(self, project_id: str, region: str = "asia-southeast1"):
        self.project_id = project_id
        self.region = region
        self.client = run_v2.ServicesClient()
        self.parent = f"projects/{project_id}/locations/{region}"
    
    def deploy_service(
        self,
        service_name: str,
        image_uri: str,
        env_vars: dict = None,
        secrets: dict = None,
        min_instances: int = 0,
        max_instances: int = 10,
        cpu: str = "1",
        memory: str = "512Mi",
        concurrency: int = 80,
        timeout_seconds: int = 300
    ) -> run_v2.Service:
        """Deploy service ไปยัง Cloud Run"""
        
        # สร้าง container configuration
        container = run_v2.Container(
            image=image_uri,
            ports=[run_v2.ContainerPort(container_port=8000)],
            env=[
                run_v2.EnvVar(name=k, value=v)
                for k, v in (env_vars or {}).items()
            ],
            resources=run_v2.ResourceRequirements(
                limits={"cpu": cpu, "memory": memory},
                cpu_idle=False  # เก็บ CPU สำหรับ background tasks
            )
        )
        
        # เพิ่ม secrets
        if secrets:
            for secret_name, secret_path in secrets.items():
                project, secret_id, version = secret_path.split("/")
                container.volume_mounts.append(
                    run_v2.VolumeMount(
                        name=secret_name,
                        mount_path=f"/secrets/{secret_name}"
                    )
                )
        
        # สร้าง service template
        template = run_v2.RevisionTemplate(
            containers=[container],
            scaling=run_v2.RevisionScaling(
                min_instance_count=min_instances,
                max_instance_count=max_instances
            ),
            timeout=f"{timeout_seconds}s",
            max_instance_request_concurrency=concurrency,
            service_account=f"{service_name}@{self.project_id}.iam.gserviceaccount.com"
        )
        
        # Deploy service
        service_path = f"{self.parent}/services/{service_name}"
        
        try:
            # ตรวจสอบว่า service มีอยู่แล้ว
            existing = self.client.get_service(name=service_path)
            
            # อัพเดท service
            service = run_v2.Service(
                name=service_path,
                template=template,
                traffic=[run_v2.TrafficTarget(
                    type_=run_v2.TrafficTargetAllocationType.TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST,
                    percent=100
                )]
            )
            
            operation = self.client.update_service(service=service)
            result = operation.result()
            print(f"Updated Cloud Run service: {service_name}")
            return result
        
        except NotFound:
            # สร้าง service ใหม่
            service = run_v2.Service(
                template=template,
                traffic=[run_v2.TrafficTarget(
                    type_=run_v2.TrafficTargetAllocationType.TRAFFIC_TARGET_ALLOCATION_TYPE_LATEST,
                    percent=100
                )],
                ingress=run_v2.IngressTraffic.INGRESS_TRAFFIC_ALL
            )
            
            operation = self.client.create_service(
                parent=self.parent,
                service=service,
                service_id=service_name
            )
            result = operation.result()
            print(f"Created Cloud Run service: {service_name}")
            return result
    
    def get_service_url(self, service_name: str) -> str:
        """ดึง URL ของ service"""
        service = self.client.get_service(
            name=f"{self.parent}/services/{service_name}"
        )
        return service.uri


# Cloud Run deployment script
"""
#!/bin/bash
# deploy.sh

PROJECT_ID="my-project"
REGION="asia-southeast1"
SERVICE_NAME="python-app"
IMAGE_URI="gcr.io/$PROJECT_ID/$SERVICE_NAME:$GITHUB_SHA"

# Build และ push image
gcloud builds submit --tag $IMAGE_URI .

# Deploy ไปยัง Cloud Run
gcloud run deploy $SERVICE_NAME \\
  --image $IMAGE_URI \\
  --region $REGION \\
  --platform managed \\
  --allow-unauthenticated \\
  --set-env-vars "ENV=production" \\
  --set-secrets "DB_PASSWORD=db-password:latest" \\
  --min-instances 1 \\
  --max-instances 100 \\
  --concurrency 80 \\
  --cpu 1 \\
  --memory 512Mi \\
  --timeout 300 \\
  --service-account $SERVICE_NAME@$PROJECT_ID.iam.gserviceaccount.com
"""
```

---

## 3. File Storage (S3 / GCS)

```python
# storage/s3_manager.py
import boto3
import os
from typing import Optional, BinaryIO
from botocore.exceptions import ClientError
import mimetypes
import hashlib


class S3Manager:
    """จัดการ file storage บน AWS S3"""
    
    def __init__(
        self,
        bucket_name: str,
        region: str = "ap-southeast-1"
    ):
        self.bucket_name = bucket_name
        self.region = region
        self.s3 = boto3.client("s3", region_name=region)
        self.s3_resource = boto3.resource("s3", region_name=region)
    
    def upload_file(
        self,
        file_obj: BinaryIO,
        key: str,
        content_type: str = None,
        make_public: bool = False,
        metadata: dict = None
    ) -> str:
        """Upload file ไปยัง S3"""
        
        # ตรวจหา content type
        if not content_type:
            content_type, _ = mimetypes.guess_type(key)
            content_type = content_type or "application/octet-stream"
        
        extra_args = {
            "ContentType": content_type,
        }
        
        if make_public:
            extra_args["ACL"] = "public-read"
        
        if metadata:
            extra_args["Metadata"] = metadata
        
        # Server-side encryption
        extra_args["ServerSideEncryption"] = "AES256"
        
        self.s3.upload_fileobj(
            file_obj,
            self.bucket_name,
            key,
            ExtraArgs=extra_args
        )
        
        return f"https://{self.bucket_name}.s3.{self.region}.amazonaws.com/{key}"
    
    def upload_from_path(self, local_path: str, key: str, **kwargs) -> str:
        """Upload file จาก local path"""
        with open(local_path, "rb") as f:
            return self.upload_file(f, key, **kwargs)
    
    def download_file(self, key: str, local_path: str) -> bool:
        """Download file จาก S3"""
        try:
            self.s3.download_file(self.bucket_name, key, local_path)
            return True
        except ClientError as e:
            if e.response["Error"]["Code"] == "404":
                return False
            raise
    
    def get_file_url(self, key: str) -> str:
        """ดึง public URL"""
        return f"https://{self.bucket_name}.s3.{self.region}.amazonaws.com/{key}"
    
    def generate_presigned_url(
        self,
        key: str,
        operation: str = "get_object",
        expiration: int = 3600
    ) -> str:
        """สร้าง pre-signed URL สำหรับ temporary access"""
        return self.s3.generate_presigned_url(
            operation,
            Params={"Bucket": self.bucket_name, "Key": key},
            ExpiresIn=expiration
        )
    
    def generate_presigned_upload_url(
        self,
        key: str,
        content_type: str,
        max_size_mb: int = 10,
        expiration: int = 3600
    ) -> dict:
        """สร้าง pre-signed URL สำหรับ direct upload จาก client"""
        conditions = [
            ["content-length-range", 0, max_size_mb * 1024 * 1024],
            ["starts-with", "$Content-Type", content_type.split("/")[0]],
        ]
        
        fields = {
            "Content-Type": content_type,
        }
        
        response = self.s3.generate_presigned_post(
            Bucket=self.bucket_name,
            Key=key,
            Fields=fields,
            Conditions=conditions,
            ExpiresIn=expiration
        )
        
        return response
    
    def delete_file(self, key: str) -> bool:
        """ลบ file"""
        try:
            self.s3.delete_object(Bucket=self.bucket_name, Key=key)
            return True
        except ClientError:
            return False
    
    def copy_file(self, source_key: str, dest_key: str) -> bool:
        """Copy file ภายใน bucket"""
        try:
            self.s3.copy_object(
                Bucket=self.bucket_name,
                CopySource={"Bucket": self.bucket_name, "Key": source_key},
                Key=dest_key
            )
            return True
        except ClientError:
            return False
    
    def list_files(self, prefix: str = "", max_files: int = 100) -> list:
        """แสดงรายการ files"""
        response = self.s3.list_objects_v2(
            Bucket=self.bucket_name,
            Prefix=prefix,
            MaxKeys=max_files
        )
        
        return [
            {
                "key": obj["Key"],
                "size": obj["Size"],
                "last_modified": obj["LastModified"].isoformat(),
            }
            for obj in response.get("Contents", [])
        ]


# FastAPI integration กับ S3
from fastapi import FastAPI, UploadFile, File, HTTPException
import uuid

app = FastAPI()
s3_manager = S3Manager(bucket_name="my-app-files")


@app.post("/upload/")
async def upload_file(
    file: UploadFile = File(...),
    user_id: int = 1
):
    """Upload file endpoint"""
    
    # ตรวจสอบ file size
    MAX_SIZE = 10 * 1024 * 1024  # 10MB
    content = await file.read()
    
    if len(content) > MAX_SIZE:
        raise HTTPException(status_code=400, detail="File too large (max 10MB)")
    
    # สร้าง unique key
    file_ext = os.path.splitext(file.filename)[1]
    file_hash = hashlib.md5(content).hexdigest()[:8]
    key = f"uploads/{user_id}/{file_hash}{file_ext}"
    
    # Upload ไปยัง S3
    import io
    url = s3_manager.upload_file(
        file_obj=io.BytesIO(content),
        key=key,
        content_type=file.content_type,
        metadata={"original_name": file.filename, "user_id": str(user_id)}
    )
    
    return {
        "url": url,
        "key": key,
        "size": len(content),
        "content_type": file.content_type
    }


@app.get("/upload/presigned/{filename}")
async def get_presigned_upload_url(filename: str, user_id: int = 1):
    """สร้าง pre-signed URL สำหรับ direct upload"""
    file_ext = os.path.splitext(filename)[1]
    key = f"uploads/{user_id}/{uuid.uuid4().hex}{file_ext}"
    
    presigned_data = s3_manager.generate_presigned_upload_url(
        key=key,
        content_type="image/",
        max_size_mb=10,
        expiration=3600
    )
    
    return {
        "upload_url": presigned_data["url"],
        "fields": presigned_data["fields"],
        "key": key,
        "expires_in": 3600
    }
```

---

## 4. Secrets Management

```python
# secrets/secrets_manager.py
"""
Secrets Management - ไม่เก็บ secrets ใน code หรือ environment variables ธรรมดา

AWS Secrets Manager หรือ GCP Secret Manager
"""
import boto3
import json
import os
from typing import Optional
from functools import lru_cache
import time


class SecretsManager:
    """จัดการ secrets อย่างปลอดภัย"""
    
    def __init__(self, region: str = "ap-southeast-1"):
        self.client = boto3.client("secretsmanager", region_name=region)
        self._cache = {}
        self._cache_ttl = 300  # 5 นาที
    
    def get_secret(self, secret_id: str, force_refresh: bool = False) -> dict:
        """ดึง secret พร้อม caching"""
        cache_key = secret_id
        
        # ตรวจสอบ cache
        if not force_refresh and cache_key in self._cache:
            cached_value, cached_time = self._cache[cache_key]
            if time.time() - cached_time < self._cache_ttl:
                return cached_value
        
        # ดึงจาก Secrets Manager
        try:
            response = self.client.get_secret_value(SecretId=secret_id)
            secret_string = response.get("SecretString")
            
            if secret_string:
                value = json.loads(secret_string)
            else:
                # Binary secret
                import base64
                value = base64.b64decode(response["SecretBinary"])
            
            # Cache
            self._cache[cache_key] = (value, time.time())
            return value
        
        except self.client.exceptions.ResourceNotFoundException:
            raise ValueError(f"Secret not found: {secret_id}")
    
    def get_db_credentials(self, db_identifier: str) -> dict:
        """ดึง database credentials"""
        return self.get_secret(f"rds/{db_identifier}/credentials")
    
    def rotate_secret(self, secret_id: str) -> bool:
        """Rotate secret"""
        try:
            self.client.rotate_secret(SecretId=secret_id)
            # Clear cache
            self._cache.pop(secret_id, None)
            return True
        except Exception as e:
            print(f"Failed to rotate secret {secret_id}: {e}")
            return False
    
    def create_secret(
        self,
        name: str,
        value: dict | str,
        description: str = "",
        tags: dict = None
    ) -> str:
        """สร้าง secret ใหม่"""
        if isinstance(value, dict):
            secret_string = json.dumps(value)
        else:
            secret_string = value
        
        response = self.client.create_secret(
            Name=name,
            Description=description,
            SecretString=secret_string,
            Tags=[{"Key": k, "Value": v} for k, v in (tags or {}).items()]
        )
        
        return response["ARN"]


# Settings management ด้วย Pydantic
from pydantic_settings import BaseSettings
from functools import lru_cache


class Settings(BaseSettings):
    """Application settings"""
    
    # Database
    database_url: str = ""
    db_pool_size: int = 10
    db_max_overflow: int = 20
    
    # Redis
    redis_url: str = "redis://localhost:6379"
    redis_pool_size: int = 50
    
    # AWS
    aws_region: str = "ap-southeast-1"
    s3_bucket: str = ""
    
    # App
    secret_key: str = ""
    debug: bool = False
    log_level: str = "INFO"
    allowed_origins: list = ["*"]
    
    # Feature flags
    enable_cache: bool = True
    enable_metrics: bool = True
    
    class Config:
        env_file = ".env"
        env_file_encoding = "utf-8"
        case_sensitive = False


@lru_cache()
def get_settings() -> Settings:
    """Singleton pattern สำหรับ settings"""
    settings = Settings()
    
    # โหลด secrets จาก AWS Secrets Manager ใน production
    if not settings.debug:
        try:
            sm = SecretsManager(region=settings.aws_region)
            
            # โหลด database credentials
            db_creds = sm.get_db_credentials("production-db")
            settings.database_url = (
                f"postgresql://{db_creds['username']}:{db_creds['password']}"
                f"@{db_creds['host']}:{db_creds['port']}/{db_creds['dbname']}"
            )
            
            # โหลด secret key
            app_secrets = sm.get_secret("python-app/secrets")
            settings.secret_key = app_secrets.get("secret_key", "")
            
        except Exception as e:
            print(f"Warning: Could not load secrets from AWS: {e}")
    
    return settings


# ตัวอย่างการใช้
from fastapi import Depends

def get_db_url(settings: Settings = Depends(get_settings)) -> str:
    return settings.database_url
```

---

## 5. Terraform Infrastructure as Code

```hcl
# terraform/main.tf
# Terraform configuration สำหรับ Python App บน AWS

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket = "terraform-state-bucket"
    key    = "python-app/terraform.tfstate"
    region = "ap-southeast-1"
  }
}

provider "aws" {
  region = var.aws_region
}

# Variables
variable "aws_region" {
  default = "ap-southeast-1"
}

variable "app_name" {
  default = "python-app"
}

variable "environment" {
  default = "production"
}

variable "db_password" {
  sensitive = true
}

# VPC
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  
  name = "${var.app_name}-vpc"
  cidr = "10.0.0.0/16"
  
  azs             = ["${var.aws_region}a", "${var.aws_region}b", "${var.aws_region}c"]
  public_subnets  = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
  private_subnets = ["10.0.11.0/24", "10.0.12.0/24", "10.0.13.0/24"]
  
  enable_nat_gateway = true
  single_nat_gateway = false  # Production: ใช้ NAT Gateway แยก AZ
  
  tags = {
    Environment = var.environment
    Application = var.app_name
  }
}

# ECS Cluster
resource "aws_ecs_cluster" "main" {
  name = "${var.app_name}-cluster"
  
  setting {
    name  = "containerInsights"
    value = "enabled"
  }
  
  tags = {
    Environment = var.environment
  }
}

# RDS PostgreSQL
resource "aws_db_instance" "main" {
  identifier = "${var.app_name}-db"
  
  engine               = "postgres"
  engine_version       = "15.4"
  instance_class       = "db.t3.medium"
  allocated_storage    = 20
  max_allocated_storage = 100
  storage_encrypted    = true
  
  db_name  = "appdb"
  username = "appuser"
  password = var.db_password
  
  multi_az               = true
  publicly_accessible    = false
  deletion_protection    = true
  skip_final_snapshot    = false
  final_snapshot_identifier = "${var.app_name}-final-snapshot"
  
  backup_retention_period = 7
  backup_window          = "03:00-04:00"
  maintenance_window     = "sun:04:00-sun:05:00"
  
  vpc_security_group_ids = [aws_security_group.rds.id]
  db_subnet_group_name   = aws_db_subnet_group.main.name
  
  enabled_cloudwatch_logs_exports = ["postgresql", "upgrade"]
  performance_insights_enabled    = true
  
  tags = {
    Environment = var.environment
  }
}

# S3 Bucket
resource "aws_s3_bucket" "files" {
  bucket = "${var.app_name}-files-${var.environment}"
  
  tags = {
    Environment = var.environment
  }
}

resource "aws_s3_bucket_versioning" "files" {
  bucket = aws_s3_bucket.files.id
  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "files" {
  bucket = aws_s3_bucket.files.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# ElastiCache Redis
resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "${var.app_name}-redis"
  engine               = "redis"
  node_type            = "cache.t3.micro"
  num_cache_nodes      = 1
  parameter_group_name = "default.redis7"
  engine_version       = "7.0"
  port                 = 6379
  
  subnet_group_name  = aws_elasticache_subnet_group.main.name
  security_group_ids = [aws_security_group.redis.id]
  
  tags = {
    Environment = var.environment
  }
}

# Outputs
output "ecs_cluster_arn" {
  value = aws_ecs_cluster.main.arn
}

output "rds_endpoint" {
  value     = aws_db_instance.main.endpoint
  sensitive = true
}

output "redis_endpoint" {
  value = aws_elasticache_cluster.redis.cache_nodes[0].address
}

output "s3_bucket_name" {
  value = aws_s3_bucket.files.id
}
```

```python
# deploy/deploy_script.py
"""
Complete deployment script
"""
import subprocess
import os
import sys
import boto3
import json


class ProductionDeployer:
    """Automated deployment pipeline"""
    
    def __init__(self, environment: str = "production"):
        self.env = environment
        self.region = os.getenv("AWS_REGION", "ap-southeast-1")
        self.account_id = self._get_account_id()
    
    def _get_account_id(self) -> str:
        sts = boto3.client("sts", region_name=self.region)
        return sts.get_caller_identity()["Account"]
    
    def run_tests(self) -> bool:
        """รัน tests ก่อน deploy"""
        print("🧪 Running tests...")
        result = subprocess.run(
            ["pytest", "tests/", "-v", "--tb=short"],
            capture_output=True,
            text=True
        )
        
        if result.returncode != 0:
            print(f"❌ Tests failed:\n{result.stdout}\n{result.stderr}")
            return False
        
        print("✅ All tests passed!")
        return True
    
    def build_and_push_image(self, image_tag: str) -> str:
        """Build และ push Docker image"""
        ecr_repo = f"{self.account_id}.dkr.ecr.{self.region}.amazonaws.com/python-app"
        image_uri = f"{ecr_repo}:{image_tag}"
        
        # Login to ECR
        print("🔑 Logging in to ECR...")
        login_cmd = subprocess.run(
            ["aws", "ecr", "get-login-password", "--region", self.region],
            capture_output=True, text=True
        )
        
        subprocess.run(
            ["docker", "login", "--username", "AWS", "--password-stdin", 
             f"{self.account_id}.dkr.ecr.{self.region}.amazonaws.com"],
            input=login_cmd.stdout, text=True
        )
        
        # Build image
        print("🏗️  Building Docker image...")
        subprocess.run([
            "docker", "build",
            "-f", "Dockerfile.production",
            "-t", image_uri,
            "."
        ], check=True)
        
        # Push image
        print("📤 Pushing image to ECR...")
        subprocess.run(["docker", "push", image_uri], check=True)
        
        print(f"✅ Image pushed: {image_uri}")
        return image_uri
    
    def run_migrations(self, db_url: str):
        """รัน database migrations"""
        print("🗄️  Running database migrations...")
        
        env = os.environ.copy()
        env["DATABASE_URL"] = db_url
        
        subprocess.run(
            ["python", "-m", "alembic", "upgrade", "head"],
            env=env,
            check=True
        )
        
        print("✅ Migrations completed!")
    
    def deploy(self, git_sha: str):
        """Full deployment pipeline"""
        print(f"🚀 Starting deployment for {git_sha[:8]}...")
        
        # 1. Run tests
        if not self.run_tests():
            sys.exit(1)
        
        # 2. Build and push image
        image_uri = self.build_and_push_image(git_sha)
        
        # 3. Deploy to ECS
        ecs = ECSDeployer(region=self.region)
        
        task_def = ecs.create_task_definition(
            family="python-app",
            image_uri=image_uri,
            cpu=512,
            memory=1024,
            env_vars={
                "ENVIRONMENT": self.env,
                "AWS_REGION": self.region,
            },
            secrets={
                "DATABASE_URL": f"arn:aws:secretsmanager:{self.region}:{self.account_id}:secret:python-app/db-url",
                "SECRET_KEY": f"arn:aws:secretsmanager:{self.region}:{self.account_id}:secret:python-app/secret-key",
            }
        )
        
        ecs.deploy_service(
            cluster="production-cluster",
            service_name="python-app-service",
            task_definition=task_def["family"],
            desired_count=3
        )
        
        # 4. Wait for deployment
        ecs.wait_for_deployment("production-cluster", "python-app-service")
        
        print(f"✅ Deployment completed successfully!")


if __name__ == "__main__":
    import sys
    git_sha = sys.argv[1] if len(sys.argv) > 1 else "latest"
    
    deployer = ProductionDeployer()
    deployer.deploy(git_sha)
```

---

## 6. สรุป Part 101

✅ Deploy Python apps ไปยัง AWS ECS Fargate และ GCP Cloud Run
✅ สร้าง Production-ready Dockerfile ด้วย Multi-stage build
✅ ใช้งาน AWS RDS PostgreSQL พร้อม Multi-AZ, encryption, backups
✅ จัดการ File Storage ด้วย AWS S3 รวมถึง pre-signed URLs
✅ ใช้ AWS Secrets Manager สำหรับ secure secrets management
✅ สร้าง Infrastructure as Code ด้วย Terraform (VPC, ECS, RDS, S3, Redis)
✅ Automated deployment pipeline พร้อม tests, migrations และ rolling updates

## ➡️ ถัดไป: Part 102 - Monitoring and Observability

*Part 101/105 | Python Course - World-Class Level*
