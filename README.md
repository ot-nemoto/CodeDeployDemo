# CodeDeployGitHubDemo

### [AWS CodeDeploy のサービスロールを作成する](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/getting-started-create-service-role.html)

```sh
cat << EOT > CodeDeployDemo-Trust.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "codedeploy.amazonaws.com"
                ]
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOT

aws iam create-role \
  --role-name CodeDeployServiceRole \
  --assume-role-policy-document file://CodeDeployDemo-Trust.json

aws iam attach-role-policy \
  --role-name CodeDeployServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSCodeDeployRole
```

### [Amazon EC2 インスタンス用の IAM インスタンスプロファイルを作成する](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/getting-started-create-iam-instance-profile.html)

```sh
cat << EOT > CodeDeployDemo-EC2-Trust.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "",
            "Effect": "Allow",
            "Principal": {
                "Service": "ec2.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOT

cat << EOT > CodeDeployDemo-EC2-Permissions.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
EOT

aws iam create-role \
  --role-name CodeDeployDemo-EC2-Instance-Profile \
  --assume-role-policy-document file://CodeDeployDemo-EC2-Trust.json

aws iam put-role-policy \
  --role-name CodeDeployDemo-EC2-Instance-Profile \
  --policy-name CodeDeployDemo-EC2-Permissions \
  --policy-document file://CodeDeployDemo-EC2-Permissions.json

aws iam create-instance-profile \
  --instance-profile-name CodeDeployDemo-EC2-Instance-Profile
aws iam add-role-to-instance-profile \
  --instance-profile-name CodeDeployDemo-EC2-Instance-Profile \
  --role-name CodeDeployDemo-EC2-Instance-Profile
```

### [AWS CodeDeploy 用の Amazon EC2 インスタンスの作成](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/instances-ec2-create.html)

Create Security Group

```sh
aws ec2 create-security-group \
  --group-name CodeDeployDemo-Windows-Security-Group \
  --description "For launching Windows Server images for use with AWS CodeDeploy"

aws ec2 authorize-security-group-ingress \
  --group-name CodeDeployDemo-Windows-Security-Group \
  --to-port 80 \
  --ip-protocol tcp \
  --cidr-ip 0.0.0.0/0 \
  --from-port 80
```

Create EC2 Instances

```sh
cat << EOT > instance-setup.sh
#!/bin/bash
yum -y update
yum install -y ruby
cd /home/ec2-user
curl -O https://aws-codedeploy-ap-northeast-1.s3.amazonaws.com/latest/install
chmod +x ./install
./install auto
EOT

# ami-02e680c4540db351e: Amazon Linux 2 AMI (HVM), SSD Volume Type
aws ec2 run-instances \
  --image-id ami-02e680c4540db351e \
  --user-data file://instance-setup.sh \
  --count 1 \
  --instance-type t2.micro \
  --iam-instance-profile Name=CodeDeployDemo-EC2-Instance-Profile \
  --security-groups CodeDeployDemo-Windows-Security-Group \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=CodeDeployDemo}]' 'ResourceType=volume,Tags=[{Key=Name,Value=CodeDeployDemo}]'
```

### [アプリケーションおよびデプロイグループの作成](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/tutorials-github-create-application.html)

```
aws deploy create-application \
  --application-name CodeDeployGitHubDemo-App

aws deploy create-deployment-group \
  --application-name CodeDeployGitHubDemo-App \
  --ec2-tag-filters Key=Name,Type=KEY_AND_VALUE,Value=CodeDeployDemo \
  --deployment-group-name CodeDeployGitHubDemo-DepGrp \
  --service-role-arn $(aws iam get-role --role-name CodeDeployServiceRole --query 'Role.Arn' --output text)
```

### [アプリケーションをインスタンスにデプロイする](https://docs.aws.amazon.com/ja_jp/codedeploy/latest/userguide/tutorials-github-deploy-application.html)

```sh
aws deploy create-deployment \
  --application-name CodeDeployGitHubDemo-App \
  --deployment-config-name CodeDeployDefault.OneAtATime \
  --deployment-group-name CodeDeployGitHubDemo-DepGrp \
  --github-location repository=ot-nemoto/CodeDeployGitHubDemo,commitId=e22cc3c5494cfacd9b95c6788259903350ebe0b1
```

### アプリケーションがデプロイされていることを確認

以下で出力されるパブリックドメイン名にブラウザでアクセス

```sh
aws ec2 describe-instances \
  --filters Name=tag:Name,Values=CodeDeployDemo Name=instance-state-name,Values=running \
  --query 'Reservations[].Instances[].PublicDnsName' \
  --output text
```
