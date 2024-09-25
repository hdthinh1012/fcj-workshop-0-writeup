---
title : "AWS Service Preparation"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 2. </b> "
---

Steps in this section contains 2 parts:
- Part 1: create neccessary AWS instance
  - Create IAM user
  - Create Security Group policy for an EC2
  - Create an AWS S3 Bucket 
  - Create an EC2 instance
- Part 2: setup EC2 instance packages
  - Install NVM, NodeJS
  - Install FFMPEG
  - Install s3fs, mounted AWS S3 Bucket to a local File Path

### Content
  - [Create AWS IAM user, EC2 instance, S3 bucket](2.1-set-up-aws/)
  - [Setup EC2 instance](2.2-set-up-ec2/)