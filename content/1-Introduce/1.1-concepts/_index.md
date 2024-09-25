---
title : "Related concepts"
date :  "`r Sys.Date()`" 
weight : 1 
chapter : false
pre : " <b> 1.1 </b> "
---
## HTTP Live Streaming
A protocol of adaptive streaming proposed by Apple, providing ability to stream videos with different bitrate, automatically or manually chosen by the client.

HLS breaks down videos into smaller chunks and delivers them using HTTP protocol. The client devices implemented to comply to HLS protocol downloads the video chunk and serve to users. HLS protocol also provided options to generate and deliver chunk into different video encodings, bitrate, resolution, audio quality.  

## AWS Elasic Compute Service (EC2)
A High-level general purpose virtual private server provided by AWS, providing multiple features such as:
- Amazon Machine Images (AMIs):  
Provide multiple image packages for variety of Operating System, developement environment, package components.
- Instance types:  
Multiple configurations of CPU cores, memory, disk size, network capacity and hardware graphic acclerations
- Amazon EBS volumes:  
Persistent storage volumes
- Instance store volumes:   
Storage volumes for temporary data that is deleted when you stop, hibernate, or terminate your instance.
- Key pairs:  
Secure login information for your instances. AWS stores the public key and you store the private key in a secure place.

## AWS VPC Security Group
Act as virtual firewall, setting allow/deny rules for inbound/outbound connection based on protocol, port, destination IP ranges, source IP ranges, etc.

## AWS Simple Storage Service (S3)
A cloud file storage system provided by AWS for general purpose file storage.  
AWS S3 is implemented as flat file system, meaning all the files stored in the same level, but is abstract as folder-like structure for easier view and management.  
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

## NodeJS
Stream is a method of handling data method that worked best for large file size transmission. Instead of reading or writing all data at once, NodeJS stream read or write data in a chunk-by-chunk manner, and into its own implemented custom buffer. NodeJS stream must release memory from it buffer by pushing the data it has read / sending the data it read to final destination when its buffer reach full capacity (determined by `highWaterMark` properties of a NodeJS stream instance) before continuing reading or writing data.

### stream:Writable, stream:Readable, Multer custom engine
The application ultilizes stream:Writable, stream:Readable from "node:stream" library to implement custom read and write stream to S3 buckets by using AWS v3 SDK. Besides, a Multer custom engine is also implemented to directly upload file to AWS S3 from client without writing to server disk, saving uploading times.

You can learn more about implement custom stream in this brilliant guide [Node.js: How to implement own streams?](https://medium.com/@vaibhavmoradiya/how-to-implement-own-streams-in-node-js-edd9ab54a59b) and implement custom multer engine in [Make your own Storage Engine for Multer in TypeScript](https://javascript.plainenglish.io/custom-storage-engine-for-multer-in-typescript-613ebd35d61e)

#### stream.Writable
To implement a custom write stream, we extends `Writable` class from `node:stream` library. The important method to override are:
- `construct(opt: {}): void`: constructor for passing configuration argument for the stream
- `_construct():void`: called right after the constructor to intialize the data, like opening local file or setting AWS sdk command
- `_write(chunk: any, encoding: BufferEncoding, callback: (error?: Error | null) => void): void`: called any time user called `writeStreamObj.write(<some-bytes...>)` or the stream right before it in a NodeJS pipeline call `push(chunk: any, encoding: Encoding): boolean` into it.
- `_final(callback: (err: Error|undefined) => void): void`: call after user called `writeStreamObj.end(<some-bytes...>)` or the stream right before it in a NodeJS pipeline emit the event 'end'.
- `_destroy(err: Error | undefined, callback: (err: Error|undefined) => void)`: called after the `_final` function, cleanup unused resources.

An example code of reimplement fs.WriteStream by extending the stream.Writable class
```js
const { Writable } = require("node:stream");
const fs = require("fs");

class FileWriteStream extends Writable {
  constructor({ highWaterMark, fileName }) {
    super({ highWaterMark });

    this.fileName = fileName;
    this.fd = null;
    this.chunks = [];
    this.chunkSize = 0;
    this.writesCount = 0;
  }

  // This method is run after the constructor has been called and it will put off all calling the
  // other methods until we call the callback function
  // we can use this method to initialize the data like opening a new file
  _construct(callback) {
    fs.open(this.fileName, "w", (err, fd) => {
      if (err) {
        // if we call the callback with an arguments means that we have an error
        // and we should not proceed
        callback(err);
      } else {
        this.fd = fd;
        // no arguments means it was successfull
        callback();
      }
    });
  }

  _write(chunk, encoding, callback) {
    this.chunks.push(chunk);
    this.chunkSize += chunk.length;
    if (this.chunkSize > this.highWaterMark) {
      fs.write(this.fd, Buffer.concat(this.chunks), (err) => {
        if (err) return callback(err);
        this.chunks = [];
        this.chunkSize = 0;
        ++this.writesCount;
        callback();
      });
    } else {
      // when we are done, we should call the callback function
      callback();
    }
  }

  // this will run after the our stream has finished
  _final(callback) {
    fs.write(this.fd, Buffer.concat(this.chunks), (err) => {
      if (err) {
        return callback(err);
      }

      this.chunks = [];
      callback();
    });
  }

  // this method is called when we are done with the final method
  _destroy(error, callback) {
    console.log("Write Count:", this.writesCount);
    if (this.fd) {
      fs.close(this.fd, (err) => {
        callback(err | error);
      });
    } else {
      callback(error);
    }
  }
}

const stream = new FileWriteStream({
  highWaterMark: 1800,
  fileName: "text.txt",
});

stream.write(Buffer.from("abc"));
stream.end(Buffer.from("xyz"));

stream.on("finish", () => {
  console.log("stream was finished");
});
```
#### stream.Readable
To implement a custom read stream, we extends `Readable` class from `node:stream` library. The important method to override are:
- `construct(opt: {}): void`: constructor for passing configuration argument for the stream
- `_construct():void`: called right after the constructor to intialize the data, like opening local file or setting AWS sdk command
- `_read(size: number): void`: responsible to read data from input and push to following stream, the method body must call `this.push(chunk: any): void`. The `_read` method is called the first time when the event listener to 'data' is added to the read stream or the read stream is piped into another write stream. 
  - if `this.push(chunk)` called with chunk is not nulled, the `_read` method will be called again and again until `this.push(chunk)` push a null data
- `_destroy(err: Error | undefined, callback: (err: Error|undefined) => void)`: called after the last `_read` called, clean unused resource.
An example code of reimplement fs.ReadStream by extending the stream.Readable class
```js
const { Readable } = require("node:stream");
const fs = require("fs");

class FileReadStream extends Readable {
  constructor({ highWaterMark, fileName }) {
    super({ highWaterMark });
    this.fileName = fileName;
    this.fd = null;
  }

  _construct(callback) {
    fs.open(this.fileName, "r", (err, fd) => {
      if (err) return callback(err);
      this.fd = fd;
      callback();
    });
  }

  _read(size) {
    const buff = Buffer.alloc(size);
    fs.read(this.fd, buff, 0, size, null, (err, byteRead) => {
      if (err) this.destroy(err);
      // null means that we are at end of the stream
      this.push(byteRead > 0 ? buff.subarray(0, byteRead) : null);
    });
  }

  _destroy(error, callback) {
    if (this.fd) {
      fs.close(this.fd, (err) => callback(err | error));
    } else {
      callback(error);
    }
  }
}

const stream = new FileReadStream({ fileName: "text.txt" });

stream.on("data", (chunk) => {
  console.log(chunk.toString("utf-8"));
});

stream.on("end", () => {
  console.log("Stream read complete");
});
```

#### Multer custom engine
`multer` is a popular library package used for parsing multipart POST request, especially for file uploading. `multer` provided a `multer.diskStorage()` for easy file upload parsing and temporarily storing before processing in request handler.

To implement Multer custom engine for uploading file to AWS S3 Bucket, we will extend `multer.StorageEngine` class in `multer` package. The important method to override are:
- `_handleFile = (req: Request, file: Express.Multer.File, cb: (error?: any, info?: CustomFileResult) => void): void `: responsible to write uploaded file to the desired destination, whether it is a local file storage or cloud bucket  
- `_removeFile = (_req: Request, file: Express.Multer.File & { name: string }, cb: (error: Error | null) => void): void`: remove the uploaded file in case an error occured.

An example code of implement multer custom engine to upload to google cloud storage by extending the `multer.StorageEngine` class (for referencing only, we will implement a AWS S3 bucket one in later section :v)
```js
import { Bucket, CreateWriteStreamOptions } from "@google-cloud/storage";
import { Request } from "express";
import multer from "multer";
import path from "path";

type nameFnType = (req: Request, file: Express.Multer.File) => string;

type Options = {
  bucket: Bucket;
  options?: CreateWriteStreamOptions;
  nameFn?: nameFnType;
  validator?: validatorFn;
};

const defaultNameFn: nameFnType = (
  _req: Request,
  file: Express.Multer.File
) => {
  const fileExt = path.extname(file.originalname);

  return `${file.fieldname}_${Date.now()}${fileExt}`;
};

interface CustomFileResult extends Partial<Express.Multer.File> {
  name: string;
}

class CustomStorageEngine implements multer.StorageEngine {
  private bucket: Bucket;
  private options?: CreateWriteStreamOptions;
  private nameFn: nameFnType;

  constructor(opts: Options) {
    this.bucket = opts.bucket;
    this.options = opts.options || undefined;
    this.nameFn = opts.nameFn || defaultNameFn;
  }

  _handleFile = (
    req: Request,
    file: Express.Multer.File,
    cb: (error?: any, info?: CustomFileResult) => void
  ): void => {
    if (!this.bucket) {
      cb(new Error("bucket is a required field."));
      return;
    }

    const fileName = this.nameFn(req, file);

    const storageFile = this.bucket.file(fileName);
    const fileWriteStream = storageFile.createWriteStream(this.options);
    const fileReadStream = file.stream;

    fileReadStream
      .pipe(fileWriteStream)
      .on("error", (err) => {
        fileWriteStream.end();
        storageFile.delete({ ignoreNotFound: true });
        cb(err);
      })
      .on("finish", () => {
        cb(null, { name: fileName });
      });
  };

  _removeFile = (
    _req: Request,
    file: Express.Multer.File & { name: string },
    cb: (error: Error | null) => void
  ): void => {
    this.bucket.file(file.name).delete({ ignoreNotFound: true });
    cb(null);
  };
}

export default (opts: Options) => {
  return new CustomStorageEngine(opts);
};
```