---
title : "Setup IAM user, EC2, S3"
date :  "`r Sys.Date()`" 
weight : 1
chapter : false
pre : " <b> 2.2 </b> "
---

## Create IAM User
Login into your root account or an IAM account with full administrative right, then search `IAM` dashboard.
![iam-dashboard](/images/2-Aws-Service/2.1-set-up-aws/iam-dashboard.png)
Click the `users`, then click `Create user` button 
![iam-users](/images/2-Aws-Service/2.1-set-up-aws/iam-users.png)
Choose `I want to create an IAM user`
![create-iam-1](/images/2-Aws-Service/2.1-set-up-aws/create-iam-1.png)
Attach 2 permssion to the account: `AmazonEC2FullAccess` and `AmazonS3FullAccess`
![create-iam-permission-1](/images/2-Aws-Service/2.1-set-up-aws/create-iam-permission-1.png)
![create-iam-permission-2](/images/2-Aws-Service/2.1-set-up-aws/create-iam-permission-2.png)
Review account info before creation
![create-iam-review](/images/2-Aws-Service/2.1-set-up-aws/create-iam-review.png)
Save your password for later use
![create-iam-password](/images/2-Aws-Service/2.1-set-up-aws/create-iam-password.png)
Generate access key for later EC2 SSH connection. Open the newly create account info
![create-iam-ac-1](/images/2-Aws-Service/2.1-set-up-aws/create-iam-ac-1.png)
![create-iam-ac-2](/images/2-Aws-Service/2.1-set-up-aws/create-iam-ac-2.png)
![create-iam-ac-3](/images/2-Aws-Service/2.1-set-up-aws/create-iam-ac-3.png)
Download .csv and save for later use
![create-iam-ac-4](/images/2-Aws-Service/2.1-set-up-aws/create-iam-ac-4.png)

Then login to your newly create IAM account.
![iam-user-login](/images/2-Aws-Service/2.1-set-up-aws/iam-user-login.png)
![iam-user-logged-in](/images/2-Aws-Service/2.1-set-up-aws/iam-user-logged-in.png)

## Create S3 Bucket
Open the S3 panel, click the `Create bucket` button.
![s3-panel](/images/2-Aws-Service/2.1-set-up-aws/s3-panel.png)
Type the bucket name and choose `General Purpose` bucket option, keep the ACLs option as default.
![create-s3-1](/images/2-Aws-Service/2.1-set-up-aws/create-s3-1.png)
Uncheck the `Block all public access` checkbox and check `I acknowledge the current settings might result in this bucket and the objects within becoming public` in the warning box.
![create-s3-2](/images/2-Aws-Service/2.1-set-up-aws/create-s3-2.png)
Keep all other options as default and click `Create bucket`.
![create-s3-3](/images/2-Aws-Service/2.1-set-up-aws/create-s3-3.png)
![create-s3-4](/images/2-Aws-Service/2.1-set-up-aws/create-s3-4.png)
![create-s3-done](/images/2-Aws-Service/2.1-set-up-aws/create-s3-done.png)

## Create security group
Open the VPC dashboard, click the `Security Groups` tab on the left navigation bar.
![vpc-dashboard](/images/2-Aws-Service/2.1-set-up-aws/vpc-dashboard.png)
Click `Create security group`.
![sg-panel](/images/2-Aws-Service/2.1-set-up-aws/sg-panel.png)
Type the security group name and description, choose the VPC network that this security group applies to.
![create-sg-1](/images/2-Aws-Service/2.1-set-up-aws/create-sg-1.png)
Setting inbound rules and outbound rules as images below
- Inbound rules consist of port 22 for SSH, port 443 for HTTPS, port 80 for HTTP. In this project, the server will be configured to listened at port 8080, so another allow rule for the port is added.
- Outbound rules is keep as default.
![create-sg-2](/images/2-Aws-Service/2.1-set-up-aws/create-sg-2.png)
![create-sg-3](/images/2-Aws-Service/2.1-set-up-aws/create-sg-3.png)
Click `Create security group`
![create-sg-4](/images/2-Aws-Service/2.1-set-up-aws/create-sg-4.png)

## Create EC2 Instance
Open the EC2 panels, click `Create instance` button.
![ec2-panel](/images/2-Aws-Service/2.1-set-up-aws/ec2-panel.png)
Type the instance name, choose Amazon Linux 2023 as Operating System and choose the Instance Type.
{{% notice note %}}
This project uses FFMPEG to generate HLS Master Playlist to provide multiple resolution, bitrate options for Adaptive Streaming. Since the command for HLS Master Playlist generation consumes large amount of RAM, an instance type with at least 4GB of RAM is recommended. Otherwise, please follow swap memory add-in step in the later section to avoid the ffmpeg command getting SIGKILL signal for RAM overflow.
{{% /notice %}}
![create-ec2-1](/images/2-Aws-Service/2.1-set-up-aws/create-ec2-1.png)
Click `Create keypair` and download for later SSH login session.
![create-ec2-2](/images/2-Aws-Service/2.1-set-up-aws/create-ec2-2.png)
Choose the newly created keypair for authentication.
![create-ec2-3](/images/2-Aws-Service/2.1-set-up-aws/create-ec2-3.png)
Choose the security group created in the previous section.
![create-ec2-4](/images/2-Aws-Service/2.1-set-up-aws/create-ec2-4.png)
Configure the storage, 4GB will be used as swap memory in later section and multiple package will be installed: FFMPEG, s3fs, fuse, nvm, node. So 24GB of storage is configured for this instance.
Recheck the instance info, then click `Launch instance`.
![create-ec2-5](/images/2-Aws-Service/2.1-set-up-aws/create-ec2-5.png)
![create-ec2-done](/images/2-Aws-Service/2.1-set-up-aws/create-ec2-done.png)
