---
title : "Architecture"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 1.2 </b> "
---

This workshop web applciation is implemented using basic architecture, with NodeJS backend application deployed in a single public EC2 instance, interating with AWS S3 Bucket through SDK and S3FS mounted system.  
User clients run a React app to send request directly to public EC2 instance through internet.

![architecture](/images/1-Introduction/architecture.jpg)