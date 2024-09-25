---
title : "Deploy back-end source code on EC2 instance"
date :  "`r Sys.Date()`" 
weight : 1
chapter : false
pre : " <b> 3.1 </b> "
---

## Deploy Back-end source code
Clone the back-end source code from [https://github.com/hdthinh1012/aws-workshop-0-hls-streaming](https://github.com/hdthinh1012/aws-workshop-0-hls-streaming "Backend Github Link").
Then cd into the project folder and create environment file `.env` and add the following text.

```txt
PORT=8080
SERVER_URL=http://<your-ec2-public-ip>:8080 // For static file serve
AWS_ACCESS_KEY_ID=<your_iam_access_key_id>
AWS_SECRET_ACCESS_KEY=<your_iam_secret_access_key>
BUCKET_NAME=<your-bucket-name>

IS_AWS_S3=1 // or 0 for local file system
AWS_S3_BUCKET_PATH=<your-mounted-s3fs-location>
```

![clone-backend-source](/images/3-Project-source-code/3.1-deploy-back-end/clone-backend-source.png)
![edit-env-file](/images/3-Project-source-code/3.1-deploy-back-end/edit-env-file.png)

Replace `<your_iam_access_key_id>`, `<your_iam_secret_access_key>` with your IAM user corresponding access key id and secret access key, `<your-bucket-name>` with your newly created S3 bucket's name and `<your-mounted-s3fs-location>` with the local file system path on EC2 instance that is used for S3FS-FUSE mount system from the section 2.2.

Then run `npm install` to install all dependencies.
Run `npm run build` for Webpack bundler and `npm run start`. The back-end web is serving at the port 8080 (or any port configured in .env).
![back-end-start](/images/3-Project-source-code/3.1-deploy-back-end/back-end-start.png)
