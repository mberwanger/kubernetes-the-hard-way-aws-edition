# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

## Compute Instances

Delete the controller and worker compute instances:

```
INSTANCE_IDS=$(aws ec2 describe-instances \
  --output text \
  --filters "{\"Name\":\"tag:Name\", \"Values\":[\"controller-*\",\"worker-*\"]}" "{\"Name\":\"instance-state-name\", \"Values\":[\"running\"]}" \
  --query 'Reservations[*].Instances[*].InstanceId')
```

```
aws ec2 terminate-instances --instance-ids ${INSTANCE_IDS}
```

## Networking

Delete the external load balancer network resources:

```
LOAD_BALANCER_ARN=$(aws elbv2 describe-load-balancers \
  --output text --names kubernetes-the-hard-way --query 'LoadBalancers[*].LoadBalancerArn')
```

```
aws elbv2 delete-load-balancer --load-balancer-arn ${LOAD_BALANCER_ARN}
```

```
TARGET_GROUP_ARN=$(aws elbv2 describe-target-groups \
  --output text \
  --names kubernetes-targets \
  --query 'TargetGroups[*].TargetGroupArn')
```

```
aws elbv2 delete-target-group --target-group-arn ${TARGET_GROUP_ARN}
```

Delete the `kubernetes-the-hard-way` static IP address:

```
ELASTIC_IP=$(aws ec2 describe-addresses \
  --output text \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" \
  --query 'Addresses[*].AllocationId')
```

```
aws ec2 release-address --allocation-id ${ELASTIC_IP}
```

Delete the `kubernetes-the-hard-way` security groups:

```
EXT_SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --output text \
  --filters "Name=group-name,Values=kubernetes-the-hard-way-allow-external" \
  --query 'SecurityGroups[*].GroupId')
```

```
aws ec2 delete-security-group \
  --group-id ${EXT_SECURITY_GROUP_ID}
```

```
NGINX_SECURITY_GROUP_ID=$(aws ec2 describe-security-groups \
  --output text \
  --filters "Name=group-name,Values=kubernetes-the-hard-way-allow-nginx-service" \
  --query 'SecurityGroups[*].GroupId')
```

```
aws ec2 delete-security-group \
  --group-id ${NGINX_SECURITY_GROUP_ID}
```

Delete the `kubernetes` internet-gateway:

```
VPC_ID=$(aws ec2 describe-vpcs \
  --output text \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" \
  --query 'Vpcs[].VpcId')
```

```
INTERNET_GATEWAY_ID=$(aws ec2 describe-internet-gateways \
  --output text \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" \
  --query 'InternetGateways[*].InternetGatewayId')
```

```
aws ec2 detach-internet-gateway \
    --vpc-id ${VPC_ID} \
    --internet-gateway-id ${INTERNET_GATEWAY_ID}
```  

```
aws ec2 delete-internet-gateway \
  --internet-gateway-id ${INTERNET_GATEWAY_ID}
```

Delete the `kubernetes` subnet:

```
SUBNET_ID=$(aws ec2 describe-subnets \
  --output text \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" \
  --query 'Subnets[*].SubnetId')
```

```
aws ec2 delete-subnet \
  --subnet-id ${SUBNET_ID}
```

Delete the `kubernetes-the-hard-way` network VPC:

```
VPC_ID=$(aws ec2 describe-vpcs \
  --output text \
  --filters "Name=tag:Name,Values=kubernetes-the-hard-way" \
  --query 'Vpcs[].VpcId')
```

```
aws ec2 delete-vpc \
  --vpc-id ${VPC_ID}
```

Delete the `kubernetes-the-hard-way` key pair:

```
aws ec2 delete-key-pair \
  --key-name kubernetes-the-hard-way
```
