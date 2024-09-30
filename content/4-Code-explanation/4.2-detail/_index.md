---
title : "Detailed explanation"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 4.2 </b> "
---

This subsections presents implementation detail of NodeJS read/write streams and Multer custom engine working with AWS S3 SDK.

## S3Client configuration
Initialize S3Client object with credentials, region and other HTTP configurations.
```js
import { S3Client } from '@aws-sdk/client-s3';
import dotenv from 'dotenv';
dotenv.config();
import { NodeHttpHandler } from "@smithy/node-http-handler";
import https from "https";

let jsonSecret = {
    AWS_ACCESS_KEY_ID: process.env.AWS_ACCESS_KEY_ID,
    AWS_SECRET_ACCESS_KEY: process.env.AWS_SECRET_ACCESS_KEY,
    BUCKET_NAME: process.env.BUCKET_NAME,
};

let s3 = new S3Client({
    credentials: {
        accessKeyId: jsonSecret.AWS_ACCESS_KEY_ID,
        secretAccessKey: jsonSecret.AWS_SECRET_ACCESS_KEY,
    },
    region: "us-east-1",
    // Use a custom request handler so that we can adjust the HTTPS Agent and
    // socket behavior.
    requestHandler: new NodeHttpHandler({
        httpsAgent: new https.Agent({
            maxSockets: 500,

            // keepAlive is a default from AWS SDK. We want to preserve this for
            // performance reasons.
            keepAlive: true,
            keepAliveMsecs: 1000,
        }),
        socketTimeout: 900000,
    }),
});

export { s3, jsonSecret };
```

Next initalize singleton objects for file system path and file system action based on environment variable `IS_AWS_S3`, create neccessary directories.

```js
import dotenv from 'dotenv';
dotenv.config();

import { AbstractFileSystemAction, LocalFileSystemAction } from 'service/fileSystem/localFileSystemAction';
import { AbstractFileSystemPath, LocalFileSystemPath } from 'service/fileSystem/localFileSystemPath';
import { AWSS3FileSystemAction } from 'service/fileSystem/awsS3FileSystemAction';
import { AWSS3FileSystemPath } from 'service/fileSystem/awsS3FileSystemPath';

let fileSystemActionObject: AbstractFileSystemAction;
let fileSystemPathObject: AbstractFileSystemPath;
const isAWSS3 = process.env.IS_AWS_S3 === '1';
if (isAWSS3) {
    fileSystemActionObject = new AWSS3FileSystemAction();
    fileSystemPathObject = new AWSS3FileSystemPath();
} else {
    fileSystemActionObject = new LocalFileSystemAction();
    fileSystemPathObject = new LocalFileSystemPath();
}

fileSystemActionObject.createDir(fileSystemPathObject.streamDirectoryAbsolutePath(), { recursive: true });
fileSystemActionObject.createDir(fileSystemPathObject.uploadVideoDirectoryAbsolutePath(), { recursive: true });
fileSystemActionObject.createDir(fileSystemPathObject.uploadChunkDirectoryAbsolutePath(), { recursive: true });

export { fileSystemActionObject, fileSystemPathObject };
```

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

```js
import { Request } from "express";
import multer from "multer";
import path from "path";

import {
    CreateMultipartUploadCommand,
    UploadPartCommand,
    CompleteMultipartUploadCommand,
    AbortMultipartUploadCommand,
    S3Client,
    DeleteObjectCommand,
} from "@aws-sdk/client-s3";
import { s3, jsonSecret } from "service/fileSystem/awsS3Config";
import { fileSystemPathObject } from "initFs";

type nameFnType = (req: Request, file: Express.Multer.File) => string;

type Options = {
    nameFn?: nameFnType
}

const defaultNameFn: nameFnType = (
    _req: Request,
    file: Express.Multer.File
) => {
    return file.originalname;
};

interface CustomFileResult extends Partial<Express.Multer.File> {
    name: string;
}

export const multipartUploadPartSize = 6 * 1024 * 1024;

export class AWSS3CustomStorageEngine implements multer.StorageEngine {
    private nameFn: nameFnType;

    constructor(opts: Options) {
        this.nameFn = opts.nameFn || defaultNameFn;
    }

    async _handleFile(req: Request, file: Express.Multer.File, callback: (error?: any, info?: Partial<Express.Multer.File>) => void): Promise<void> {
        if (!s3) {
            callback(new Error("S3 Client not exist!!!"));
        }

        const fileName = this.nameFn(req, file);
        const chunkReadStream = file.stream;
        const chunkS3Path = fileSystemPathObject.uploadChunkFilePath(fileName);

        let uploadId: string;
        let tmpBuffer: Buffer = Buffer.alloc(0);
        try {
            const multipartUpload = await s3.send(
                new CreateMultipartUploadCommand({
                    Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                    Key: chunkS3Path,
                }),
            );
            uploadId = multipartUpload.UploadId;
            const uploadResults = [];

            let partNumberCnt = 0;
            chunkReadStream.on('data', async (chunk: Buffer) => {
                tmpBuffer = Buffer.concat([tmpBuffer, chunk]);
            });

            let fileSize;
            chunkReadStream.on('end', async () => {
                fileSize = tmpBuffer.byteLength;
                try {
                    while (tmpBuffer.byteLength > 0) {
                        const cutBuffer = tmpBuffer.subarray(0, multipartUploadPartSize);
                        tmpBuffer = tmpBuffer.subarray(multipartUploadPartSize);
                        let partNumber = partNumberCnt + 1;
                        console.log('Reading buffer length:', cutBuffer.byteLength);
                        const uploadResult = await s3.send(new UploadPartCommand({
                            Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                            Key: chunkS3Path,
                            UploadId: uploadId,
                            Body: cutBuffer,
                            PartNumber: partNumber
                        }))
                        console.log("Part", partNumber, "uploaded");
                        uploadResults.push(uploadResult);
                        console.log('Reading tmpBuffer done length:', tmpBuffer.byteLength);
                        partNumberCnt += 1;
                    }
                } catch (error) {
                    console.error('UploadPartCommand error:', error);
                }

                console.log('Prepare complete upload command');
                try {
                    const res = await s3.send(
                        new CompleteMultipartUploadCommand({
                            Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                            Key: chunkS3Path,
                            UploadId: uploadId,
                            MultipartUpload: {
                                Parts: uploadResults.map(({ ETag }, i) => ({
                                    ETag,
                                    PartNumber: i + 1,
                                })),
                            },
                        }),
                    );
                    callback(null, {
                        fieldname: 'video',
                        originalname: fileName,
                        size: fileSize
                    });
                } catch (error) {
                    console.error('CompleteMultipartUploadCommand error:', error);
                }
            });

            chunkReadStream.on('error', (error) => {
                console.error('Error reading the file:', error);
            });
        } catch (err) {
            if (uploadId) {
                const abortCommand = new AbortMultipartUploadCommand({
                    Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
                    Key: chunkS3Path,
                    UploadId: uploadId,
                });

                await s3.send(abortCommand);
            }
        }
    }

    async _removeFile(req: Request, file: Express.Multer.File, callback: (error: Error | null) => void): Promise<void> {
        const fileName = this.nameFn(req, file);
        const chunkS3Path = fileSystemPathObject.uploadChunkFilePath(fileName);
        const deleteCommand = new DeleteObjectCommand({
            Bucket: jsonSecret.BUCKET_NAME ? jsonSecret.BUCKET_NAME : "",
            Key: chunkS3Path,
        })
        await s3.send(deleteCommand);
    }
}
```