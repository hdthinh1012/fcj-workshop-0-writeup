---
title : "AWS Services in this workshop"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 1.1.2 </b> "
---

## AWS Elasic Compute Service (EC2)
A High-level general purpose virtual private server provided by AWS, providing multiple features such as:
- Amazon Machine Images (AMIs): provide multiple image packages for variety of Operating System, developement environment, package components.
- Instance types: multiple configurations of CPU cores, memory, disk size, network capacity and hardware graphic acclerations
- Amazon EBS volumes: persistent storage volumes
- Instance store volumes:  storage volumes for temporary data that is deleted when you stop, hibernate, or terminate your instance.
- Key pairs: secure login information for your instances. AWS stores the public key and you store the private key in a secure place.

## AWS VPC Security Group
Act as virtual firewall, setting allow/deny rules for inbound/outbound requests based on protocol, port, destination IP ranges, source IP ranges, etc.

## AWS Simple Storage Service (S3)
A cloud file storage system provided by AWS for general purpose file storage. AWS S3 is a flat file system, meaning all the files stored in the same level, but abstracted as folder-like structure for easier management.  
For example, a bucket display as below in the AWS S3 management console:
    
```
uploads/
    chunks/
        sintel_trailer.mp4.part_0
        sintel_trailer.mp4.part_1
    videos/
        sintel_trailer.mp4
```

Is stored physically as:
```
- uploads/chunks/sintel_trailer.mp4.part_0
- uploads/chunks/sintel_trailer.mp4.part_1
- uploads/videos/sintel_trailer.mp4
```

## AWS SDK v3 for S3
This workshop uses AWS SDK v3 to upload and download large files from AWS S3 Bucket, the code example for be founded in "Upload or download large files" section in this [Developer Guide](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/javascript_s3_code_examples.html "Amazon S3 examples using SDK for JavaScript (v3)").

Commands for uploading files:  
- `CreateMultipartUploadCommand`: Initialize a multipart upload process, return an `uploadId` value for subsequent uses.  
- `UploadPartCommand`: The actual upload command, called multiple times to upload multiple chunks to destination defined by `CreateMultipartUploadCommand`  
- `CompleteMultipartUploadCommand`: Finalize the multipart upload process.  
- `AbortMultipartUploadCommand`: Deleted the uploaded part and abort multipart upload process (use in case of error)  

Commands for downloading files:
`GetObjectCommand`: Get all or part of data bytes from a AWS S3 file

AWS S3 Client SDK is provided throught an NPM library package [here](https://www.npmjs.com/package/@aws-sdk/client-s3 "aws-sdk/client-s3"). To send a request, you:
- Initiate client with configuration (e.g. credentials, region):
```js
import { S3Client } from "@aws-sdk/client-s3";
const client = new S3Client({ 
    region: "REGION",
    credentials: {
        accessKeyId: process.env.AWS_ACCESS_KEY_ID,
        secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY,
    },
});
```

- Initiate command with input parameter
```js
import { ListBucketsCommand  } from "@aws-sdk/client-s3";
const params = {
  /** input parameters */
};
const command = new ListBucketsCommand(params);
```

- Executing command with send method:
```js
// async/await.
try {
  const data = await client.send(command);
  // process data.
} catch (error) {
  // error handling.
} finally {
  // finally.
}
```