
Ensure the Cloud9-> Preferences-AWS Settings -> 'AWS Managed Security Credentials' are switched off. and the Cloud9 Instance has Admin role assigned. 

Ask for the Variables as Subnet, Unique Project Name

```
read -p "Enter a unique Project Name to identify all resources: " Project_Name ; 

echo $Project_Name

cluster_name=${Project_Name}-cluster; echo $cluster_name
aws ecs create-cluster --cluster-name $cluster_name --region us-east-1

sudo yum -y install jq gettext

cd ~/environment
git clone https://github.com/vijay-khanna/aws-xray-fargate.git
cd aws-xray-fargate/src

```

Create a task role that allows the task to write traces to AWS X-Ray.  Replace *<role_name>* with your role name. 

```
export role_name=${Project_Name}-role ; echo $role_name
export policy_name=${Project_Name}-policy ; echo $policy_name


export TASK_ROLE_NAME=$(aws iam create-role --role-name $role_name --assume-role-policy-document file://ecs-trust-pol.json | jq -r '.Role.RoleName')
export XRAY_POLICY_ARN=$(aws iam create-policy --policy-name $policy_name --policy-document file://xray-pol.json | jq -r '.Policy.Arn')
aws iam attach-role-policy --role-name $TASK_ROLE_NAME --policy-arn $XRAY_POLICY_ARN
```

If this is your first time using ECS you will need to create the ECS Task Execution Role.

```
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://ecs-trust-pol.json

export ECS_EXECUTION_POLICY_ARN=$(aws iam list-policies --scope AWS --query 'Policies[?PolicyName==`AmazonECSTaskExecutionRolePolicy`].Arn' | jq -r '.[]')

aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn $ECS_EXECUTION_POLICY_ARN
```

Export the Arns of the task role and the task execution role. 

```
export TASK_ROLE_ARN=$(aws iam get-role --role-name $role_name --query "Role.Arn" --output text) ; echo $TASK_ROLE_ARN

export TASK_EXECUTION_ROLE_ARN=$(aws iam get-role --role-name ecsTaskExecutionRole --query "Role.Arn" --output text) ; echo $TASK_EXECUTION_ROLE_ARN

```

Choose at least 2 subnets to set as environment variables.  These will be used to populate the ecs-params.yml file.
Enter the VPC ID, and Two Subnets for Containers. 

Ensure there is no whte space / Enter symbol in the copy-paste of VPC/Subnets. 

```

read -p "Enter the VPC ID : " vpc_id ; echo $vpc_id

read -p "Enter the SUBNET_ID_1 : " SUBNET_ID_1 ; echo $SUBNET_ID_1

read -p "Enter the SUBNET_ID_2 ID : " SUBNET_ID_2 ; echo $SUBNET_ID_2

export vpc_id ; 
export SUBNET_ID_1=$SUBNET_ID_1
export SUBNET_ID_2=$SUBNET_ID_2
```

Create a security group. Replace *<group_name>*, *<description_text>*, and *<vpc_id>* with the appropriate values. The *<vpc_id>* should match the vpc id you used earlier. 

```

export sec_group_name=${Project_Name}-sec-group ; echo $sec_group_name

export SG_ID=$(aws ec2 create-security-group --group-name $sec_group_name --description sec-grp-for-containers --vpc-id $vpc_id | jq -r '.GroupId')
```

Add the following inbound rules to the security group.

```
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol all --port all --source-group $SG_ID
```

Create an Application Load Balancer (ALB), listener, and target group for service B.

```
export load_balancer_name_b=${Project_Name}-lb-b ; echo $load_balancer_name_b
export target_group_name_b=${Project_Name}-tg-b ; echo $target_group_name_b

export LOAD_BALANCER_ARN_B=$(aws elbv2 create-load-balancer --name $load_balancer_name_b --subnets $SUBNET_ID_1 $SUBNET_ID_2 --security-groups $SG_ID --scheme internet-facing --type application | jq -r '.LoadBalancers[].LoadBalancerArn')


export TARGET_GROUP_ARN_B=$(aws elbv2 create-target-group --name $target_group_name_b --protocol HTTP --port 8080 --vpc-id $vpc_id --target-type ip --health-check-path /health | jq -r '.TargetGroups[].TargetGroupArn')

aws elbv2 create-listener --load-balancer-arn $LOAD_BALANCER_ARN_B --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN_B
```

Get the DNS name of the load balancer. 

```
export SERVICE_B_ENDPOINT=$(aws elbv2 describe-load-balancers --load-balancer-arn $LOAD_BALANCER_ARN_B | jq -r '.LoadBalancers[].DNSName') ; echo $SERVICE_B_ENDPOINT
```

Install ECS CLI

```
sudo curl -o /usr/local/bin/ecs-cli https://amazon-ecs-cli.s3.amazonaws.com/ecs-cli-linux-amd64-latest
gpg --keyserver hkp://keys.gnupg.net --recv BCE9D9A42D51784F
curl -o ecs-cli.asc https://amazon-ecs-cli.s3.amazonaws.com/ecs-cli-linux-amd64-latest.asc

gpg --verify ecs-cli.asc /usr/local/bin/ecs-cli

sudo chmod +x /usr/local/bin/ecs-cli

ecs-cli --version


```

Build and push the containers to ECR.

```
cd ~/environment/aws-xray-fargate/src/service-b/


docker build -t service-b .
ecs-cli push service-b


cd ../service-a/
docker build -t service-a .
ecs-cli push service-a
```

Set the registry URLs

```
export REGISTRY_URL_SERVICE_B=$(aws ecr describe-repositories --repository-name service-b | jq -r '.repositories[].repositoryUri')

export REGISTRY_URL_SERVICE_A=$(aws ecr describe-repositories --repository-name service-a | jq -r '.repositories[].repositoryUri')
```

Create logs groups.

```
aws logs create-log-group --log-group-name /ecs/service-b
aws logs create-log-group --log-group-name /ecs/service-a
```

Create service B.

```
cd ../service-b/
envsubst < docker-compose.yml-template > docker-compose.yml
envsubst < ecs-params.yml-template > ecs-params.yml

ecs-cli compose service up --deployment-max-percent 100 --deployment-min-healthy-percent 0 --target-group-arn $TARGET_GROUP_ARN_B --launch-type FARGATE --container-name service-b --container-port 8080 --cluster $cluster_name

```

Create an Application Load Balancer (ALB), listener, and target group for service A.

```
export load_balancer_name_a=${Project_Name}-lb-a ; echo $load_balancer_name_a
export target_group_name_a=${Project_Name}-tg-a ; echo $target_group_name_a

export LOAD_BALANCER_ARN_A=$(aws elbv2 create-load-balancer --name $load_balancer_name_a --subnets $SUBNET_ID_1 $SUBNET_ID_2 --security-groups $SG_ID --scheme internet-facing --type application | jq -r '.LoadBalancers[].LoadBalancerArn')

export TARGET_GROUP_ARN_A=$(aws elbv2 create-target-group --name $target_group_name_a --protocol HTTP --port 8080 --vpc-id $vpc_id --target-type ip --health-check-path /health | jq -r '.TargetGroups[].TargetGroupArn')

aws elbv2 create-listener --load-balancer-arn $LOAD_BALANCER_ARN_A --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$TARGET_GROUP_ARN_A
```

Create service A. 

```
cd ../service-a/

envsubst < docker-compose.yml-template > docker-compose.yml
envsubst < ecs-params.yml-template > ecs-params.yml

ecs-cli compose service up --deployment-max-percent 100 --deployment-min-healthy-percent 0 --target-group-arn $TARGET_GROUP_ARN_A --launch-type FARGATE --container-name service-a --container-port 8080 --cluster $cluster_name
```

Simulate Load on LB-A, and Open the X-Ray console.
```
ALB_A_DNS=$(aws elbv2 describe-load-balancers --names $load_balancer_name_a |  jq -r '.LoadBalancers[].DNSName') ; echo $ALB_A_DNS

for ((i=1;i<=100;i++)); do   curl $ALB_A_DNS; echo " $i \n"; sleep 5 ; done
```
