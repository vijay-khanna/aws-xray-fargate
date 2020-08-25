
Ensure the Cloud9-> Preferences-AWS Settings -> 'AWS Managed Security Credentials' are switched off. and the Cloud9 Instance has Admin role assigned. 

Ask for the Variables as Subnet, Unique Project Name

```
read -p "Enter a unique Project Name to identify all resources: " XRay_Project_Name ; 

echo $XRay_Project_Name

XRay_cluster_name=${XRay_Project_Name}-cluster; echo $cluster_name

####### Create Cluster. Run once if Cloud9 is already used for this Demo. #Fast-Run-Command
aws ecs create-cluster --cluster-name $XRay_cluster_name --region us-east-1
aws ecs update-cluster-settings --cluster $XRay_cluster_name --settings name=containerInsights,value=enabled


echo "export XRay_Project_Name=${XRay_Project_Name}" >> ~/.bash_profile
echo "export XRay_cluster_name=${XRay_cluster_name}" >> ~/.bash_profile


sudo yum -y install jq gettext
cd ~/environment
git clone https://github.com/vijay-khanna/aws-xray-fargate.git
cd aws-xray-fargate/src

## Run aws configure, just specify the region
aws configure

```

Create a task role that allows the task to write traces to AWS X-Ray.  Replace *<role_name>* with your role name. 

```
export XRay_role_name=${XRay_Project_Name}-role ; echo $XRay_role_name
export XRay_policy_name=${XRay_Project_Name}-policy ; echo $XRay_policy_name

echo "export XRay_role_name=${XRay_role_name}" >> ~/.bash_profile
echo "export XRay_policy_name=${XRay_policy_name}" >> ~/.bash_profile

export XRay_TASK_ROLE_NAME=$(aws iam create-role --role-name $XRay_role_name --assume-role-policy-document file://ecs-trust-pol.json | jq -r '.Role.RoleName')
export XRAY_POLICY_ARN=$(aws iam create-policy --policy-name $XRay_policy_name --policy-document file://xray-pol.json | jq -r '.Policy.Arn')

echo "export XRay_TASK_ROLE_NAME=${XRay_TASK_ROLE_NAME}" >> ~/.bash_profile
echo "export XRAY_POLICY_ARN=${XRAY_POLICY_ARN}" >> ~/.bash_profile

aws iam attach-role-policy --role-name $TASK_ROLE_NAME --policy-arn $XRAY_POLICY_ARN
```

If this is your first time using ECS you will need to create the ECS Task Execution Role.

```
aws iam create-role --role-name ecsTaskExecutionRole --assume-role-policy-document file://ecs-trust-pol.json

export XRay_ECS_EXECUTION_POLICY_ARN=$(aws iam list-policies --scope AWS --query 'Policies[?PolicyName==`AmazonECSTaskExecutionRolePolicy`].Arn' | jq -r '.[]')
echo $XRay_ECS_EXECUTION_POLICY_ARN
echo "export XRay_ECS_EXECUTION_POLICY_ARN=${XRay_ECS_EXECUTION_POLICY_ARN}" >> ~/.bash_profile

aws iam attach-role-policy --role-name ecsTaskExecutionRole --policy-arn $XRay_ECS_EXECUTION_POLICY_ARN
```

Export the Arns of the task role and the task execution role. 

```
export XRay_TASK_ROLE_ARN=$(aws iam get-role --role-name $XRay_role_name --query "Role.Arn" --output text) ; echo $TASK_ROLE_ARN
export XRay_TASK_EXECUTION_ROLE_ARN=$(aws iam get-role --role-name ecsTaskExecutionRole --query "Role.Arn" --output text) ; echo $XRay_TASK_EXECUTION_ROLE_ARN

echo "export XRay_TASK_ROLE_ARN=${XRay_TASK_ROLE_ARN}" >> ~/.bash_profile
echo "export XRay_TASK_EXECUTION_ROLE_ARN=${XRay_TASK_EXECUTION_ROLE_ARN}" >> ~/.bash_profile



```

Choose at least 2 subnets to set as environment variables.  These will be used to populate the ecs-params.yml file.
Enter the VPC ID, and Two Subnets for Containers. 

Ensure there is no whte space / Enter symbol in the copy-paste of VPC/Subnets. 

```

read -p "Enter the VPC ID : " XRay_vpc_id ; echo $XRay_vpc_id

read -p "Enter the SUBNET_ID_1 : " XRay_SUBNET_ID_1 ; echo $XRay_SUBNET_ID_1

read -p "Enter the SUBNET_ID_2 ID : " XRay_SUBNET_ID_2 ; echo $XRay_SUBNET_ID_2



export XRay_vpc_id ; 
export XRay_SUBNET_ID_1=$XRay_SUBNET_ID_1
export XRay_SUBNET_ID_2=$XRay_SUBNET_ID_2
echo "export XRay_vpc_id=${XRay_vpc_id}" >> ~/.bash_profile
echo "export XRay_SUBNET_ID_1=${XRay_SUBNET_ID_1}" >> ~/.bash_profile
echo "export XRay_SUBNET_ID_2=${XRay_SUBNET_ID_2}" >> ~/.bash_profile



```

Create a security group. Replace *<group_name>*, *<description_text>*, and *<vpc_id>* with the appropriate values. The *<vpc_id>* should match the vpc id you used earlier. 

```

export XRay_sec_group_name=${XRay_Project_Name}-sec-group ; echo $XRay_sec_group_name

export XRay_SG_ID=$(aws ec2 create-security-group --group-name $XRay_sec_group_name --description sec-grp-for-containers --vpc-id $XRay_vpc_id | jq -r '.GroupId')
echo $XRay_sec_group_name
echo $XRay_SG_ID
echo "export XRay_sec_group_name=${XRay_sec_group_name}" >> ~/.bash_profile
echo "export XRay_SG_ID=${XRay_SG_ID}" >> ~/.bash_profile



```

Add the following inbound rules to the security group.

```
aws ec2 authorize-security-group-ingress --group-id $XRay_SG_ID --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $XRay_SG_ID --protocol all --port all --source-group $XRay_SG_ID
```

Create an Application Load Balancer (ALB), listener, and target group for service B.

```
## Fast-Run-Command. Run once if pre-run the same demo on Cloud9. 
export XRay_load_balancer_name_b=${XRay_Project_Name}-lb-b ; echo $XRay_load_balancer_name_b
export XRay_target_group_name_b=${XRay_Project_Name}-tg-b ; echo $XRay_target_group_name_b

export XRay_LOAD_BALANCER_ARN_B=$(aws elbv2 create-load-balancer --name $XRay_load_balancer_name_b --subnets $XRay_SUBNET_ID_1 $XRay_SUBNET_ID_2 --security-groups $XRay_SG_ID --scheme internet-facing --type application | jq -r '.LoadBalancers[].LoadBalancerArn')

echo $XRay_LOAD_BALANCER_ARN_B


export XRay_TARGET_GROUP_ARN_B=$(aws elbv2 create-target-group --name $XRay_target_group_name_b --protocol HTTP --port 8080 --vpc-id $XRay_vpc_id --target-type ip --health-check-path /health | jq -r '.TargetGroups[].TargetGroupArn') ; echo $XRay_TARGET_GROUP_ARN_B

aws elbv2 create-listener --load-balancer-arn $XRay_LOAD_BALANCER_ARN_B --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$XRay_TARGET_GROUP_ARN_B

```

Get the DNS name of the load balancer. 

```
export XRay_SERVICE_B_ENDPOINT=$(aws elbv2 describe-load-balancers --load-balancer-arn $XRay_LOAD_BALANCER_ARN_B | jq -r '.LoadBalancers[].DNSName') ; echo $XRay_SERVICE_B_ENDPOINT
echo $XRay_SERVICE_B_ENDPOINT


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
export XRay_REGISTRY_URL_SERVICE_B=$(aws ecr describe-repositories --repository-name service-b | jq -r '.repositories[].repositoryUri')

export XRay_REGISTRY_URL_SERVICE_A=$(aws ecr describe-repositories --repository-name service-a | jq -r '.repositories[].repositoryUri')

echo $XRay_REGISTRY_URL_SERVICE_B
echo $XRay_REGISTRY_URL_SERVICE_A
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

ecs-cli compose service up --deployment-max-percent 100 --deployment-min-healthy-percent 0 --target-group-arn $XRay_TARGET_GROUP_ARN_B --launch-type FARGATE --container-name service-b --container-port 8080 --cluster $XRay_cluster_name


```

Create an Application Load Balancer (ALB), listener, and target group for service A.

```
export XRay_load_balancer_name_a=${XRay_Project_Name}-lb-a ; echo $XRay_load_balancer_name_a
export XRay_target_group_name_a=${XRay_Project_Name}-tg-a ; echo $XRay_target_group_name_a

export XRay_LOAD_BALANCER_ARN_A=$(aws elbv2 create-load-balancer --name $XRay_load_balancer_name_a --subnets $XRay_SUBNET_ID_1 $XRay_SUBNET_ID_2 --security-groups $XRay_SG_ID --scheme internet-facing --type application | jq -r '.LoadBalancers[].LoadBalancerArn') ; echo $XRay_LOAD_BALANCER_ARN_A

export XRay_TARGET_GROUP_ARN_A=$(aws elbv2 create-target-group --name $XRay_target_group_name_a --protocol HTTP --port 8080 --vpc-id $XRay_vpc_id --target-type ip --health-check-path /health | jq -r '.TargetGroups[].TargetGroupArn') ; echo $XRay_TARGET_GROUP_ARN_A

aws elbv2 create-listener --load-balancer-arn $XRay_LOAD_BALANCER_ARN_A --protocol HTTP --port 80 --default-actions Type=forward,TargetGroupArn=$XRay_TARGET_GROUP_ARN_A
```

Create service A. 

```
cd ../service-a/

envsubst < docker-compose.yml-template > docker-compose.yml
envsubst < ecs-params.yml-template > ecs-params.yml

ecs-cli compose service up --deployment-max-percent 100 --deployment-min-healthy-percent 0 --target-group-arn $XRay_TARGET_GROUP_ARN_A --launch-type FARGATE --container-name service-a --container-port 8080 --cluster $XRay_cluster_name
```

Simulate Load on LB-A, and Open the X-Ray console.
```
XRay_ALB_A_DNS=$(aws elbv2 describe-load-balancers --names $XRay_load_balancer_name_a |  jq -r '.LoadBalancers[].DNSName') ; echo $XRay_ALB_A_DNS

for ((i=1;i<=100;i++)); do   curl $XRay_ALB_A_DNS; echo " $i \n"; sleep 5 ; done
```
