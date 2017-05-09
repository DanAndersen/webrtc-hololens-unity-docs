## Setting up Example Code for WebRTC from HoloLens Unity to Desktop C++ Application

This is a series of test programs I wrote in Spring 2017 as part of a research project, in which video streaming was needed from a HoloLens Unity app (the "trainee") to a remote C++ desktop application viewing frames via OpenCV (the "mentor"). This code is provided as a starting point so that others may hopefully have a template to work off of. This code is provided as is and I won't be able to give much in the way of troubleshooting for your own endeavors.

### Signalling server

**Code: https://github.com/DanAndersen/webrtcrecorder-client-andersed**

WebRTC requires that both peers have access to a "signalling server". This is a server app that is accessible via Internet, that both peers use to register themselves with. Using the signalling server, both the trainee and mentor app indirectly indicate to each other how to best communicate with each other. After signalling is complete, trainee-mentor communication happens independently of the signalling server (i.e. not all video traffic is routed via the signalling server).

We are using a slightly modified version of libsourcey's "webrtcrecorder-client" as a signalling server (https://github.com/sourcey/libsourcey/tree/master/src/webrtc/samples/webrtcrecorder) This runs as a node.js app on a normal machine (for example, it can run on the same machine as the mentor system).

First, install node.js and the npm package manager: http://blog.teamtreehouse.com/install-node-js-npm-windows

Next, clone the webrtcrecorder-client-andersed Git repo.

In a console, go to the directory that was checked out, and run "npm install". This will add all the node.js dependencies needed to run the server.

Before running the app, however, we need a publicly available hostname for the server (not an IP address). This is for two reasons: (1) it's more convenient for the trainee/mentor apps to not have to deal with a changing dynamic IP, and (2) we have to do WebRTC signalling traffic over HTTPS and not HTTP.

To get a hostname, I recommend using No-IP's free DDns service: https://www.noip.com/free You can sign up for an account free, which gives you a hostname of the form "myservername.ddns.net". There is also a Windows app that you can download from them that you can have run in the background, and every time your dynamic IP changes, they will update their records so that "myservername.ddns.net" will always point to the right IP address. One important thing to note is that the free service requires you to renew the hostname every 30 days; this is done by them automatically sending you an email saying it's about to expire, and you clicking a button on their webpage. Pretty simple but you need to remember to do it.

To get HTTPS working with this, we need to set up SSL certificates for our hostname. The best way to do this for free is to use Let's Encrypt to generate certificates and keys. Unfortunately the code to do this (certbot) has to be run on Linux instead of Windows, but once you generate them you can take the files and copy them over to the machine where you will be running the server. See https://letsencrypt.org/ for more details. 

Once you have generated the certificates, you'll have the following files: "privkey1.pem, fullchain1.pem, chain1.pem, and cert1.pem". Copy them into the "certs" directory of the "webrtcrecorder-client-andersed" project, overwriting the existing certs (which are for my own test server).

At this point, you should be able to run "node app.js" from the console and have the server run using your new certs on port. The websocket/WebRTC connections will be running on https://myservername.ddns.net:443 and a Web page should appear at https://myservername.ddns.net:442. Verify that you can open a browser and go to the address on port 442 on various machines.

This app will need to be running in the background whenever the trainee and mentor apps are communicating with each other.

The next thing to do is to get the libsourcey C++ app running, which will be able to accept incoming video data. Once that is set up, you can use the Web page on port 442 to communicate with it, sending video data via a normal browser to the C++ app. Later in this guide we will set up a UWP plugin and Unity app that will act as the "browser" to send video data to the C++ app.

### libsourcey C++ app

**Code: https://github.com/DanAndersen/webrtc-mentor-client**

I've set up a simple test app called WebRTCMentorClient that acts as a test example of a C++ app that can connect to our WebRTC signalling server and receive frames from a connected client. The code is heavily dependent on libsourcey for all of its WebRTC integration. 

First, pull down the webrtc-mentor-client Git repo somewhere. Then, make sure you have the following dependencies set up:

OpenCV (I use version 3.2.0)

ffmpeg (the version I have set up is "ffmpeg-20170225-7e4f32f-win64-dev" -- you'll need both the shared and dev versions from https://ffmpeg.zeranoe.com/builds/ )

WebRTC. Building WebRTC is a nightmare. Thankfully, you shouldn't have to do it. Sourcey provides prebuilt versions of WebRTC: https://sourcey.com/precompiled-webrtc-libraries/ I used version 58. Note that the Windows download doesn't include the headers; for that you'll need to download the linux version too and copy over the "include" directory. Make a directory somewhere and put the "libs" and "include" folders in it.

OpenSSL (I don't actually know if this is needed since WebRTC comes with its own version, but if you do end up needing it, get it from https://slproweb.com/products/Win32OpenSSL.html and make sure to get the 1.0.2 version and not the 1.1.0 version)

libsourcey (pull it from https://github.com/sourcey/libsourcey/ -- at this point I think you only need to download the source; when we build webrtc-mentor-client it will build the necessary libraries in libsourcey)

Next, you'll need to use CMake to build the Visual Studio project from the provided source code in the webrtc-mentor-client repo. Open up CMakeLists.txt and change the values for the following variables to the locations of your actual dependencies:
`OpenCV_DIR`
`OPENSSL_ROOT_DIR`
`FFMPEG_ROOT_DIR`
`WEBRTC_ROOT_DIR`
`LibSourcey_DIR`

Also take the time to read through the CMakeLists.txt file to understand what it's doing and how you can apply a similar process to your own build.

Make sure you have CMake for Windows installed and open up the cmake-gui. Set "E:/Dev/webrtc/webrtc-mentor-client" for "where is the source code" and "E:/Dev/webrtc/webrtc-mentor-client/build" for "where to build the binaries" (change to fit your own download locations). Press Configure. Create the "build" directory if it doesn't exist. Specify "Visual Studio 14 2015 Win64" as the generator. Check the output for errors. If everything looks good, press "Configure" again until there are no more red entries. Then, press Generate to create the Visual Studio project files, then Open Project to open Visual Studio.

Most of the projects in the solution are dependencies from libsourcey. "WebRTCMentorClient" is our own code. You'll want to read through all the code in that project, but in particular note:
config.h lists the signalling server that you'll be connecting to. Change this to your own signalling server. It also lists OUTPUT_WIDTH and OUTPUT_HEIGHT; when incoming frames arrive over WebRTC, the CvStreamRecorder class will auto-resize them to that resolution.
Audio transmission is currently disabled, as I was having troubles with it early on and hadn't investigated further. I recommend for the time being that trainee and mentor use a phone to communicate via audio.
At the beginning of main() in main.cpp, you can change the level of libsourcey's logs. This can be good for seeing what is happening, but too many trace logs can slow things down.

When you run WebRTCMentorClient after building it, the console will appear and show the application connecting to the signalling server. It will then wait for an incoming peer to get frames from. You can test this by going to the Web interface in the signalling server (e.g. "https://andersed-talos.ddns.net:442" on my machine). Video from your webcam on your computer should automatically stream and be received as RGB frames in WebRTCMentorClient, at which point they appear as OpenCV Mats and are drawn using the OpenCV GUI.

### UWP plugin: WSAUnity and WSAStub

**Code: https://github.com/DanAndersen/WSAUnity**

In order to use the WebRTC .NET APIs (which require .NET 4.5) in Unity (which only supports up to 3.5), we have to set up a stub-plugin. This is essentially a Universal Windows solution that has two projects:
a plugin that has the whole codebase (WSAUnity)
a plugin that has identical public class/method names but without any of the new code (WSAStub).
We do this by setting up the two projects in the solution, setting WSAStub to only support .NET 3.5, and wrapping all such code in `#if NETFX_CORE ... #endif` macros. At this point I recommend looking at and following this particular tutorial: https://channel9.msdn.com/Blogs/2p-start/Combining-Windows-10-features-with-Unity-games-in-your-own-plugin and then reading my HoloLens forum posts at https://forums.hololens.com/discussion/comment/14258/#Comment_14258 before proceeding.

The WSAUnity solution contains both the full plugin, the stub plugin, and a test Universal Windows 2D application that shows how to use the plugin code. You'll need to adapt the plugin code to how you are organizing your trainee system (in terms of when you connect/disconnect/reconnect to a mentor), but the general program flow is there. The codebase is an almost-complete function-for-function port of the "webrtcrecorder" client code in the libsourcey codebase. In particular, you should note the setting of CLIENT_OPTIONS in Plugin.cs, and change the signalling server URL to your own.

The test Universal Windows app is called PluginTestApp and you should run it to make sure it works. This code can be deployed either on the desktop or on the HoloLens as a 2D (non-Unity) app. When it runs, you will see a textbox and button. Pressing the button will initialize the socket connection to the WebRTC signalling server and then start sending frames to the WebRTCMentorClient app. Before running the codebase in Unity, make sure that this works right. Also check the required app permissions, as any Unity app you make will need to have the same permissions set.

## Unity app - WinPluginTest

**Code: https://github.com/DanAndersen/WinPluginTest**

Finally we come to the Unity codebase that uses the above plugin. When you open the main scene, you'll see the normal HoloLens cursor and camera, along with a UI that floats in front of the user. The UI displays a "Test Init" button and a window to display debug log output. There is also a "WebRTCTestManager" object that shows an example of a Unity script using the plugin that we created.

Most importantly, note the "Plugins" folder and the "WSA" folder inside that one. As described in the channel9 MSDN tutorial above, you need to set up the plugin project settings so that WSAUnity and WSAStub are built in these folders, otherwise you'll need to copy them in here every time you make a change.

The folder structure is:
```
Plugins folder
    WSAUnity.dll (Include only Editor)
    WSA folder
        WSAUnity.dll (Include only WSAPlayer, Check "Don't Process", and set "Assets/Plugins/WSAUnity.dll" as placeholder)
        EngineIoClientDotNet.dll (necessary library)
        Newtonsoft.Json.dll (necessary library)
        SocketIoClientDotNet.dll (necessary library)
```

If all goes well, you should be able to build and run this like any other HoloLens app. Any of the code that was excluded by the `#if NETFX_CORE ... #endif` macros will not run in the Unity editor, but will run when actually executing the app in the HoloLens. When the app loads up on the HoloLens, press the Test Init button and after some time you'll see the log statements indicating that a connection is being made. Eventually the video frames will start streaming to the mentor app.