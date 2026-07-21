---
layout: post
title: "Tinkering with automation. Respond to an end user disabling Gatekeeper."
date: 2023-02-22
slug: tinkering-with-automation-respond-to-an-end-user-disabling-gatekeeper
---

Way too long ago, I posted my last blog around a POC on how to offer self serviced iOS Inventory Updates to end users with Jamf Pro.


I have been playing since then with few other ideas and thought this one was worth to share.


Disclaimer: this is not nearly as close as a production ready workflow, it's mainly a personal learning project to familiarize with python, webhooks and, in general, automation. 


What's the problem we're trying to solve?


End users with admin privileges could potentially disable Gatekeeper. There's enough "junk" forums on the net suggesting to turn off Gatekeeper as a solution for very suspicious apps.


From Apple's website: 


*"macOS offers the Gatekeeper technology and runtime protection to help ensure that only trusted software runs on a user’s Mac."*
[https://support.apple.com/en-gb/guide/security/sec5599b66df/web](https://support.apple.com/en-gb/guide/security/sec5599b66df/web)


We could surely rely on Jamf Pro inventory to track when a user disable Gatekeeper, but that would rely on the device submitting an inventory update, which depending on the configuration chosen by the administrators, sometimes it can take hours/days to have fresh data in.


The idea is to leverage Protect "Add to Smart Group" functionality, Jamf Pro webhooks and MDM framework to make the detection and remediation as fast as possible.


How we can work on this.


Let's start with some diagram to break this down in smaller components.


Key components:


- Jamf Pro


- Jamf Protect


- AWS: API gateway + Lambda + CloudWatch + Secrets Manager


![](/assets/img/workflow.png)


- Jamf Protect will detect that Gatekeper have been disabled (we'll see later how to) and use the "Add to Smart Group" built in feature to add the device to a Smart Group in Jamf Pro.


- Jamf Pro will fire off a webhook - `SmartGroupComputerMembershipChange` - to the API gatway


- API gateway passes the content of the webhook (json formatted file) to Lambda.


- Lambda iterate through the file, determine the event (Gatekeeper disabled) and makes an API call to Jamf Pro.We'll take advantage of the `/api/v1/deploy-package` api endpoint in Jamf Pro.


- A pkg containing a postinstall script to re-enable Gatekeeper and notify the user is sent to the device.


What I like about this workflow is that it relies on installing the pkg via MDM, instead of a traditional policy using the jamf binary.


Time to get started then!

Let's first build a detection for Jamf Protect to be able to pick up when an admin user disables Gatekeeper.


We will use an Analytic to help with the detection.


To disable Gatekeeper, the `spctl` binary can be used, specifically running


```
spctl --master-disable
```


First, the predicate for our Analytic:


```
$event.type == 1 AND 
$event.process.signingInfo.appid == "com.apple.spctl" AND 
$event.process.commandLine CONTAINS[cd] " --master-disable"
```


For the `Event Type` we'll choose `Process Event`.


![](/assets/img/screenshot-2023-02-22-at-12.58.12.png)


To setup the Smart Group integration in Protect and the equivalent Smart Group in Pro we can just follow the docs: https://learn.jamf.com/en-US/bundle/jamf-protect-documentation/page/Setting_Up_Analytic_Remediation_With_Jamf_Pro.html


Cool, so we now have Protect which is detecting when Gatekeeper is disabled and it's adding the device to a Smart Group in Pro.


We now need to build out our AWS API gateway and Lambda, you can follow my previous blog post for the details: https://skartek.dev/2022/08/24/a-beginners-approach-to-serverless-and-python-to-solve-real-life-scenarios/


Once setup it should look like:


![](/assets/img/screenshot-2023-02-22-at-13.03.15.png)


![](/assets/img/screenshot-2023-02-22-at-13.02.40.png)


Remember to choose JSON for the content-type as that would help us easily parse in our Lambda which will be written in python.


Once we have our webhook in place, let's build the actual package that will be installed on the device via MDM.


For the purpose of this exercise I'm keeping it limited to the very basic:
We are going to first check if Gatekeeper have actually been disabled.
We'll then use `jamfHelper` to notify the user they're not allowed to disable Gatekeeper.
We'll then remove the "tag" Jamf Protect made for this device, as we've now re-enabled gatekeeper by clearing up the `/Library/Application\ Support/JamfProtect/groups/Gatekeeper_Disabled_EU`
Finally, we will submit an inventory update to kick the device out of the Smart Group in Jamf Pro as Gatekeeper has now been safely re-enabled.


```
gatekeeper_status=`/usr/sbin/spctl --status | grep "assessments" | cut -c13-`
if [ $gatekeeper_status = "disabled" ]; then
	/usr/sbin/spctl --master-enable
else
	echo "False Detection"
fi


"/Library/Application Support/JAMF/bin/jamfHelper.app/Contents/MacOS/jamfHelper" -windowType hud -title "Oh... you naughty!" -heading "An attempt to disable Gatekeeper have been detected" -alignHeading natural -description "Disabling Gatekeeper is not allowed!." -alignDescription natural -icon "/System/Library/CoreServices/CoreTypes.bundle/Contents/Resources/AlertStopIcon.icns" -button1 Ok -alignCountdown center -lockHUD

/bin/rm -rf /Library/Application\ Support/JamfProtect/groups/Gatekeeper_Disabled_EU

/bin/sleep 60

/usr/local/bin/jamf recon
```


You can use your preferred tool to build the package.


Now, once you've built the package, there's a few things we'll need.


To start, in order to send an `InstallEnterpriseApplication` MDM command, the pkg needs to be signed.
You can follow the superb article from TTG on how to create a signing certificate and how to sign a package: https://travellingtechguy.blog/signing-packages-and-configuration-profiles-with-the-built-in-jamf-pro-certificate-authority/


Once singed, we need the size in bytes as this will need to be used in the manifest which we'll use to send the MDM command.


```
stat -f %z Enable_Gatekeeper.pkg 
11111
```


Lastly, we need to get the MD5 signature of the pkg.


```
md5 Enable_Gatekeeper.pkg
MD5 (Enable_Gatekeeper.pkg) =
11111111111111111111111111111111
```


We now have to choose where to host this package.
Any HTTPS file share would do, but you could also opt to use an S3 bucket.
Create an S3 bucket, upload the pkg and then we'll need to think how to secure it.
You could either leave the S3 bucket open (won't recommend) or you could make use of pre-signed URLs (basically URLs with an auth token backed in and you can choose the validity of the token).


For the purpose of this exercise, I'm using a manually generated pre-signed URL, valid for 12 hours.


![](/assets/img/screenshot-2023-02-22-at-13.16.03.png)


In AWS, S3 choose Object actions and Share with a presigned URL.


Time to get started writing the code of our Lambda.


First of all, we need to thing that Jamf Pro will fire the webhook twice.
The webhook is designed to notify of Smart Group membership changes, so it will be fired when the device is added to the Smart Group and also when the device falls out of the Smart Group.


We can build a simple check in our code to identify if the event is device added (and hence action needed) or if it's being removed (hence remediation has taken place and nothing should be done more.)


```
output = json.loads(event["body"])
 device_removed = output['event']['groupRemovedDevicesIds']
 device_removed_check = str(device_removed)[1:-1]

 if device_removed_check != "":
 print(f"This webhook have been generated for a device being removed from the Smart Group, nothing to do here. Device ID {device_removed}")
 
 else:
 device_id = output['event']['groupAddedDevicesIds']
```


After this, we will call Secrets Manager to retrieve the Jamf Pro API user/password.


```
response = client.get_secret_value(
 SecretId='SecretManager_name_here')
 
 secretDict = json.loads(response['SecretString'])
 api_user = secretDict['username']
 api_password = secretDict['password']
```


We can then proceed and get a bearer token from the Jamf Pro API.


```
token_url = f"{jamf_url}/api/v1/auth/token"
 headers = {"Accept": "application/json"}
 
 resp = requests.post(token_url, auth=(api_user, api_password), headers=headers)
 resp.raise_for_status()
 resp_data = resp.json()
 
 print(f"Access token granted, valid until {resp_data['expires']}.")
 
 data = resp.json()
 token = data["token"]
```


We now need to build the payload, which is actually the manifest.


```
payload = {
 "manifest": {
 "url": presigned_url,
 "hash": "11111111111111111111",
 "hashType": "MD5",
 "bundleId": "com.jamf.gatekeeper.enable",
 "bundleVersion": "1.0.0",
 "subtitle": "Subtitle",
 "title": "EnableGatekeeper.pkg",
 "sizeInBytes": 11111
 },
 "installAsManaged": False,
 "devices": device_id,
 }
 
 headers={
 "Authorization": f"Bearer {token}",
 "Accept": "application/json"
 }
```


The `hash` will be the value we got from the `md5` command previously and the `sizeInBytes` same for the `stat` command.


Lastly, the actual API call:


```
response = requests.post(url, json=payload, headers=headers)
 print(f"Jamf Pro API call HTTP Status: {response.status_code}")
```


The full script can be found on GitHub: 


https://github.com/matteo-bolognini/AWS-Projects/blob/main/Gatekeeper_Disabled_Remediation


That's pretty much it from the configuration and setup side so, time to test this workflow out and see how it looks like:


https://youtu.be/HD99Y8Q3Mho


Well... that's a wrap!


See you on the next episode :)
