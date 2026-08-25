# SECTION 14: HANDS-ON LABS & PRACTICAL PROJECTS

## TABLE OF CONTENTS
- [Lab Environment Setup](#lab-environment-setup)
- [Beginner Labs](#beginner-labs)
- [Intermediate Labs](#intermediate-labs)
- [Advanced Labs](#advanced-labs)
- [Expert Capstone Projects](#expert-capstone-projects)

---

## LAB ENVIRONMENT SETUP

**Prerequisites:**
- AWS account (free tier or paid).
- AWS CLI v2 installed and configured.
- kubectl v1.27+.
- Terraform v1.5+.
- Docker (optional, for local testing).
- Code editor (VS Code recommended).

**Setup AWS Credentials:**
```bash
aws configure
# Enter Access Key, Secret Key, default region (us-east-1)

# Verify
aws sts get-caller-identity
# Should return your account ID and ARN
```

**Recommended AWS Region:** us-east-1 (most services, lowest cost).

**Cost Awareness:** Each lab may cost $1–$20. Destroy resources after labs to avoid surprises. Use AWS Budgets to set spending alerts.

---

## BEGINNER LABS

### Lab 1: Deploy a Three-Tier Application to ECS Fargate

**Objective:** Understand ECS, task definitions, load balancing, and container deployment.

**Duration:** 2–3 hours.

**Architecture:**
```
Internet → ALB → ECS Fargate Tasks (Web Server) → RDS (Database) + ElastiCache (Cache)
```

**Steps:**

1. **Create VPC and Subnets:**
   ```bash
   # Use Terraform (or console)
   cat > main.tf << 'EOF'
   provider "aws" {
     region = "us-east-1"
   }
   
   resource "aws_vpc" "lab" {
     cidr_block = "10.0.0.0/16"
   }
   
   resource "aws_subnet" "public_1" {
     vpc_id            = aws_vpc.lab.id
     cidr_block        = "10.0.1.0/24"
     availability_zone = "us-east-1a"
   }
   
   resource "aws_subnet" "public_2" {
     vpc_id            = aws_vpc.lab.id
     cidr_block        = "10.0.2.0/24"
     availability_zone = "us-east-1b"
   }
   EOF
   
   terraform init
   terraform plan
   terraform apply
   ```

2. **Create Application Load Balancer:**
   ```bash
   aws elbv2 create-load-balancer \
     --name lab-alb \
     --subnets subnet-xxxxx subnet-yyyyy \
     --security-groups sg-xxxxx \
     --scheme internet-facing \
     --region us-east-1
   ```

3. **Create ECS Cluster:**
   ```bash
   aws ecs create-cluster --cluster-name lab-cluster --region us-east-1
   ```

4. **Create Task Definition (container spec):**
   ```bash
   cat > task-definition.json << 'EOF'
   {
     "family": "lab-web-app",
     "networkMode": "awsvpc",
     "requiresCompatibilities": ["FARGATE"],
     "cpu": "256",
     "memory": "512",
     "containerDefinitions": [
       {
         "name": "web-app",
         "image": "nginx:latest",
         "portMappings": [
           {
             "containerPort": 80,
             "hostPort": 80,
             "protocol": "tcp"
           }
         ],
         "essential": true,
         "logConfiguration": {
           "logDriver": "awslogs",
           "options": {
             "awslogs-group": "/ecs/lab-web-app",
             "awslogs-region": "us-east-1",
             "awslogs-stream-prefix": "ecs"
           }
         }
       }
     ]
   }
   EOF
   
   aws ecs register-task-definition --cli-input-json file://task-definition.json --region us-east-1
   ```

5. **Create ECS Service (deploy tasks):**
   ```bash
   aws ecs create-service \
     --cluster lab-cluster \
     --service-name lab-web-service \
     --task-definition lab-web-app \
     --desired-count 2 \
     --load-balancers targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=web-app,containerPort=80 \
     --network-configuration "awsvpcConfiguration={subnets=[subnet-xxxxx,subnet-yyyyy],securityGroups=[sg-xxxxx],assignPublicIp=ENABLED}" \
     --region us-east-1
   ```

6. **Verify:**
   ```bash
   # Check service status
   aws ecs describe-services --cluster lab-cluster --services lab-web-service --region us-east-1 | jq '.services[0].status'
   
   # Get ALB DNS
   aws elbv2 describe-load-balancers --names lab-alb --region us-east-1 | jq '.LoadBalancers[0].DNSName'
   
   # Visit in browser (should see nginx page)
   curl http://<ALB-DNS>
   ```

7. **Cleanup:**
   ```bash
   aws ecs delete-service --cluster lab-cluster --service lab-web-service --force --region us-east-1
   aws ecs delete-cluster --cluster lab-cluster --region us-east-1
   terraform destroy
   ```

**Learning Outcomes:**
- Understand ECS/Fargate architecture.
- Load balancing concepts.
- Network configuration (VPC, subnets, security groups).
- CloudWatch logging.

**Interview Connection:** Explain ECS task lifecycle, how ALB health checks work, scaling.

---

### Lab 2: Build a Lambda-based Serverless API

**Objective:** Create REST API using API Gateway + Lambda + DynamoDB.

**Duration:** 1.5–2 hours.

**Architecture:**
```
Client → API Gateway → Lambda → DynamoDB (storage)
```

**Steps:**

1. **Create DynamoDB Table:**
   ```bash
   aws dynamodb create-table \
     --table-name lab-todos \
     --attribute-definitions AttributeName=id,AttributeType=S \
     --key-schema AttributeName=id,KeyType=HASH \
     --billing-mode PAY_PER_REQUEST \
     --region us-east-1
   ```

2. **Create Lambda Function:**
   ```bash
   mkdir lambda-function
   cd lambda-function
   
   cat > index.py << 'EOF'
   import json
   import boto3
   import uuid
   from datetime import datetime
   
   dynamodb = boto3.resource('dynamodb')
   table = dynamodb.Table('lab-todos')
   
   def lambda_handler(event, context):
       http_method = event['httpMethod']
       
       if http_method == 'POST':
           body = json.loads(event['body'])
           todo_id = str(uuid.uuid4())
           table.put_item(Item={
               'id': todo_id,
               'title': body['title'],
               'created_at': datetime.utcnow().isoformat()
           })
           return {
               'statusCode': 201,
               'body': json.dumps({'id': todo_id})
           }
       
       elif http_method == 'GET':
           response = table.scan()
           return {
               'statusCode': 200,
               'body': json.dumps(response['Items'])
           }
       
       return {'statusCode': 400, 'body': 'Invalid method'}
   EOF
   
   zip function.zip index.py
   
   # Upload
   aws lambda create-function \
     --function-name lab-todo-api \
     --runtime python3.11 \
     --role arn:aws:iam::123456789:role/lambda-execution-role \
     --handler index.lambda_handler \
     --zip-file fileb://function.zip \
     --region us-east-1
   ```

3. **Create API Gateway:**
   ```bash
   # Create REST API
   API_ID=$(aws apigateway create-rest-api \
     --name lab-todo-api \
     --region us-east-1 | jq -r '.id')
   
   # Get resource ID
   RESOURCE_ID=$(aws apigateway get-resources --rest-api-id $API_ID --region us-east-1 | jq -r '.items[0].id')
   
   # Create POST method
   aws apigateway put-method \
     --rest-api-id $API_ID \
     --resource-id $RESOURCE_ID \
     --http-method POST \
     --authorization-type NONE \
     --region us-east-1
   
   # Create integration with Lambda
   aws apigateway put-integration \
     --rest-api-id $API_ID \
     --resource-id $RESOURCE_ID \
     --http-method POST \
     --type AWS_PROXY \
     --integration-http-method POST \
     --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789:function:lab-todo-api/invocations \
     --region us-east-1
   
   # Deploy
   aws apigateway create-deployment \
     --rest-api-id $API_ID \
     --stage-name prod \
     --region us-east-1
   ```

4. **Test:**
   ```bash
   API_URL="https://$API_ID.execute-api.us-east-1.amazonaws.com/prod"
   
   # Create todo
   curl -X POST $API_URL \
     -H "Content-Type: application/json" \
     -d '{"title": "Learn AWS"}'
   
   # Get todos
   curl -X GET $API_URL
   ```

**Interview Connection:** Explain Lambda cold starts, API Gateway caching, DynamoDB consistency.

---

## INTERMEDIATE LABS

### Lab 3: Multi-Region EKS Cluster with Failover

**Objective:** Deploy EKS in 2 regions, set up Route53 failover, demonstrate disaster recovery.

**Duration:** 4–5 hours.

**Key Concepts:**
- Multi-region architecture.
- Route53 health checks and failover routing.
- Cross-region RDS replication.

**Steps (High-Level):**

1. Create EKS cluster in us-east-1.
2. Deploy application to cluster.
3. Create EKS cluster in us-west-2.
4. Replicate application.
5. Set up RDS global database (replication).
6. Configure Route53 failover.
7. Simulate region failure; verify automatic failover.

**Terraform Template:**
```hcl
# Modules for multi-region setup
module "eks_us_east" {
  source = "./modules/eks"
  region = "us-east-1"
}

module "eks_us_west" {
  source = "./modules/eks"
  region = "us-west-2"
}

module "route53_failover" {
  source = "./modules/route53"
  
  primary_region_endpoint = module.eks_us_east.alb_dns
  secondary_region_endpoint = module.eks_us_west.alb_dns
}
```

**Interview Connection:** Explain RPO/RTO, consistency across regions, traffic failover.

---

## ADVANCED LABS

### Lab 4: Implement Canary Deployment with GitOps

**Objective:** Use Fluxv2 (GitOps tool) to manage EKS deployments with canary traffic shifting.

**Duration:** 6–8 hours.

**Key Concepts:**
- GitOps workflow.
- Canary deployments (5% → 25% → 100% traffic).
- Automated rollback on error rate spike.

**Tools:** Flux, Flagger, Prometheus, Grafana.

**Interview Connection:** Explain CI/CD pipeline, blue-green vs canary, observability-driven deployments.

---

## EXPERT CAPSTONE PROJECTS

### Project 1: Build a Distributed Metrics Collection System

**Objective:** Collect metrics from 100+ services, store in time-series database, query for dashboards.

**Architecture:**
- **Collection:** Prometheus scrape targets.
- **Storage:** Amazon Managed Prometheus (AMP) or self-hosted Prometheus + S3.
- **Visualization:** Grafana.
- **Scaling:** ~1M metrics/minute.

**Complexity:** Expert-level. Requires understanding of:
- Prometheus architecture and relabeling.
- Time-series database optimization.
- High-cardinality metrics handling.
- Cost optimization (compress, downsample).

**Interview Connection:** Ask about this project during system design rounds. Interviewers test:
- Understanding of observability at scale.
- Cost-performance trade-offs.
- Operational insights (what metrics matter).

---

### Project 2: Design a Production-Ready ML Pipeline on AWS

**Objective:** Train ML model, deploy to production, enable A/B testing.

**Architecture:**
- **Data:** S3 data lake.
- **Processing:** SageMaker Processing jobs (Spark).
- **Training:** SageMaker Training (auto-scaling).
- **Model Registry:** SageMaker Model Registry.
- **Serving:** SageMaker Endpoints or Lambda (real-time inference).
- **A/B Testing:** Route 10% traffic to new model.
- **Monitoring:** CloudWatch + SageMaker Model Monitor.

**Complexity:** Expert-level. Requires understanding of:
- Model lifecycle management.
- Data pipeline orchestration (Step Functions).
- Cost optimization (spot instances for training).
- Compliance (model explainability, bias detection).

**Interview Connection:** Demonstrates end-to-end ownership. Interviewers ask:
- How do you handle model drift?
- How do you perform canary deployment of models?
- Cost of model serving at scale?

---

## LABS CHECKLIST

| Lab | Duration | Cost | Skills | Difficulty |
|-----|----------|------|--------|------------|
| Deploy ECS App | 2h | $3–5 | ECS, ALB, VPC | Beginner |
| Lambda API | 1.5h | $1–2 | Lambda, DynamoDB, API Gateway | Beginner |
| Multi-Region EKS | 5h | $10–15 | EKS, RDS, Route53 | Intermediate |
| Canary Deployment | 7h | $5–10 | GitOps, Flagger, Prometheus | Advanced |
| Metrics System | 15h | $20–50 | Prometheus, Grafana, Time-series | Expert |
| ML Pipeline | 20h | $30–100 | SageMaker, Step Functions | Expert |

**Recommended Learning Path:**
1. Start with Beginner Labs (get comfortable with AWS).
2. Do 1–2 Intermediate Labs (understand multi-region, failover).
3. Pick 1 Advanced Lab that aligns with your interests.
4. Optional: Capstone project for deep expertise.

---

## DOCUMENTATION LINKS

- [AWS Skill Builder (Free Labs)](https://skillbuilder.aws.com/)
- [AWS Workshops](https://workshops.aws/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/)

