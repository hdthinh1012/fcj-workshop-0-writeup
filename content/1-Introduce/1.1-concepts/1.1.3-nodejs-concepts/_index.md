---
title : "NodeJS Classes in this workshop"
date :  "`r Sys.Date()`" 
weight : 3
chapter : false
pre : " <b> 1.1.3 </b> "
---

## NodeJS Stream
Stream is a method of handling data method that worked best for large file size transmission. Instead of reading or writing data all at once, NodeJS streams read or write data chunk-by-chunk into its buffers.  
When the buffers reach full capacity (determined by `highWaterMark` properties of a NodeJS stream instance), NodeJS streams must release it data by pushing onto the next stream or write to a final file destination

### NodeJS bases classes (Writable, Readable)
The application ultilizes stream:Writable, stream:Readable from "node:stream" library to implement custom read and write stream to S3 buckets using AWS v3 SDK.

You can learn more about implement custom stream in this brilliant guide [Node.js: How to implement own streams?](https://medium.com/@vaibhavmoradiya/how-to-implement-own-streams-in-node-js-edd9ab54a59b)
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

###  Multer storage engine
`multer` is a popular library package used for parsing multipart POST request, especially for file uploading. `multer` provided a `multer.diskStorage()` for easy file upload parsing and temporarily storing before processing in request handler. In this workshop, a Multer custom engine is implemented for direct AWS S3 file upload without writing to server disk, saving uploading times. (You can learn about implementing custom multer engine in this [article](https://javascript.plainenglish.io/custom-storage-engine-for-multer-in-typescript-613ebd35d61e))

We will extend `multer.StorageEngine` class in `multer` package. The important method to override are:
- `_handleFile = (req: Request, file: Express.Multer.File, cb: (error?: any, info?: CustomFileResult) => void): void `: responsible to write uploaded file to the desired destination, whether it is a local file storage or cloud bucket  
- `_removeFile = (_req: Request, file: Express.Multer.File & { name: string }, cb: (error: Error | null) => void): void`: remove the uploaded file in case an error occured.

An example code of implement multer custom engine to upload to google cloud storage by extending the `multer.StorageEngine` class.
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