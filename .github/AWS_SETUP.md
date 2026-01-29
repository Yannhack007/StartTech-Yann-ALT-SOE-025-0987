# Infrastructure AWS Setup Guide

Ce guide explique comment configurer l'infrastructure AWS nécessaire pour déployer l'application MuchToDo Backend.

## 📋 Prérequis

- Compte AWS actif
- AWS CLI configuré
- Permissions administrateur ou IAM appropriées

## 🏗️ Ressources à créer

### 1. ECR Repository

```bash
aws ecr create-repository \
  --repository-name starttech-backend \
  --region eu-north-1 \
  --image-scanning-configuration scanOnPush=true
```

### 2. VPC et Sous-réseaux (si pas déjà existants)

```bash
# Créer un VPC
aws ec2 create-vpc \
  --cidr-block 10.0.0.0/16 \
  --region eu-north-1 \
  --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=starttech-vpc}]'

# Créer des sous-réseaux publics dans 2 AZs
aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 10.0.1.0/24 \
  --availability-zone eu-north-1a \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=starttech-public-1a}]'

aws ec2 create-subnet \
  --vpc-id <VPC_ID> \
  --cidr-block 10.0.2.0/24 \
  --availability-zone eu-north-1b \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=starttech-public-1b}]'
```

### 3. Security Groups

```bash
# Security Group pour ALB
aws ec2 create-security-group \
  --group-name starttech-alb-sg \
  --description "Security group for ALB" \
  --vpc-id <VPC_ID> \
  --region eu-north-1

# Autoriser HTTP/HTTPS
aws ec2 authorize-security-group-ingress \
  --group-id <ALB_SG_ID> \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
  --group-id <ALB_SG_ID> \
  --protocol tcp \
  --port 443 \
  --cidr 0.0.0.0/0

# Security Group pour EC2
aws ec2 create-security-group \
  --group-name starttech-backend-sg \
  --description "Security group for backend instances" \
  --vpc-id <VPC_ID> \
  --region eu-north-1

# Autoriser le trafic depuis l'ALB
aws ec2 authorize-security-group-ingress \
  --group-id <BACKEND_SG_ID> \
  --protocol tcp \
  --port 8080 \
  --source-group <ALB_SG_ID>
```

### 4. IAM Role pour EC2

```bash
# Créer le rôle
aws iam create-role \
  --role-name starttech-backend-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ec2.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attacher les policies
aws iam attach-role-policy \
  --role-name starttech-backend-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly

aws iam attach-role-policy \
  --role-name starttech-backend-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# Créer l'instance profile
aws iam create-instance-profile \
  --instance-profile-name starttech-backend-profile

aws iam add-role-to-instance-profile \
  --instance-profile-name starttech-backend-profile \
  --role-name starttech-backend-role
```

### 5. Target Group

```bash
aws elbv2 create-target-group \
  --name starttech-backend-tg \
  --protocol HTTP \
  --port 8080 \
  --vpc-id <VPC_ID> \
  --health-check-enabled \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --region eu-north-1
```

### 6. Application Load Balancer

```bash
aws elbv2 create-load-balancer \
  --name starttech-alb \
  --type application \
  --scheme internet-facing \
  --subnets <SUBNET_1_ID> <SUBNET_2_ID> \
  --security-groups <ALB_SG_ID> \
  --region eu-north-1

# Créer un listener
aws elbv2 create-listener \
  --load-balancer-arn <ALB_ARN> \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=<TARGET_GROUP_ARN>
```

### 7. Launch Template

```bash
aws ec2 create-launch-template \
  --launch-template-name starttech-backend-lt \
  --version-description "Initial version" \
  --launch-template-data '{
    "ImageId": "ami-0d0b75c8c47ed0d24",
    "InstanceType": "t3.micro",
    "IamInstanceProfile": {
      "Name": "starttech-backend-profile"
    },
    "SecurityGroupIds": ["<BACKEND_SG_ID>"],
    "TagSpecifications": [{
      "ResourceType": "instance",
      "Tags": [{
        "Key": "Name",
        "Value": "starttech-backend"
      }]
    }],
    "MetadataOptions": {
      "HttpTokens": "required",
      "HttpPutResponseHopLimit": 1
    }
  }' \
  --region eu-north-1 \
  --tag-specifications 'ResourceType=launch-template,Tags=[{Key=Name,Value=starttech-backend-lt}]'
```

**Note:** L'AMI `ami-0d0b75c8c47ed0d24` est Amazon Linux 2023 pour eu-north-1. Vérifiez l'AMI actuelle pour votre région.

### 8. Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name starttech-backend-asg \
  --launch-template LaunchTemplateName=starttech-backend-lt \
  --min-size 1 \
  --max-size 3 \
  --desired-capacity 1 \
  --target-group-arns <TARGET_GROUP_ARN> \
  --health-check-type ELB \
  --health-check-grace-period 300 \
  --vpc-zone-identifier "<SUBNET_1_ID>,<SUBNET_2_ID>" \
  --region eu-north-1 \
  --tags "Key=Name,Value=starttech-backend,PropagateAtLaunch=true"
```

## 🔍 Vérification

```bash
# Vérifier l'ECR
aws ecr describe-repositories --repository-names starttech-backend --region eu-north-1

# Vérifier le Launch Template
aws ec2 describe-launch-templates \
  --filters "Name=tag:Name,Values=starttech-backend-lt" \
  --region eu-north-1

# Vérifier l'ASG
aws autoscaling describe-auto-scaling-groups \
  --query "AutoScalingGroups[?contains(Tags[?Key=='Name'].Value, 'starttech-backend')]" \
  --region eu-north-1

# Vérifier l'ALB
aws elbv2 describe-load-balancers \
  --query "LoadBalancers[?contains(LoadBalancerName, 'starttech')]" \
  --region eu-north-1
```

## 💰 Estimation des coûts

- **EC2 t3.micro (1 instance):** ~$7.50/mois
- **Application Load Balancer:** ~$16.20/mois + données
- **ECR Storage:** $0.10/GB/mois
- **CloudWatch Logs:** $0.50/GB ingéré
- **Transfert de données:** Variable

**Total estimé:** ~$25-30/mois pour une configuration minimale

## 🗑️ Nettoyage (si besoin)

```bash
# Supprimer l'ASG
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name starttech-backend-asg \
  --force-delete

# Supprimer le Launch Template
aws ec2 delete-launch-template \
  --launch-template-name starttech-backend-lt

# Supprimer l'ALB
aws elbv2 delete-load-balancer --load-balancer-arn <ALB_ARN>

# Supprimer le Target Group
aws elbv2 delete-target-group --target-group-arn <TARGET_GROUP_ARN>

# Supprimer les Security Groups, IAM Roles, etc.
```

## 📚 Ressources supplémentaires

- [AWS EC2 Auto Scaling Documentation](https://docs.aws.amazon.com/autoscaling/)
- [AWS ECR Documentation](https://docs.aws.amazon.com/ecr/)
- [AWS ALB Documentation](https://docs.aws.amazon.com/elasticloadbalancing/)
