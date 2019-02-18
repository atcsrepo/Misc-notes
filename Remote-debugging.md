# Remote Debugging Android Chrome/Firefox on Windows

While there is a guide from Google on how to [remote debug](https://developers.google.com/web/tools/chrome-devtools/remote-debugging/), they have failed to mention the need for ADB (Android Debug Bridge) and proper drivers. Likewise, Firefox appears to need ADB and the associated drivers as well.

----

### Installing ADB:

To get ADB, there are two options. The first is to install the entire Android Studio package (~1 GB in size). Following which, you should be able to find ADB.exe at the default path:

`C:\Users\<User Name>\AppData\Local\Android\Sdk\platform-tools
`

Alternatively, it is possible to download ADB directly, and much more quickly, considering it's only a few megabytes.

Google does provide a direct download [here](https://developer.android.com/studio/releases/platform-tools#downloads).

Otherwise, it is also possible to download older versions from the [XDA forums](https://forum.xda-developers.com/showthread.php?t=2317790).

The ADB file can be executed via command prompt, with the most relevant commands being:

`adb devices`

`adb start-server` 

`adb kill-server`

which are used to check connected devices, start server and stop server, respectively. If lucky, things should work out at this point. If not, then you might get an error along the lines of:

`adb server version(36) doesn't match this client(40)`

or some variation of it. At this point, you can either curse the powers that be and cry stoically, or forge on, or maybe both. But anyway, this error seems largely related to path issues and multiple installs of ADB, as can be seen on [SO](https://stackoverflow.com/questions/53067917/adb-server-version-39-doesnt-match-this-client-40-killing-adb-server-di), where there are a number of posts on the problem. Solutions generally revolve around defining the path to a specific version of ADB or removing extra copies. Of course, that's assuming Window&#39;s search would be kind enough to turn up something useful.

Of note was the observation that running the ADB downloaded from the XDA forums produced a slightly different error as compared to running the ADB downloaded from Google. More specifically, running the ADB from the XDA forums produced an error wherein the server version was greater than the client version. In contrast, the one downloaded from Google had the opposite error. One difference between the two packages are the .dll files. It turns out that once you identify a pair of packages with matching server and client versions, it is possible to just simply copy and paste the .dll files over, which can alleviate the server/client mismatch problem. 

----

### Setting up drivers:

After ADB is working, next comes setting up the relevant drivers. To confirm if everything is working, go to Control Panel > Device Manager. If installed properly you should be able to find the device under "Android Device" or something similar; otherwise, if the device is found under "Other devices" > "ADB Interface&#34;, then there may be a problem.

To get around this, go to the SDK manager in Android Studio (or for a direct download from Google, [go here](https://developer.android.com/studio/run/win-usb)) and download the Google USB Driver (or an OEM alternative). Go back to Device Mananger and update the driver for the ADB Interface device.

To do so:
Go "Browse my computer for software driver" > "Let me pick" > "Have Disk" and select the prior downloaded USB driver.

This should fix the ADB Interface problem and allow ADB to detect the phone/device.

Before going further, ensure that USB debugging is enabled on the device.

----

### Remote connection for Chrome:

On the host machine, open up the Developer Tools and go to "More Tools", which can be found by clicking the 3 dots in the upper right hand corner. Select Remote Devices.

At this point, it should be possible to see the device as connected. Clicking on the device will display the current tabs running on the mobile device and allow for inspecting of said tabs. When inspecting tabs, a new console window will open up. In some instances, it may say "HTTP/1.1 404 Not Found."

If this happens, go to "chrome://inspect/#devices" and use "inspect fallback", as the remote browser may be newer than the client browser.

### Remote connection for Firefox:

Under the Web Developer option, look for WebIDE. In the WebIDE, it should be possible to see the device listed under "USB DEVICES". Clicking on the selected device will prompt an authorization request on the device itself. If allowed, then the browser tabs should show up under "TABS" on the right-side of the WebIDE. If nothing shows up, try restarting FireFox on the device. Clicking on any of the tabs should bring up a Web Dev tool/console instance.
