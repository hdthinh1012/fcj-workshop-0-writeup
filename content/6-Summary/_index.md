---
title : "Summary"
date :  "`r Sys.Date()`" 
weight : 6
chapter : false
pre : " <b> 6. </b> "
---

In summary, this workshop project contains 2 parts:
- Part 1: Implement video upload and stream workflow web application for local file system
  - Implement large video upload mechanism by splitting into chunks
  - Generate HLS Master playlist from original upload.
  - Streaming video with front-end React App
- Part 2: Migrate to deploy on EC2 instance and store in AWS S3 Bucket.
  - Implementing custom engine for Multer to support direct uploading to AWS S3 Bucket.
  - Implement custom NodeJS Writable Stream and Readable Stream with Multipart Uploading/Downloading command from the AWS S3 SDK v3, replacing fs.WriteStream and fs.ReadStream.
  - Mounting AWS S3 Bucket into local file system using s3fs package, executing ffmpeg Master Playist generation with mounted S3 files with the same command as when working with local file.

## Limitation and improvement suggestion
The project workflow is limited in 2 aspect, network upload speed and FFMPEG command execution speed, both are too slow. Some suggestion includes:

- Run EC2 instance in nearer region (Singapore).
- Using a bigger instance type to remove the need of swap usage that cause significant slow down in process time.
- Split the server into 2 different instance, one handle upload, one handle video streaming.

These are some improvement suggestion I can think of. In the future, I will learn some method for performance testing then apply for this project to measure how much an improvement these change brings and update this posts with some performance measurements.