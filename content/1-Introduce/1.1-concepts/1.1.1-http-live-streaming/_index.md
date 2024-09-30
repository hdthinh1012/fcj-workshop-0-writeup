---
title : "HTTP Live Streaming"
date :  "`r Sys.Date()`" 
weight : 1
chapter : false
pre : " <b> 1.1.1 </b> "
---

## HTTP Live Streaming
A protocol of adaptive streaming proposed by Apple, providing ability to stream videos with different bitrate, automatically or manually chosen by the client.

HLS breaks down videos into smaller chunks and delivers them using HTTP protocol. Client front-end compliant to HLS protocol downloads the video chunk and serve to users. HLS protocol provides options to generate and deliver chunks with different video encodings, bitrate, resolution, audio quality.

You can learn more about HTTP Live Streaming in this brilliant guide [here](https://duthanhduoc.com/blog/hls-streaming-nodejs "HLS Streaming với Node.js. Tạo server phát video như Youtube") (Vietnamese)