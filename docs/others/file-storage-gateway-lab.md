---
title: Storage File Gateway
description: create file storage gateway from cli
created: 2022-09-16 13:40:22.702
last_modified: 2023-12-24
tags:
  - aws/storage/storage-gateway
---

# Storage File Gateway 
## prepare
- create a cloud9 desktop for this lab
- ensure you have enough privilege to create resources

## create s3 bucket
```sh
AWS_REGION=us-east-1
BUCKET_NAME=fgwlab-$RANDOM
PREFIX_NAME=fgw
aws s3 mb s3://${BUCKET_NAME}
aws s3api put-object --bucket ${BUCKET_NAME} \
  --key ${PREFIX_NAME}/

```

## create fgw instance
```sh
# ensure this key existed
KEY_NAME=awskey

IMAGE_ID=$(aws ec2 describe-images --region ${AWS_REGION}  \
  --filters Name=name,Values='aws-storage-gateway-*'  \
  --query 'Images[*].[ImageId,CreationDate,Name]' --output text \
  |sort -k2 -r |head -n 1 |awk '{print $1}')
# another way to get IMAGE_ID
# aws ssm get-parameter --name /aws/service/storagegateway/ami/FILE_S3/latest

# cloud 9 subnet
INST_ID=$(curl http://169.254.169.254/1.0/meta-data/instance-id 2>/dev/null)
SUBNET_ID=$(aws ec2 describe-instances --instance-ids ${INST_ID} --query 'Reservations[0].Instances[0].SubnetId' --output text)

# create sg
FGW_SG_NAME=fgw-sg-$RANDOM
VPC_ID=$(aws ec2 describe-instances --instance-ids ${INST_ID} --query 'Reservations[0].Instances[0].VpcId' --output text)
FGW_SG_ID=$(aws ec2 create-security-group \
  --description ${FGW_SG_NAME} \
  --group-name ${FGW_SG_NAME} \
  --vpc-id ${VPC_ID} \
  --query 'GroupId' --output text )
# all traffic allowed
aws ec2 authorize-security-group-ingress \
  --group-name ${FGW_SG_NAME} \
  --protocol -1 \
  --port -1 \
  --cidr 0.0.0.0/0

FGW_INST_ID=$(aws ec2 run-instances --region ${AWS_REGION} --key-name ${KEY_NAME} \
  --image-id ${IMAGE_ID} --instance-type m5.xlarge \
  --block-device-mappings 'DeviceName=/dev/sdb,Ebs={VolumeSize=200}' \
  --subnet-id ${SUBNET_ID} --security-group-ids ${FGW_SG_ID} \
  --query Instances[*].InstanceId --output text )

# wait instance spin up
tmpfile=/tmp/instance-status-$$
while true ; do
  aws ec2 describe-instance-status \
  --instance-ids ${FGW_INST_ID} |tee ${tmpfile}
  inst_stat=$(cat $tmpfile |jq -r '.InstanceStatuses[0].InstanceStatus.Status')
  sys_stat=$(cat $tmpfile |jq -r '.InstanceStatuses[0].SystemStatus.Status')
  if [[ ${inst_stat} == "ok" &&  ${sys_stat} == "ok" ]]; then
    break
  else
    sleep 30
  fi
done

# get instance ip
INST_IP=$(aws ec2 describe-instances --instance-ids ${FGW_INST_ID} --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
INST_PRIV_IP=$(aws ec2 describe-instances --instance-ids ${FGW_INST_ID} --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

ACTIVATION_KEY=$(wget "${INST_IP}/?activationRegion=${AWS_REGION}" 2>&1 | \
grep -i location | \
grep -oE 'activationKey=[A-Z0-9-]+' | \
cut -f2 -d=)

FGW_ARN=$(aws storagegateway activate-gateway \
--gateway-name FGW-$RANDOM \
--gateway-timezone "GMT+8:00" \
--gateway-region ${AWS_REGION} \
--gateway-type FILE_S3 \
--activation-key  ${ACTIVATION_KEY} \
--query 'GatewayARN' --output text)

# aws storagegateway list-gateways --query 'Gateways[?GatewayARN==`'${FGW_ARN}'`].GatewayOperationalState' --output text

# HostEnvironment --> EC2
aws storagegateway list-gateways --query 'Gateways[?GatewayARN==`'${FGW_ARN}'`]' --output json

while true ; do
    aws storagegateway describe-gateway-information \
    --gateway-arn ${FGW_ARN}
    if [[ $? -eq 0 ]]; then
        break
    else
        sleep 30
    fi
done

DISK_IDS=$(aws storagegateway list-local-disks \
--gateway-arn ${FGW_ARN} \
--query 'Disks[*].DiskId' --output text)

aws storagegateway add-cache \
--gateway-arn ${FGW_ARN} \
--disk-ids ${DISK_IDS}

```

## create-nfs-share-
- create iam role
```sh
account_id=$(aws sts get-caller-identity --query "Account" --output text)
fgw_role_name=StorageGatewayBucketAccessRole-$RANDOM.$RANDOM
aws iam create-role --role-name ${fgw_role_name} --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"storagegateway.amazonaws.com"},"Action":"sts:AssumeRole","Condition":{"StringEquals":{"aws:SourceAccount":"'"${account_id}"'","aws:SourceArn":"'"${FGW_ARN}"'"}}}]}'
envsubst > ${fgw_role_name}-policy.yaml <<-EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:GetAccelerateConfiguration",
                "s3:GetBucketLocation",
                "s3:GetBucketVersioning",
                "s3:ListBucket",
                "s3:ListBucketVersions",
                "s3:ListBucketMultipartUploads"
            ],
            "Resource": "arn:aws:s3:::${BUCKET_NAME}",
            "Effect": "Allow"
        },
        {
            "Action": [
                "s3:AbortMultipartUpload",
                "s3:DeleteObject",
                "s3:DeleteObjectVersion",
                "s3:GetObject",
                "s3:GetObjectAcl",
                "s3:GetObjectVersion",
                "s3:ListMultipartUploadParts",
                "s3:PutObject",
                "s3:PutObjectAcl"
            ],
            "Resource": "arn:aws:s3:::${BUCKET_NAME}/*",
            "Effect": "Allow"
        }
    ]
}
EOF
aws iam put-role-policy --role-name ${fgw_role_name} --policy-name ${fgw_role_name}-policy --policy-document "file://./${fgw_role_name}-policy.yaml"
fgw_role_arn=$(aws iam get-role --role-name ${fgw_role_name} --query 'Role.Arn' --output text)

```

- create file share
```sh
# ensure client list is correct
client_token=$(echo $RANDOM |md5sum |tr -d ' -')
aws storagegateway create-nfs-file-share \
--client-token ${client_token} \
--gateway-arn ${FGW_ARN} \
--role ${fgw_role_arn} \
--location-arn "arn:aws:s3:::${BUCKET_NAME}/${PREFIX_NAME}/" \
--file-share-name ${BUCKET_NAME}-${PREFIX_NAME} \
--client-list "172.31.0.0/16" \
--cache-attributes "CacheStaleTimeoutInSeconds=300" \
--squash NoSquash \
--bucket-region ${AWS_REGION}

fs_arn=$(aws storagegateway list-file-shares --gateway-arn ${FGW_ARN} \
--query 'FileShareInfoList[0].FileShareARN' \
--output text)

echo "On Linux:"
echo "mount -t nfs -o nolock,hard ${INST_PRIV_IP}:/${BUCKET_NAME}-${PREFIX_NAME} /mnt_point "

```

## more

- https://aws.amazon.com/blogs/storage/mounting-amazon-s3-to-an-amazon-ec2-instance-using-a-private-connection-to-s3-file-gateway/
- using s3 gateway endpoint to enhance security of data transferring
    - https://aws.amazon.com/blogs/architecture/connect-amazon-s3-file-gateway-using-aws-privatelink-for-amazon-s3/





