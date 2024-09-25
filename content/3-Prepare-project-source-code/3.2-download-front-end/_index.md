---
title : "Run front-end web application to upload and stream video"
date :  "`r Sys.Date()`" 
weight : 2
chapter : false
pre : " <b> 3.2 </b> "
---

## Deploy Front-end source code
Clone the front-end source code from [https://github.com/hdthinh1012/aws-workshop-0-hls-streaming-fe](https://github.com/hdthinh1012/aws-workshop-0-hls-streaming-fe "Frontend Github Link").
![clone-front-end](/images/3-Project-source-code/3.2-download-front-end/clone-front-end.png)

Run `npm install` to install all the dependencies and open `.env`
![npm-install-env](/images/3-Project-source-code/3.2-download-front-end/npm-install-env.png)

Edit the IP address and port that your backend is currently listening on.
![env-config](/images/3-Project-source-code/3.2-download-front-end/env-config.png)

Run `npm run dev` to start the Vite React App in hot-reload mode.
![npm-run-dev](/images/3-Project-source-code/3.2-download-front-end/npm-run-dev.png)

## Upload and stream video demo
Open the `http://localhost:5173/upload`:
![demo-1](/images/3-Project-source-code/3.2-download-front-end/demo-1.png)
Click `Choose File` button and upload your video (Recommend an MP4 file format with H264 video and AAC audio encoding).
![demo-2](/images/3-Project-source-code/3.2-download-front-end/demo-2.png)
Then click `Upload Video` button, the front end will send a initial request for the server to setup uploading process. If the setup succeed, an alert appears showing the file chunk size and number of chunks will be split from file to upload to server. Click `OK` to start the uploading process
![demo-3](/images/3-Project-source-code/3.2-download-front-end/demo-3.png)
The server showing uploading message
![demo-4](/images/3-Project-source-code/3.2-download-front-end/demo-4.png)
![demo-5](/images/3-Project-source-code/3.2-download-front-end/demo-5.png)
![demo-6](/images/3-Project-source-code/3.2-download-front-end/demo-6.png)
![demo-7](/images/3-Project-source-code/3.2-download-front-end/demo-7.png)
After uploading chunks, the server will merge chunks into original video file, then run FFMPEG command to generate HLS master playlist (which contains several folder with `.ts` file contain video sequence and `.m3u8` contain metadata for video sequence order)
![demo-8](/images/3-Project-source-code/3.2-download-front-end/demo-8.png)
The uploaded video original file saved in `uploads` folder
![demo-8A](/images/3-Project-source-code/3.2-download-front-end/demo-8A.png)
The generate HLS playlist saved in `streams` folder
![demo-8B](/images/3-Project-source-code/3.2-download-front-end/demo-8B.png)
![demo-8C](/images/3-Project-source-code/3.2-download-front-end/demo-8C.png)
Back to the front-end web application, change to `stream` tab. All the HLS playlist folder generated and saved in `streams` is listed here.
![demo-9](/images/3-Project-source-code/3.2-download-front-end/demo-9.png)
Click `Click here` button to open video stream page
![demo-10](/images/3-Project-source-code/3.2-download-front-end/demo-10.png)
The video is adaptively streamed from the server by first loading the `streams/sintel_trailer.mp4/master.m3u8` file, then `hls.js` npm package will handle the streaming process.
![demo-11](/images/3-Project-source-code/3.2-download-front-end/demo-11.png)
The network inspector show that `hls.js` automatically chose 360P resolution based on network status. User can choose between 360P, 480P, 720P, 1080P in the select form below the video.
![demo-12](/images/3-Project-source-code/3.2-download-front-end/demo-12.png)