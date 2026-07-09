# Python for DevOps Interview Questions - Complete Guide

> **300+ Python for DevOps Interview Questions for Senior DevOps Engineer, SRE, and Platform Engineer roles at FAANG companies**

---

## 📋 Table of Contents

- [Python Fundamentals](#python-fundamentals)
- [Automation & Scripting](#automation--scripting)
- [API Development](#api-development)
- [Cloud SDKs](#cloud-sdks)
- [Kubernetes Client](#kubernetes-client)
- [Infrastructure as Code](#infrastructure-as-code)
- [Testing & Quality](#testing--quality)
- [Best Practices](#best-practices)

---

## Python Fundamentals

### 🟢 Basic Questions

#### Q1: Explain key Python concepts relevant to DevOps scripting.

**Basic Answer:**
Key concepts include: data structures (lists, dicts, sets), file handling, exception handling, modules/packages, generators, decorators, context managers, and async programming.

**Advanced Answer:**

```python
# ==================== DATA STRUCTURES ====================
# Lists - ordered, mutable
servers = ['web1', 'web2', 'web3']
servers.append('web4')
servers[1:3]  # Slicing: ['web2', 'web3']

# Dictionaries - key-value, mutable
config = {
    'host': 'localhost',
    'port': 8080,
    'debug': True
}
config.get('timeout', 30)  # Default value if missing

# Sets - unique, unordered
active_hosts = {'host1', 'host2'}
all_hosts = {'host1', 'host2', 'host3'}
inactive = all_hosts - active_hosts  # Set difference

# Comprehensions
squared = [x**2 for x in range(10)]
filtered = {k: v for k, v in config.items() if v is not None}


# ==================== CONTEXT MANAGERS ====================
# Automatic resource cleanup
class SSHConnection:
    def __init__(self, host):
        self.host = host
        
    def __enter__(self):
        print(f"Connecting to {self.host}")
        return self
        
    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"Disconnecting from {self.host}")
        return False  # Don't suppress exceptions

with SSHConnection('server1') as conn:
    # Connection automatically closed after block
    pass

# Using contextlib
from contextlib import contextmanager

@contextmanager
def temporary_env(key, value):
    """Temporarily set environment variable"""
    import os
    original = os.environ.get(key)
    os.environ[key] = value
    try:
        yield
    finally:
        if original is None:
            del os.environ[key]
        else:
            os.environ[key] = original


# ==================== DECORATORS ====================
import functools
import time

def retry(max_attempts=3, delay=1):
    """Retry decorator for flaky operations"""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    print(f"Attempt {attempt + 1} failed: {e}")
                    time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

@retry(max_attempts=3, delay=2)
def unstable_api_call():
    # Might fail occasionally
    pass


# ==================== GENERATORS ====================
def read_large_file(filepath):
    """Memory-efficient file reading"""
    with open(filepath, 'r') as f:
        for line in f:
            yield line.strip()

# Process without loading entire file into memory
for line in read_large_file('/var/log/syslog'):
    if 'ERROR' in line:
        print(line)


# ==================== ASYNC PROGRAMMING ====================
import asyncio
import aiohttp

async def check_health(session, url):
    """Async health check"""
    try:
        async with session.get(url, timeout=5) as response:
            return url, response.status
    except Exception as e:
        return url, str(e)

async def check_all_services(urls):
    """Check multiple services concurrently"""
    async with aiohttp.ClientSession() as session:
        tasks = [check_health(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
        return dict(results)

# Run async code
urls = [
    'http://service1:8080/health',
    'http://service2:8080/health',
    'http://service3:8080/health'
]
results = asyncio.run(check_all_services(urls))
```

```
┌─────────────────────────────────────────────────────────────────┐
│                    PYTHON DEVOPS CONCEPTS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  EXECUTION FLOW COMPARISON:                                     │
│                                                                  │
│  Synchronous:                                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Task1 ████████                                          │    │
│  │  Task2         ████████                                  │    │
│  │  Task3                 ████████                          │    │
│  │  Total: 24 units                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  Async (Concurrent):                                            │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │  Task1 ████████                                          │    │
│  │  Task2 ████████                                          │    │
│  │  Task3 ████████                                          │    │
│  │  Total: 8 units (3x faster for I/O-bound tasks)          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  WHEN TO USE WHAT:                                              │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                          │    │
│  │  Threading    → I/O-bound, GIL-limited CPU               │    │
│  │  Multiprocess → CPU-bound, true parallelism              │    │
│  │  Async/Await  → Many I/O operations, single thread       │    │
│  │                                                          │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Automation & Scripting

### 🟡 Intermediate Questions

#### Q2: Write a script to automate server provisioning and configuration.

**Advanced Answer:**

```python
#!/usr/bin/env python3
"""
Server Provisioning Automation Script
Handles VM creation, configuration, and validation
"""

import subprocess
import json
import logging
import sys
from dataclasses import dataclass
from typing import List, Optional
from pathlib import Path
from concurrent.futures import ThreadPoolExecutor, as_completed

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


@dataclass
class ServerConfig:
    """Server configuration dataclass"""
    name: str
    ip: str
    role: str
    ssh_user: str = 'ubuntu'
    ssh_key: str = '~/.ssh/id_rsa'


class ServerProvisioner:
    """Handles server provisioning and configuration"""
    
    def __init__(self, servers: List[ServerConfig]):
        self.servers = servers
        
    def run_remote_command(
        self, 
        server: ServerConfig, 
        command: str,
        timeout: int = 60
    ) -> tuple[int, str, str]:
        """Execute command on remote server via SSH"""
        ssh_command = [
            'ssh',
            '-o', 'StrictHostKeyChecking=no',
            '-o', 'ConnectTimeout=10',
            '-i', Path(server.ssh_key).expanduser(),
            f'{server.ssh_user}@{server.ip}',
            command
        ]
        
        try:
            result = subprocess.run(
                ssh_command,
                capture_output=True,
                text=True,
                timeout=timeout
            )
            return result.returncode, result.stdout, result.stderr
        except subprocess.TimeoutExpired:
            return -1, '', 'Command timed out'
        except Exception as e:
            return -1, '', str(e)
    
    def copy_file(
        self, 
        server: ServerConfig, 
        local_path: str, 
        remote_path: str
    ) -> bool:
        """Copy file to remote server via SCP"""
        scp_command = [
            'scp',
            '-o', 'StrictHostKeyChecking=no',
            '-i', Path(server.ssh_key).expanduser(),
            local_path,
            f'{server.ssh_user}@{server.ip}:{remote_path}'
        ]
        
        try:
            result = subprocess.run(scp_command, capture_output=True, text=True)
            return result.returncode == 0
        except Exception as e:
            logger.error(f"Failed to copy file: {e}")
            return False
    
    def configure_server(self, server: ServerConfig) -> dict:
        """Configure a single server based on its role"""
        logger.info(f"Configuring {server.name} ({server.role})")
        results = {'server': server.name, 'status': 'success', 'steps': []}
        
        # Common setup for all servers
        common_commands = [
            'sudo apt-get update -qq',
            'sudo apt-get install -y -qq python3-pip htop vim',
            'sudo timedatectl set-timezone UTC'
        ]
        
        # Role-specific commands
        role_commands = {
            'web': [
                'sudo apt-get install -y -qq nginx',
                'sudo systemctl enable nginx',
                'sudo systemctl start nginx'
            ],
            'db': [
                'sudo apt-get install -y -qq postgresql postgresql-contrib',
                'sudo systemctl enable postgresql',
                'sudo systemctl start postgresql'
            ],
            'monitoring': [
                'sudo apt-get install -y -qq prometheus node-exporter',
                'sudo systemctl enable prometheus node-exporter'
            ]
        }
        
        commands = common_commands + role_commands.get(server.role, [])
        
        for cmd in commands:
            returncode, stdout, stderr = self.run_remote_command(server, cmd)
            step_result = {
                'command': cmd,
                'success': returncode == 0,
                'output': stdout if returncode == 0 else stderr
            }
            results['steps'].append(step_result)
            
            if returncode != 0:
                results['status'] = 'failed'
                logger.error(f"Failed on {server.name}: {cmd}")
                break
        
        return results
    
    def validate_server(self, server: ServerConfig) -> dict:
        """Validate server configuration"""
        logger.info(f"Validating {server.name}")
        
        validations = {
            'web': ['curl -s http://localhost:80 > /dev/null && echo OK'],
            'db': ['sudo -u postgres psql -c "SELECT 1" > /dev/null && echo OK'],
            'monitoring': ['curl -s http://localhost:9090/-/healthy && echo OK']
        }
        
        checks = validations.get(server.role, ['echo OK'])
        results = []
        
        for check in checks:
            returncode, stdout, _ = self.run_remote_command(server, check)
            results.append({
                'check': check,
                'passed': returncode == 0 and 'OK' in stdout
            })
        
        return {
            'server': server.name,
            'all_passed': all(r['passed'] for r in results),
            'checks': results
        }
    
    def provision_all(self, max_workers: int = 5) -> List[dict]:
        """Provision all servers concurrently"""
        results = []
        
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            # Submit all configuration tasks
            future_to_server = {
                executor.submit(self.configure_server, server): server
                for server in self.servers
            }
            
            # Collect results as they complete
            for future in as_completed(future_to_server):
                server = future_to_server[future]
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    results.append({
                        'server': server.name,
                        'status': 'error',
                        'error': str(e)
                    })
        
        return results


def main():
    """Main entry point"""
    # Define server inventory
    servers = [
        ServerConfig('web1', '10.0.1.10', 'web'),
        ServerConfig('web2', '10.0.1.11', 'web'),
        ServerConfig('db1', '10.0.2.10', 'db'),
        ServerConfig('monitor1', '10.0.3.10', 'monitoring')
    ]
    
    provisioner = ServerProvisioner(servers)
    
    # Provision all servers
    logger.info("Starting server provisioning...")
    results = provisioner.provision_all()
    
    # Generate report
    success_count = sum(1 for r in results if r['status'] == 'success')
    logger.info(f"Provisioning complete: {success_count}/{len(servers)} successful")
    
    # Validate configurations
    logger.info("Running validation checks...")
    for server in servers:
        validation = provisioner.validate_server(server)
        status = "✓" if validation['all_passed'] else "✗"
        logger.info(f"  {status} {server.name}")
    
    # Output results as JSON
    print(json.dumps(results, indent=2))
    
    # Exit with appropriate code
    sys.exit(0 if success_count == len(servers) else 1)


if __name__ == '__main__':
    main()
```

---

## Cloud SDKs

### 🔴 Advanced Questions

#### Q3: Show how to use boto3 for AWS automation.

**Advanced Answer:**

```python
"""
AWS Automation with boto3
Covers EC2, S3, Lambda, and CloudWatch
"""

import boto3
from botocore.exceptions import ClientError
from typing import List, Dict, Optional
import json
import logging
from datetime import datetime, timedelta

logger = logging.getLogger(__name__)


class AWSAutomation:
    """AWS automation utilities"""
    
    def __init__(self, region: str = 'us-east-1'):
        self.region = region
        self.ec2 = boto3.client('ec2', region_name=region)
        self.s3 = boto3.client('s3', region_name=region)
        self.lambda_client = boto3.client('lambda', region_name=region)
        self.cloudwatch = boto3.client('cloudwatch', region_name=region)
        self.ssm = boto3.client('ssm', region_name=region)
    
    # ==================== EC2 OPERATIONS ====================
    
    def find_instances(
        self, 
        filters: Optional[List[Dict]] = None,
        states: List[str] = ['running']
    ) -> List[Dict]:
        """Find EC2 instances with filters"""
        filter_list = filters or []
        filter_list.append({
            'Name': 'instance-state-name',
            'Values': states
        })
        
        response = self.ec2.describe_instances(Filters=filter_list)
        
        instances = []
        for reservation in response['Reservations']:
            for instance in reservation['Instances']:
                # Extract tags as dict
                tags = {t['Key']: t['Value'] for t in instance.get('Tags', [])}
                instances.append({
                    'id': instance['InstanceId'],
                    'type': instance['InstanceType'],
                    'state': instance['State']['Name'],
                    'private_ip': instance.get('PrivateIpAddress'),
                    'public_ip': instance.get('PublicIpAddress'),
                    'name': tags.get('Name', 'unnamed'),
                    'tags': tags
                })
        
        return instances
    
    def stop_instances_by_tag(
        self, 
        tag_key: str, 
        tag_value: str
    ) -> List[str]:
        """Stop instances matching a tag"""
        instances = self.find_instances(
            filters=[{'Name': f'tag:{tag_key}', 'Values': [tag_value]}]
        )
        
        instance_ids = [i['id'] for i in instances]
        
        if instance_ids:
            self.ec2.stop_instances(InstanceIds=instance_ids)
            logger.info(f"Stopped instances: {instance_ids}")
        
        return instance_ids
    
    def run_command_on_instances(
        self, 
        instance_ids: List[str],
        commands: List[str]
    ) -> Dict:
        """Run commands on instances via SSM"""
        response = self.ssm.send_command(
            InstanceIds=instance_ids,
            DocumentName='AWS-RunShellScript',
            Parameters={'commands': commands},
            TimeoutSeconds=300
        )
        
        command_id = response['Command']['CommandId']
        
        # Wait for completion
        import time
        for _ in range(60):
            result = self.ssm.list_command_invocations(
                CommandId=command_id,
                Details=True
            )
            
            statuses = [i['Status'] for i in result['CommandInvocations']]
            if all(s in ['Success', 'Failed'] for s in statuses):
                break
            time.sleep(5)
        
        return result
    
    # ==================== S3 OPERATIONS ====================
    
    def sync_directory(
        self, 
        local_path: str, 
        bucket: str, 
        prefix: str = ''
    ) -> int:
        """Sync local directory to S3"""
        from pathlib import Path
        import hashlib
        
        local_path = Path(local_path)
        uploaded = 0
        
        for file_path in local_path.rglob('*'):
            if file_path.is_file():
                # Calculate S3 key
                relative_path = file_path.relative_to(local_path)
                s3_key = f"{prefix}/{relative_path}" if prefix else str(relative_path)
                
                # Check if upload needed (compare etag/md5)
                try:
                    head = self.s3.head_object(Bucket=bucket, Key=s3_key)
                    existing_etag = head['ETag'].strip('"')
                    
                    with open(file_path, 'rb') as f:
                        local_md5 = hashlib.md5(f.read()).hexdigest()
                    
                    if existing_etag == local_md5:
                        continue  # Skip unchanged files
                except ClientError:
                    pass  # File doesn't exist, upload it
                
                # Upload file
                self.s3.upload_file(str(file_path), bucket, s3_key)
                uploaded += 1
                logger.info(f"Uploaded: {s3_key}")
        
        return uploaded
    
    def cleanup_old_objects(
        self, 
        bucket: str, 
        prefix: str,
        days_old: int = 30
    ) -> int:
        """Delete S3 objects older than specified days"""
        paginator = self.s3.get_paginator('list_objects_v2')
        cutoff = datetime.now(datetime.timezone.utc) - timedelta(days=days_old)
        
        deleted = 0
        for page in paginator.paginate(Bucket=bucket, Prefix=prefix):
            for obj in page.get('Contents', []):
                if obj['LastModified'] < cutoff:
                    self.s3.delete_object(Bucket=bucket, Key=obj['Key'])
                    deleted += 1
                    logger.info(f"Deleted: {obj['Key']}")
        
        return deleted
    
    # ==================== CLOUDWATCH OPERATIONS ====================
    
    def get_metric_statistics(
        self,
        namespace: str,
        metric_name: str,
        dimensions: List[Dict],
        period: int = 300,
        hours: int = 1
    ) -> List[Dict]:
        """Get CloudWatch metric statistics"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)
        
        response = self.cloudwatch.get_metric_statistics(
            Namespace=namespace,
            MetricName=metric_name,
            Dimensions=dimensions,
            StartTime=start_time,
            EndTime=end_time,
            Period=period,
            Statistics=['Average', 'Maximum', 'Minimum']
        )
        
        return sorted(response['Datapoints'], key=lambda x: x['Timestamp'])
    
    def create_alarm(
        self,
        alarm_name: str,
        metric_name: str,
        namespace: str,
        threshold: float,
        comparison: str = 'GreaterThanThreshold',
        sns_topic_arn: Optional[str] = None
    ) -> None:
        """Create CloudWatch alarm"""
        alarm_config = {
            'AlarmName': alarm_name,
            'MetricName': metric_name,
            'Namespace': namespace,
            'Statistic': 'Average',
            'Period': 300,
            'EvaluationPeriods': 2,
            'Threshold': threshold,
            'ComparisonOperator': comparison,
            'TreatMissingData': 'notBreaching'
        }
        
        if sns_topic_arn:
            alarm_config['AlarmActions'] = [sns_topic_arn]
        
        self.cloudwatch.put_metric_alarm(**alarm_config)
        logger.info(f"Created alarm: {alarm_name}")


# ==================== USAGE EXAMPLE ====================
if __name__ == '__main__':
    aws = AWSAutomation(region='us-east-1')
    
    # Find production instances
    prod_instances = aws.find_instances(
        filters=[{'Name': 'tag:Environment', 'Values': ['production']}]
    )
    
    for instance in prod_instances:
        print(f"{instance['name']}: {instance['id']} ({instance['state']})")
    
    # Get CPU metrics for an instance
    metrics = aws.get_metric_statistics(
        namespace='AWS/EC2',
        metric_name='CPUUtilization',
        dimensions=[{'Name': 'InstanceId', 'Value': 'i-1234567890abcdef0'}],
        hours=24
    )
    
    # Create high CPU alarm
    aws.create_alarm(
        alarm_name='HighCPU-Production',
        metric_name='CPUUtilization',
        namespace='AWS/EC2',
        threshold=80.0,
        sns_topic_arn='arn:aws:sns:us-east-1:123456789:alerts'
    )
```

---

## Kubernetes Client

### 🔴 Advanced Questions

#### Q4: Write Python code to interact with Kubernetes clusters.

**Advanced Answer:**

```python
"""
Kubernetes Automation with Python
Using the official kubernetes-client library
"""

from kubernetes import client, config, watch
from kubernetes.client.rest import ApiException
from typing import List, Dict, Optional
import logging
import time
import yaml

logger = logging.getLogger(__name__)


class KubernetesOperations:
    """Kubernetes cluster operations"""
    
    def __init__(self, kubeconfig_path: Optional[str] = None):
        """Initialize Kubernetes client"""
        try:
            if kubeconfig_path:
                config.load_kube_config(config_file=kubeconfig_path)
            else:
                # Try in-cluster config first, fall back to kubeconfig
                try:
                    config.load_incluster_config()
                except config.ConfigException:
                    config.load_kube_config()
        except Exception as e:
            logger.error(f"Failed to load kubeconfig: {e}")
            raise
        
        self.core_v1 = client.CoreV1Api()
        self.apps_v1 = client.AppsV1Api()
        self.batch_v1 = client.BatchV1Api()
        self.custom_objects = client.CustomObjectsApi()
    
    # ==================== NAMESPACE OPERATIONS ====================
    
    def list_namespaces(self) -> List[str]:
        """List all namespaces"""
        namespaces = self.core_v1.list_namespace()
        return [ns.metadata.name for ns in namespaces.items]
    
    def create_namespace(
        self, 
        name: str, 
        labels: Optional[Dict] = None
    ) -> bool:
        """Create a namespace"""
        namespace = client.V1Namespace(
            metadata=client.V1ObjectMeta(
                name=name,
                labels=labels or {}
            )
        )
        
        try:
            self.core_v1.create_namespace(namespace)
            logger.info(f"Created namespace: {name}")
            return True
        except ApiException as e:
            if e.status == 409:  # Already exists
                logger.warning(f"Namespace {name} already exists")
                return True
            raise
    
    # ==================== DEPLOYMENT OPERATIONS ====================
    
    def get_deployment(
        self, 
        name: str, 
        namespace: str = 'default'
    ) -> Optional[Dict]:
        """Get deployment details"""
        try:
            deployment = self.apps_v1.read_namespaced_deployment(name, namespace)
            return {
                'name': deployment.metadata.name,
                'namespace': deployment.metadata.namespace,
                'replicas': deployment.spec.replicas,
                'ready_replicas': deployment.status.ready_replicas or 0,
                'image': deployment.spec.template.spec.containers[0].image,
                'labels': deployment.metadata.labels
            }
        except ApiException as e:
            if e.status == 404:
                return None
            raise
    
    def scale_deployment(
        self, 
        name: str, 
        replicas: int, 
        namespace: str = 'default'
    ) -> bool:
        """Scale a deployment"""
        body = {'spec': {'replicas': replicas}}
        
        try:
            self.apps_v1.patch_namespaced_deployment_scale(
                name=name,
                namespace=namespace,
                body=body
            )
            logger.info(f"Scaled {name} to {replicas} replicas")
            return True
        except ApiException as e:
            logger.error(f"Failed to scale deployment: {e}")
            return False
    
    def update_image(
        self, 
        name: str, 
        image: str, 
        namespace: str = 'default',
        container_name: Optional[str] = None
    ) -> bool:
        """Update deployment image"""
        deployment = self.apps_v1.read_namespaced_deployment(name, namespace)
        
        # Find container to update
        containers = deployment.spec.template.spec.containers
        target = container_name or containers[0].name
        
        for container in containers:
            if container.name == target:
                container.image = image
                break
        
        try:
            self.apps_v1.patch_namespaced_deployment(
                name=name,
                namespace=namespace,
                body=deployment
            )
            logger.info(f"Updated {name} to image {image}")
            return True
        except ApiException as e:
            logger.error(f"Failed to update image: {e}")
            return False
    
    def wait_for_rollout(
        self, 
        name: str, 
        namespace: str = 'default',
        timeout: int = 300
    ) -> bool:
        """Wait for deployment rollout to complete"""
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            deployment = self.apps_v1.read_namespaced_deployment(name, namespace)
            
            replicas = deployment.spec.replicas
            ready = deployment.status.ready_replicas or 0
            updated = deployment.status.updated_replicas or 0
            
            if ready == replicas == updated:
                logger.info(f"Deployment {name} rolled out successfully")
                return True
            
            logger.info(f"Waiting for rollout: {ready}/{replicas} ready")
            time.sleep(5)
        
        logger.error(f"Rollout timeout for {name}")
        return False
    
    # ==================== POD OPERATIONS ====================
    
    def list_pods(
        self, 
        namespace: str = 'default',
        label_selector: Optional[str] = None
    ) -> List[Dict]:
        """List pods in namespace"""
        pods = self.core_v1.list_namespaced_pod(
            namespace=namespace,
            label_selector=label_selector
        )
        
        return [{
            'name': pod.metadata.name,
            'namespace': pod.metadata.namespace,
            'status': pod.status.phase,
            'ip': pod.status.pod_ip,
            'node': pod.spec.node_name,
            'containers': [c.name for c in pod.spec.containers],
            'restarts': sum(
                cs.restart_count 
                for cs in (pod.status.container_statuses or [])
            )
        } for pod in pods.items]
    
    def get_pod_logs(
        self, 
        name: str, 
        namespace: str = 'default',
        container: Optional[str] = None,
        tail_lines: int = 100
    ) -> str:
        """Get pod logs"""
        return self.core_v1.read_namespaced_pod_log(
            name=name,
            namespace=namespace,
            container=container,
            tail_lines=tail_lines
        )
    
    def exec_command(
        self, 
        name: str, 
        command: List[str],
        namespace: str = 'default',
        container: Optional[str] = None
    ) -> str:
        """Execute command in pod"""
        from kubernetes.stream import stream
        
        response = stream(
            self.core_v1.connect_get_namespaced_pod_exec,
            name,
            namespace,
            command=command,
            container=container,
            stderr=True,
            stdin=False,
            stdout=True,
            tty=False
        )
        return response
    
    # ==================== WATCH EVENTS ====================
    
    def watch_pods(
        self, 
        namespace: str = 'default',
        timeout: int = 60
    ):
        """Watch pod events"""
        w = watch.Watch()
        
        for event in w.stream(
            self.core_v1.list_namespaced_pod,
            namespace=namespace,
            timeout_seconds=timeout
        ):
            event_type = event['type']
            pod = event['object']
            yield {
                'type': event_type,
                'name': pod.metadata.name,
                'status': pod.status.phase
            }
    
    # ==================== APPLY MANIFEST ====================
    
    def apply_manifest(self, manifest: str) -> bool:
        """Apply YAML manifest (like kubectl apply)"""
        documents = yaml.safe_load_all(manifest)
        
        for doc in documents:
            if not doc:
                continue
            
            kind = doc.get('kind')
            name = doc['metadata']['name']
            namespace = doc['metadata'].get('namespace', 'default')
            
            try:
                if kind == 'Deployment':
                    try:
                        self.apps_v1.read_namespaced_deployment(name, namespace)
                        self.apps_v1.patch_namespaced_deployment(
                            name, namespace, doc
                        )
                        logger.info(f"Updated deployment: {name}")
                    except ApiException as e:
                        if e.status == 404:
                            self.apps_v1.create_namespaced_deployment(
                                namespace, doc
                            )
                            logger.info(f"Created deployment: {name}")
                        else:
                            raise
                            
                elif kind == 'Service':
                    try:
                        self.core_v1.read_namespaced_service(name, namespace)
                        self.core_v1.patch_namespaced_service(
                            name, namespace, doc
                        )
                        logger.info(f"Updated service: {name}")
                    except ApiException as e:
                        if e.status == 404:
                            self.core_v1.create_namespaced_service(
                                namespace, doc
                            )
                            logger.info(f"Created service: {name}")
                        else:
                            raise
                            
                # Add more resource types as needed
                
            except ApiException as e:
                logger.error(f"Failed to apply {kind}/{name}: {e}")
                return False
        
        return True


# ==================== USAGE EXAMPLE ====================
if __name__ == '__main__':
    k8s = KubernetesOperations()
    
    # List all pods in default namespace
    pods = k8s.list_pods(namespace='default')
    for pod in pods:
        print(f"{pod['name']}: {pod['status']} ({pod['restarts']} restarts)")
    
    # Scale deployment
    k8s.scale_deployment('my-app', replicas=3, namespace='production')
    
    # Update image and wait for rollout
    k8s.update_image('my-app', 'my-registry/my-app:v2.0.0', namespace='production')
    success = k8s.wait_for_rollout('my-app', namespace='production')
    
    if success:
        print("Deployment successful!")
    else:
        print("Deployment failed, consider rollback")
```

---

## Testing & Quality

### 🟡 Intermediate Questions

#### Q5: How do you test DevOps Python scripts?

**Advanced Answer:**

```python
"""
Testing DevOps Python Code
Unit tests, integration tests, and mocking
"""

import pytest
from unittest.mock import Mock, patch, MagicMock
import responses
import boto3
from moto import mock_s3, mock_ec2


# ==================== UNIT TESTS WITH MOCKING ====================

class TestServerProvisioner:
    """Test server provisioning logic"""
    
    @patch('subprocess.run')
    def test_run_remote_command_success(self, mock_run):
        """Test successful SSH command execution"""
        # Arrange
        mock_run.return_value = Mock(
            returncode=0,
            stdout='success',
            stderr=''
        )
        
        from server_provisioner import ServerProvisioner, ServerConfig
        server = ServerConfig('test', '10.0.0.1', 'web')
        provisioner = ServerProvisioner([server])
        
        # Act
        returncode, stdout, stderr = provisioner.run_remote_command(
            server, 'echo hello'
        )
        
        # Assert
        assert returncode == 0
        assert stdout == 'success'
        mock_run.assert_called_once()
    
    @patch('subprocess.run')
    def test_run_remote_command_timeout(self, mock_run):
        """Test SSH command timeout handling"""
        import subprocess
        mock_run.side_effect = subprocess.TimeoutExpired('ssh', 60)
        
        from server_provisioner import ServerProvisioner, ServerConfig
        server = ServerConfig('test', '10.0.0.1', 'web')
        provisioner = ServerProvisioner([server])
        
        returncode, stdout, stderr = provisioner.run_remote_command(
            server, 'sleep 100'
        )
        
        assert returncode == -1
        assert 'timed out' in stderr.lower()


# ==================== AWS TESTS WITH MOTO ====================

@mock_s3
class TestAWSS3Operations:
    """Test S3 operations with moto mock"""
    
    def test_sync_directory(self, tmp_path):
        """Test S3 sync functionality"""
        # Create mock S3 bucket
        s3 = boto3.client('s3', region_name='us-east-1')
        s3.create_bucket(Bucket='test-bucket')
        
        # Create test files
        (tmp_path / 'file1.txt').write_text('content1')
        (tmp_path / 'subdir').mkdir()
        (tmp_path / 'subdir' / 'file2.txt').write_text('content2')
        
        # Run sync
        from aws_automation import AWSAutomation
        aws = AWSAutomation(region='us-east-1')
        uploaded = aws.sync_directory(str(tmp_path), 'test-bucket', 'prefix')
        
        # Verify
        assert uploaded == 2
        response = s3.list_objects_v2(Bucket='test-bucket', Prefix='prefix')
        keys = [obj['Key'] for obj in response['Contents']]
        assert 'prefix/file1.txt' in keys
        assert 'prefix/subdir/file2.txt' in keys


@mock_ec2
class TestAWSEC2Operations:
    """Test EC2 operations with moto mock"""
    
    def test_find_instances_by_tag(self):
        """Test finding instances by tag"""
        # Create mock EC2 instances
        ec2 = boto3.resource('ec2', region_name='us-east-1')
        instances = ec2.create_instances(
            ImageId='ami-12345',
            MinCount=2,
            MaxCount=2,
            InstanceType='t2.micro',
            TagSpecifications=[{
                'ResourceType': 'instance',
                'Tags': [
                    {'Key': 'Environment', 'Value': 'production'},
                    {'Key': 'Name', 'Value': 'test-server'}
                ]
            }]
        )
        
        # Test
        from aws_automation import AWSAutomation
        aws = AWSAutomation(region='us-east-1')
        found = aws.find_instances(
            filters=[{'Name': 'tag:Environment', 'Values': ['production']}]
        )
        
        assert len(found) == 2
        assert all(i['tags']['Environment'] == 'production' for i in found)


# ==================== API TESTS WITH RESPONSES ====================

@responses.activate
def test_health_check():
    """Test HTTP health check with mocked responses"""
    # Mock the endpoint
    responses.add(
        responses.GET,
        'http://service:8080/health',
        json={'status': 'healthy'},
        status=200
    )
    
    # Test
    import requests
    response = requests.get('http://service:8080/health')
    
    assert response.status_code == 200
    assert response.json()['status'] == 'healthy'


# ==================== KUBERNETES TESTS ====================

class TestKubernetesOperations:
    """Test Kubernetes operations"""
    
    @patch('kubernetes.config.load_kube_config')
    @patch('kubernetes.client.CoreV1Api')
    def test_list_namespaces(self, mock_core_api, mock_config):
        """Test listing namespaces"""
        # Setup mock
        mock_ns1 = Mock()
        mock_ns1.metadata.name = 'default'
        mock_ns2 = Mock()
        mock_ns2.metadata.name = 'kube-system'
        
        mock_api_instance = Mock()
        mock_api_instance.list_namespace.return_value = Mock(
            items=[mock_ns1, mock_ns2]
        )
        mock_core_api.return_value = mock_api_instance
        
        # Test
        from k8s_automation import KubernetesOperations
        k8s = KubernetesOperations()
        namespaces = k8s.list_namespaces()
        
        assert 'default' in namespaces
        assert 'kube-system' in namespaces


# ==================== FIXTURES ====================

@pytest.fixture
def sample_server_config():
    """Fixture for server configuration"""
    from server_provisioner import ServerConfig
    return ServerConfig(
        name='test-server',
        ip='10.0.0.1',
        role='web',
        ssh_user='ubuntu',
        ssh_key='~/.ssh/test_key'
    )


@pytest.fixture
def mock_kubernetes_client():
    """Fixture for mocked Kubernetes client"""
    with patch('kubernetes.config.load_kube_config'), \
         patch('kubernetes.client.CoreV1Api') as mock_core, \
         patch('kubernetes.client.AppsV1Api') as mock_apps:
        yield {
            'core': mock_core.return_value,
            'apps': mock_apps.return_value
        }


# ==================== RUN TESTS ====================
# pytest tests/ -v --cov=. --cov-report=html
```

---

## 📚 Documentation Links

- [Python Documentation](https://docs.python.org/3/)
- [boto3 Documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/index.html)
- [Kubernetes Python Client](https://github.com/kubernetes-client/python)
- [pytest Documentation](https://docs.pytest.org/)
- [asyncio Documentation](https://docs.python.org/3/library/asyncio.html)

---

**[← Back to Main README](../README.md)** | **[Back to Top ↑](#python-for-devops-interview-questions---complete-guide)**
