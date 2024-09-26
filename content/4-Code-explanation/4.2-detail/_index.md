---
title : "Detailed explanation"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 4.2 </b> "
---

This subsections presents implementation detail of NodeJS read/write streams and Multer custom engine working with AWS S3 SDK.

## Class AWSS3FileWriteStream
Implement a write stream to a AWS S3 Bucket file path with 3 steps of operation:
- Initial a Multipart upload in `_constructor` method using `CreateMultipartUploadCommand` class.
- Writing byte chunks multiple time, by calling `UploadPartCommand` class.
- Finish writing in `_final` method (called after `end` event is emitted to the write stream) calling `CompleteMultipartUploadCommand` class.
- In case of write error, `DeleteObjectCommand` class is called
 
```js
import { s3 } from 'service/fileSystem/awsS3Config';
import {
    _Object,
    CreateMultipartUploadCommand,
    UploadPartCommand,
    CompleteMultipartUploadCommand,
    DeleteObjectCommand
} from "@aws-sdk/client-s3";
import { jsonSecret } from "service/fileSystem/awsS3Config";
import { Writable } from "node:stream";

export class AWSS3FileWriteStream extends Writable {
    highWaterMark: number;
    filePath: string;
    chunks: Uint8Array = new Uint8Array();
    writesCount: number = 0;
    uploadResultsPromise: Promise<any>[] = [];
    uploadId: string; // AWS S3 Object Multipart Id, retain it in the whole process
    constructor({ highWaterMark, filePath }) {
        super({ highWaterMark });
        this.highWaterMark = highWaterMark;
        this.filePath = filePath;
    }

    /**
     * 
     * @param callback This optional function will be called in a tick after the stream constructor has returned, 
     * delaying any _write(), _final() and _destroy() calls until callback is called. 
     * This is useful to initialize state or asynchronously initialize resources before the stream can be used.
     */
    _construct(callback: (error?: Error | null) => void): void {
        console.log('AWS S3 write stream _construct called');
        s3.send(
            new CreateMultipartUploadCommand({
                Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                Key: this.filePath,
            }),
        ).then((multipartUpload) => {
            console.log('AWS S3 write stream _construct called');
            console.log('CreateMultipartUploadCommand success uploadId:', multipartUpload.UploadId);
            this.uploadId = multipartUpload.UploadId;
            callback();
        }).catch((err) => {
            callback(err);
        });
    }

    chunksConcat(newChunk: Uint8Array): void {
        var mergedArray = new Uint8Array(this.chunks.byteLength + newChunk.byteLength);
        mergedArray.set(this.chunks);
        mergedArray.set(newChunk, this.chunks.byteLength);
        this.chunks = mergedArray;
    }

    _write(chunk: any, encoding: BufferEncoding, callback: (error?: Error | null) => void): void {
        console.log('AWS S3 write stream _write called chunk:', chunk);
        // chunk = Uint8Array.from(chunk);
        this.chunksConcat(chunk);
        if (this.chunks.byteLength >= this.highWaterMark) {
            this.writesCount += 1;
            let partNumber = this.writesCount;
            let uploadingChunk = this.chunks;
            this.chunks = new Uint8Array(0);
            // console.log('uploadingChunk', uploadingChunk);
            // console.log('uploadingChunk.byteLength:', uploadingChunk.byteLength);
            // console.log(`AWSWriteStream uploading part ${partNumber}, ${uploadingChunk.byteLength} bytes`);
            this.uploadResultsPromise.push(
                s3.send(new UploadPartCommand({
                    Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                    Key: this.filePath,
                    UploadId: this.uploadId,
                    Body: uploadingChunk,
                    PartNumber: partNumber
                })).then((uploadResult) => {
                    console.log(`File ${this.filePath} write part number`, partNumber, "uploaded");
                    console.log('Write stream uploadResult', uploadResult);
                    return uploadResult;
                }).catch((err) => {
                    console.log('WriteStream UploadPartCommand error', err);
                    callback(err);
                })
            );
            callback();
        } else {
            // when we are done, we should call the callback function
            callback();
        }
    }

    // this will run after the our stream has finished
    _final(callback) {
        if (this.chunks.byteLength > 0) {
            this.writesCount += 1;
            let partNumber = this.writesCount;
            let uploadingChunk = this.chunks;
            this.chunks = new Uint8Array(0);
            // console.log(`AWSWriteStream uploading last part ${partNumber}, ${uploadingChunk.byteLength} bytes`);
            this.uploadResultsPromise.push(
                s3.send(new UploadPartCommand({
                    Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                    Key: this.filePath,
                    UploadId: this.uploadId,
                    Body: uploadingChunk,
                    PartNumber: this.writesCount
                })).then((uploadResult) => {
                    console.log(`File ${this.filePath} write part number`, partNumber, "uploaded");
                    return uploadResult;
                }).catch((err) => {
                    console.log('WriteStream last part UploadPartCommand error', err);
                    callback(err);
                })
            );
        }
        console.log('AWS S3 write stream _final called');
        Promise.all(this.uploadResultsPromise)
            .then((uploadResults) => {
                console.log('CompleteMultipartUploadCommand config', {
                    Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                    Key: this.filePath,
                    UploadId: this.uploadId,
                    MultipartUpload: {
                        Parts: uploadResults.map(({ ETag }, i) => ({
                            ETag,
                            PartNumber: i + 1,
                        })),
                    },
                });
                s3.send(
                    new CompleteMultipartUploadCommand({
                        Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                        Key: this.filePath,
                        UploadId: this.uploadId,
                        MultipartUpload: {
                            Parts: uploadResults.map(({ ETag }, i) => ({
                                ETag,
                                PartNumber: i + 1,
                            })),
                        },
                    }),
                )
                    .then((_) => callback())
                    .catch((err) => { console.log('CompleteMultipartUploadCommand error', err); callback(err) });
            })
            .catch((err) => {
                console.log('Promise.all(this.uploadResultsPromise)', err);
                callback(err);
            });
    }

    _destroy(error, callback) {
        console.log("Write Count:", this.writesCount);
        if (error) {
            const deleteCommand = new DeleteObjectCommand({
                Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                Key: this.filePath,
            })
            s3.send(deleteCommand).then((_) => callback(error)).catch((_) => callback(error));
        } else {
            callback();
        }
    }
}
```

## Class AWSS3FileReadStream
Implement a read stream from a AWS S3 Bucket file path, each time calling `getObjectRange` with desired byte range.

```js
import { AbstractFileSystemAction } from "./localFileSystemAction";
import { s3 } from 'service/fileSystem/awsS3Config';
import {
    ListObjectsV2Command,
    GetObjectCommand,
    DeleteObjectCommand,
    DeleteObjectsCommand,
    _Object,
    CreateMultipartUploadCommand,
    UploadPartCommand,
    CompleteMultipartUploadCommand
} from "@aws-sdk/client-s3";
import { jsonSecret } from "service/fileSystem/awsS3Config";
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { Writable, Readable } from "node:stream";

export const getObjectRange = ({ bucket, key, start, end }) => {
    const command = new GetObjectCommand({
        Bucket: bucket,
        Key: key,
        Range: `bytes=${start}-${end}`,
    });

    return s3.send(command);
};

/**
 * @param {string | undefined} contentRange
 */
export const getRangeAndLength = (contentRange: string) => {
    contentRange = contentRange.slice(6);
    console.log('getRangeAndLength input', contentRange);
    const [range, length] = contentRange.split("/");
    const [start, end] = range.split("-");
    return {
        start: parseInt(start),
        end: parseInt(end),
        length: parseInt(length),
    };
};

export const isComplete = ({ end, length }) => end === length - 1;

export class AWSS3FileReadStream extends Readable {
    filePath: string;
    highWaterMark: number;
    lastRange = { start: -1, end: -1, length: -1 };
    nextRange = { start: -1, end: -1, length: -1 };

    constructor({ highWaterMark, filePath }) {
        super({ highWaterMark });
        this.filePath = filePath;
    }

    _construct(callback: (error?: Error | null) => void): void {
        console.log('AWS read stream _construct called');
        callback();
    }

    chunksConcat(newChunk: Uint8Array): void {
        var mergedArray = new Uint8Array(this.chunks.byteLength + newChunk.byteLength);
        mergedArray.set(this.chunks);
        mergedArray.set(newChunk, this.chunks.byteLength);
        this.chunks = mergedArray;
    }

    _read(size) {
        console.log('AWS read stream _read called');
        if (isComplete(this.lastRange)) {
            this.push(null);
            return;
        }
        const { end } = this.lastRange;
        this.nextRange = { start: end + 1, end: end + size, length: size };

        getObjectRange({
            bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
            key: this.filePath,
            ...this.nextRange,
        }).then(({ ContentRange, Body }) => {
            const contentRange = getRangeAndLength(ContentRange);

            console.log('_read contentRange', contentRange);
            this.lastRange = contentRange;
            Body.transformToByteArray()
                .then((chunk) => {
                    this.chunksConcat(chunk);
                    let uploadingChunk = this.chunks;
                    this.chunks = new Uint8Array(0);
                    console.log('read stream push chunk', uploadingChunk);
                    this.push(uploadingChunk);
                })
                .catch((error) => {
                    console.log('Body.transformToByteArray() error', error);
                });
        });
    }

    _destroy(error, callback) {
        callback();
    }
}
```

## Class AWSS3CustomStorageEngine
Extending `multer.storageEngine` and implementing 2 method `_handleFile` and `_removeFile`:
- `_handleFile`: initialize multipart upload with `CreateMultipartUploadCommand` command, getting read stream from the uploaded file (`file.stream`) and read all file data into a buffer, then splice buffer into multiple of 6MiB chunks and upload to AWS one-by-one, calling the `UploadPartCommand` command; after finishing, call the `CompleteMultipartUploadCommand` to finalize uploading process.
- `_removeFile`: call `DeleteObjectCommand` to remove file in case of upload failure.