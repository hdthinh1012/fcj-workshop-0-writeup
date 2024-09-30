---
title : "Table of contents"
date :  "`r Sys.Date()`" 
weight : 3
chapter : false
pre : " <b> 1.3 </b> "
---


## Steps presented in this Workshop
- Prepare the neccessary AWS service: IAM user, Security Group policy, EC2 server, S3 bucket
- Setting up EC2 server: mounted bucket using s3fs-fuse, installing Node, FFMPEG for HLS Master Playlist generation, adding swap memory to handle ffmpeg command (ffmpeg will crash if used on a micro instance without swap memory).
- Deploying back-end server to EC2 instance, setting .env file
- Cloning front-end source to client device, running react-vite app
- Uploading video, then stream
- Cleaning resouce
- Summary