---
title : "Setup EC2 instance"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 2.2 </b> "
---
Login into newly created EC2 instance with the correct .pem files that contain SSH keypair for the instance.
```terminal
ssh ec2-user@<ec2-public-ip> -i <your-pem-file-path>
```
![ec2-login](/images/2-Aws-Service/2.2-set-up-ec2/ec2-login.png)

## Install FUSE, F3FS and mounted S3 bucket to a local filepath (Amazon Linux 2023)
If you use Amazon Linux 2023 OS for the EC2 instance, continue with the following guide to install fuse and s3fs. For other operating system, please refer to s3fs README [here](https://github.com/s3fs-fuse/s3fs-fuse?tab=readme-ov-file "s3fs Github").
Install fuse with the below command:
```terminal
sudo yum install automake fuse fuse-devel gcc-c++ git libcurl-devel libxml2-devel make openssl-devel -y
```
![install-fuse-2](/images/2-Aws-Service/2.2-set-up-ec2/install-fuse-2.png)
Clone the `s3fs-fuse` repository and run the following command
```terminal
git clone https://github.com/s3fs-fuse/s3fs-fuse.git

cd  s3fs-fuse
./autogen.sh 
./configure --prefix=/usr --with-openssl
make
sudo make install
```
![clone-s3fs-fuse](/images/2-Aws-Service/2.2-set-up-ec2/clone-s3fs-fuse.png)
After s3fs and fuse installation, create `~/.passwd-s3fs` and add IAM user's AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY for s3fs credential. AWS policy enforce permission 600 on `.passwd-s3fs` to avoid accidentally unpermitted access.
```terminal
touch ~/.passwd-s3fs
echo <your_iam_access_key_id>:<your_iam_secret_access_key> > ~/.passwd-s3fs
chmod 0600 ~/.passwd-s3fs
```
Mount AWS S3 bucket to EC2 local filesystem path using command:
```terminal
s3fs <your-bucket-name> <your-mounted-s3fs-location>
```
![check-s3fs-1](/images/2-Aws-Service/2.2-set-up-ec2/check-s3fs-1.png)
Checking if the mounted s3 file system worked.
![check-s3fs-2](/images/2-Aws-Service/2.2-set-up-ec2/check-s3fs-2.png)
## Install NVM, Node 18
Please refer NVM installation for your system in [here](https://github.com/nvm-sh/nvm "NVM Github Link"). The command snippet for installling NVM and Node 18
```terminal
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
nvm install 18
```
![install-nvm](/images/2-Aws-Service/2.2-set-up-ec2/install-nvm.png)
![install-node](/images/2-Aws-Service/2.2-set-up-ec2/install-node.png)
## Install FFMPEG (Amazon Linux 2023)
The below code snippet will present the steps to install FFMPEG on Amazon Linux 2023, please refer to the [FFMPEG Official Page](https://www.ffmpeg.org/download.html) for ffmpeg installation in your own system.
```terminal
sudo wget https://github.com/BtbN/FFmpeg-Builds/releases/download/latest/ffmpeg-master-latest-linux64-gpl.tar.xz
sudo tar -xf ffmpeg-master-latest-linux64-gpl.tar.xz
sudo mv ffmpeg-master-latest-linux64-gpl/ /usr/local/bin/ffmpeg/
sudo rm ffmpeg-master-latest-linux64-gpl.tar.xz
sudo chown -R ec2-user.ec2-user /usr/local/bin/ffmpeg/
sudo ln -s /usr/local/bin/ffmpeg/bin/ffmpeg /usr/bin/ffmpeg
sudo ln -s /usr/local/bin/ffmpeg/bin/ffprobe /usr/bin/ffprobe
```
## Add swap memory (Amazon Linux 2023)
The below code snippet contains commands to create swap memory section on Amazon Linux 2023, the original guide can be found [here](https://repost.aws/knowledge-center/ec2-memory-swap-file "EC2 memory swap guide").
```terminal
sudo dd if=/dev/zero of=/swapfile bs=128M count=32
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
sudo swapon -s
sudo echo "/swapfile swap swap defaults 0 0" >> /etc/fstab
```