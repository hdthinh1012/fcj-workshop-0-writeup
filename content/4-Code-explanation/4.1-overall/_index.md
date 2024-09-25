---
title : "Overall explanation"
date :  "`r Sys.Date()`" 
weight : 1
chapter : false
pre : " <b> 4.1 </b> "
---

## Video upload to video stream flow chart
![flow-diagram](/images/3-Project-source-code/3.3-code-explanation/flow-diagram.png)
The uploading and streaming video process revolves around 3 abstract directory: `upload/chunks` for uploading video split chunks, `upload/videos` stored the uploaded file after chunk merge, and `streams/` contains HLS master playlist generated from FFMPEG.
The upload to stream flow contains following steps:
- Front-end divides videos into chunks (chunk size are determined by server).
- Sending each chunk to server, server saved chunk into `upload/chunks` folder of the S3 bucket.
- Server read each chunk from s3 bucket and write continuosly into a complete final file in `upload/videos`.
- Server run FFMPEG to generate HLS master playlist.
- Front-end web application get the `master.m3u8` file and push to `hls.js` plugin to start adaptive streaming process.

## Some notable class
### File system path
The project was developed for local file system usage first, then migrate into AWS S3 bucket. Hence, some boilerplate class implementation is necessary for smooth transition between local file system and AWS S3 Bucket.
The file system path class responsible for generate file path whether it is relative or absolute.
![file-system-path](/images/3-Project-source-code/3.3-code-explanation/file-system-path.png)
### File system action
The file system action responsible for action such as: read from directory, create file, remove file, remove directory, create write stream, create read stream, pipe read and write stream together, etc.
![file-system-action](/images/3-Project-source-code/3.3-code-explanation/file-system-action.png)
### Multer engine
The local file system version use `multer.diskStorage` class to handle file upload parsing and storing, the AWS S3 version has to implement a custom engine using multipart upload commands from the [AWS S3 SDK](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpu-upload-object.html).
![multer](/images/3-Project-source-code/3.3-code-explanation/multer.png)

### File write stream and read stream
In the local file system version, the step of merging upload chunks into a complete upload file was handle by this section of the code
```js
const writeStream = fs.createWriteStream('<complete file path>');
for (let chunkIdx = 0; chunkIdx < chunkNums; chunkIdx += 1) {
    const readStream = fs.createReadStream('chunkIdx chunk path');
    await new Promise<void>((resolve, reject) => {
        readStream.pipe(writeStream, {end: false});
        readStream.on('end', () => resolve());
        readStream.on('error', () => reject());
    })
}
```
To easily switch back and forth between using local file system and AWS S3 bucket, a custom write stream and read stream were implemented, inherited from `Writable` and `Readable` class from `node:stream`.
{{% notice note %}}
Sidenote: The solution of using s3fs-fuse mounted storage was founded after I implement these custom stream. The stream classes work as expected so I keep them. For the later step of using FFMPEG to generate HLS playlist, I happily ultilized s3fs for an easier transition to AWS S3.
{{% /notice %}}
![stream](/images/3-Project-source-code/3.3-code-explanation/stream.png)

## API Routes
```
/api/video
    /stream
        /get-all: List all video avalable for streaming
    /upload
        /: api for uploading each chunk
        /clean-all: clear all uploaded file in the server
    /process
        /get-all: Get detail info of all videos (bitrate, encodings, resolution, etc.)

/static/hls: serving `.m3u8` file for HLS streaming
/static: serving raw uploaded video
```