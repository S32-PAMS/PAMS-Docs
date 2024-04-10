---
{"dg-publish":true,"permalink":"/frontend/camera-server/","tags":["archi","software","camera","streaming","video"],"noteIcon":""}
---



> [!attention]
> Ensure that the hosting machine for the camera server is connected to the same network as the camera(s), because the camera is identified by their IP addresses in a network. This is similar for all components in PAMS, including the hardware [[Tags\|Tags]] and [[Anchors\|Anchors]]. You may use a VPN if necessary.

By now, you should be able to use the web application to register an account and interact with our web application, accessing maps and tags, given that you have already set up the other components of the server. Refer to [[Architecture\|Architecture]] for the list of components you have to set up to get a full working prototype.

The Real-Time Streaming Protocol (RTSP) is the standard streaming protocol for most CCTV cameras in the market, but it is not compatible with HTTP streaming. In short, we can't watch the footage served via RTSP on our HTTP web app.

As the RTSP is the standard streaming protocol for most CCTV cameras in the market, this camera server plays a crucial role by converting the stream files in the RTSP format to HTTP-accepted format.

This section is a guide on setting up the camera server.

### Creation of camera server

> [!note]
> This component requires `Nodejs: >= v20.0`  

```bash
# Clone the repository
git clone https://github.com/S32-PAMS/Cam_Server.git

# Go into the repository
cd Cam_Server

# Install dependencies
npm install

# Start the server
node server.js
```

Once the camera server is set up and started up, we can proceed to setting up the cameras.

At this point, you should have cloned the repository of the camera server. Now let's take a look into the folder `"cam_files/"`.  

This is what you should see:

![cam_files.png](/img/user/Attachments/frontend-devs/cam_files.png)

As you can see, there are many files under the `cam_files/` folder. These files are the converted HTTP-accepted stream files from the camera. The camera server we've set up in the guide above will serve the `stream.m3u8` file to our [[Frontend/Camera Server#Web Application\|#Web Application]] in order to view the streams from the browser. 

> [!note]
> Each camera requires a different folder to save the converted stream files on the hosting machine. We only have 1 camera in our prototype, and hence there is only 1 folder to store the converted stream files.
> 
> In the case of multiple cameras, the developer will need to create multiple folders to store the converted stream files for each camera. The image above shows an example of organising the directories. The developer can define the directory for each camera in the `.env` file to keep the codebase neat and prevent confusion.

> [!aside]- Dockerisation
> To Dockerise the camera server, here is the Dockerfile.
> ```docker
> FROM node:latest
> 
> WORKDIR /usr/src/app
> 
> # Argument for Github Personal Access Token
> ARG GITHUB_PAT
> 
> # Configure Git to use the PAT
> RUN git config --global url. "https://${GITHUB_PAT}@github.com/".insteadOf "https://github.com"
> 
> # Clone your private repo
> RUN git clone https://github.com/S32-PAMS/Cam_Server.git
> 
> # Change the working dir to root of cloned repo
> WORKDIR /usr/src/app/Cam_Server
> 
> # Install dependencies
> RUN npm install
> 
> # Run bash script to install ffmpeg
> RUN ./ffmpeg.sh
> 
> # Run bash script to start converting
> RUN ./convert.sh
> 
> EXPOSE 1989
> CMD ["node", "server.js", "dev"]
> ```
> 
> To build this image, pass the PAT as a build argument:
> 
> ```bash
> docker build --build-arg GITHUB_PAT="your_pat_here" -t your-image-name
> ```



### RSTP Conversion

After going through the camera server functionality, let's dive into how to convert the RSTP stream files.

We will be using an open-source decoder software called `ffmpeg` to convert the RSTP formatted stream files into HTTP-accepted format.

#### Install `ffmpeg`

Assuming an Ubuntu environment:

```bash
#!/bin/bash

# Update package lists to ensure you can download the latest versions
sudo apt-get update

# Install FFmpeg
sudo apt-get install -y ffmpeg

# Verify the installation
ffmpeg -version   
```

#### Example to convert streams

Now that `ffmpeg` is installed, you are ready to receive incoming streams from cameras, and convert them to HTTP-friendly format. This is an example of converting stream from cameras under the TP-Link Tapo series.

```bash
#!/bin/bash

# Change to the directory that stores stream files.
# Replace "/path/to/your/stream/files" with the actual path where you want to save the HLS files.
cd /path/to/your/stream/files

# Run FFmpeg to convert RTSP stream to HLS format.
# Replace "192.168.195.105/stream" with your actual RTSP stream URL if different.
ffmpeg -i rtsp://CameraAccountName:CameraAccountPassword@192.168.195.105/stream1 -c:v copy -c:a copy -f hls -hls_time 2 -hls_playlist_type event stream.m3u8 
```

Because TP-Link Tapo implemented an extra layer of security on their cameras, users who wish to receive the camera streams will need to register for an "Camera Account" for each camera. This is the extra security layer which TP-Link implemented on their cameras.

Hence, the streaming URL for TP-Link Tapo cameras is in this format: `rtsp://CameraAccountName:CameraAccountPassword@CameraIP/stream1`.

> [!note]
> TP-Link Tapo cameras support optional streaming quality:
> - Higher quality streaming will be served at `/stream1`
> - Lower quality streaming will be served at `/stream2`
> 
> For more information, visit [Tapo's official documentation](https://www.tp-link.com/us/support/faq/2680/).

Different vendors may have different properties on their camera's streaming URL. For example, streaming URL for cameras without any security implementation will be `rtsp://CameraIP/stream`.

### Camera Setup

> [!note]
> In this guide, we will only be providing guides for Tapo cameras, if you are dealing with systems using other CCTV cameras, you will need to refer to the respective documentations.

#### 1. Set up a camera (Skip if already set up)

TP-Link Tapo cameras requires the Tapo app to set up. The Tapo app is the official app from TP-Link which is available on both Android and iOS. Here is an example on how to install the Tapo app and set up the TP-Link Tapo cameras.

![camera-setup.png](/img/user/Attachments/frontend-devs/camera-setup.png)

[You may visit the official guide for more details.](https://www.tp-link.com/us/support/download/tapo-c200/#Apps)  

#### 2. Create a camera account

[Follow this guide to create a camera account for each camera](https://www.tapo.com/sg/faq/76/)  

> [!note]
> The camera account is not the same as your Tapo App account. A camera account is, literally, an account for each camera, while your Tapo App account is an account for you as a user to use the Tapo App. (Though, using the same credentials for every camera account/using same credentials for both your Tapo app account and camera account is an option but may have security risk.)

#### 3. Find the IP address for each camera

[Follow this guide to find the IP for each camera.](https://www.tapo.com/sg/faq/27/)   

This is an important step for linking the camera to our system, as identification of cameras rely on their IP addresses.

### Link Cameras to [[Frontend/Camera Server#Camera Server\|#Camera Server]]

After [[Frontend/Camera Server#Camera Setup\|#Camera Setup]], we have all the required information we need.

```bash
# go to the camera files folder in the camera server repo
cd cam_files
```

Run `rtsp://CameraAccountName:CameraAccountPassword@CameraIP/stream1` for high quality stream files conversion or `rtsp://CameraAccountName:CameraAccountPassword@CameraIP/stream2` for lower quality stream files conversion.

[Follow the official guide for further help on how to run this stream](https://www.tapo.com/sg/faq/34/).

At this point, you will see the files being generated. You may now access the camera stream from the [[Frontend/Frontend\|Frontend]].

For how to run the camera server in a complete PAMS prototype implementation, see [[Guides/For Running Server#Run the Camera Servers\|For Running Server#Run the Camera Servers]]