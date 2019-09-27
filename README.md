# Reference
Installs/registers SSM and Cloudwatch for CentOS 7 and Ubuntu 1804

# Variables
Must be supplied to complete installation/registration
```
cloudwatch_key_id
cloudwatch_access_key
ssm_activation_code
ssm_activation_id
ssm_region
```

# IAM
```
# ec2 cloudwatch access - creates an iam ec2 service role and attaches the cloudwatch policy

# ec2 role-policy-document
EC2_CLOUDWATCH_ROLE_POLICY=$(mktemp)
tee $EC2_CLOUDWATCH_ROLE_POLICY << EOM
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"Service": "ec2.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }
}
EOM

# create role
aws iam create-role --role-name CloudWatchAgentServerRole --assume-role-policy-document file://$EC2_CLOUDWATCH_ROLE_POLICY

# attach cloudwatch agent to role
aws iam attach-role-policy --role-name CloudWatchAgentServerRole --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# create instance profile
aws iam create-instance-profile --instance-profile-name CloudWatchAgentServerProfile

# add role to profile (alternatively, add the role to a pre-existing instance profile)
aws iam add-role-to-instance-profile --instance-profile-name CloudWatchAgentServerProfile --role-name CloudWatchAgentServerRole

# associate the role to instance(s) - will not preempt an existing instance profile association (aws ec2 replace-iam-instance-profile-association would)
aws ec2 associate-iam-instance-profile --instance-id i-0d9ca6934def7ad4c --iam-instance-profile Name="CloudWatchAgentServerProfile"

# remove temporary role policy file
rm $EC2_CLOUDWATCH_ROLE_POLICY

###
# non ec2 ssm access - creates a policy to allow non-ec2 machines access to Systems Manager
# ssm service policy
SSM_SERVICE_ROLE_POLICY=$(mktemp)
tee $SSM_SERVICE_ROLE_POLICY << EOM
{
  "Version": "2012-10-17",
  "Statement": {
    "Effect": "Allow",
    "Principal": {"Service": "ssm.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }
}
EOM

# create role
aws iam create-role --role-name SSMServiceRole --assume-role-policy-document file://$SSM_SERVICE_ROLE_POLICY

# attach policy allowing managed instances (off premise) to use Systems Manager
aws iam attach-role-policy --role-name SSMServiceRole --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

# remove temporary service policy file
rm $SSM_SERVICE_ROLE_POLICY

###
# non ec2 cloudwatch access - creates an iam service account and attaches the cloudwatch policy

# the service account name
NON_EC2_CLOUDWATCH='non_ec2_cloudwatch'

# create user
aws iam create-user --user-name $NON_EC2_CLOUDWATCH

# attach policy to user
aws iam attach-user-policy --user-name $NON_EC2_CLOUDWATCH --policy arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# create user's key and secret
NON_EC2_CLOUDWATCH_SECRETKEY=$(aws iam create-access-key --user-name $NON_EC2_CLOUDWATCH --output=text --query "AccessKey.SecretAccessKey")
NON_EC2_CLOUDWATCH_ACCESSKEY=$(aws iam list-access-keys --user-name $NON_EC2_CLOUDWATCH --output=text --query "AccessKeyMetadata[0].AccessKeyId")

# echo IDs
echo "$NON_EC2_CLOUDWATCH credentials
access key: $NON_EC2_CLOUDWATCH_ACCESSKEY
secret key: $NON_EC2_CLOUDWATCH_SECRETKEY"
```
