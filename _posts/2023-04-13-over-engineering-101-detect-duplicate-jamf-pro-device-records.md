---
layout: post
title: "Over-Engineering 101: detect duplicate Jamf Pro device records."
date: 2023-04-13
slug: over-engineering-101-detect-duplicate-jamf-pro-device-records
---

A premise as usual, this work is mostly my "lab" to learn python and serverless and try to apply workflows to real use-case scenario.


The "over-engineering" post title is actually a comment my colleague Allen made when I showed him this workflow :) 
So I thought that'd be a great title to show what I've been working on.


To start with the usual: what problem are we trying to solve.


When Mac are sent to repair, it is not uncommon for those to get some hardware replace, causing them to come back with the same Serial Number but a different UDID.
Those devices are usually wiped before being sent to repair, so they will need to be re-enrolled in Jamf Pro.
If the previous device record haven't been deleted, Jamf Pro will create a new record for this device enrolling as it's de-facto a "new" device given it has a new UDID, but it has also the same Serial Number as the old record... and this can lead to some issue.


Lets get right into it with a graph to show what's the idea to overcome this issue (and to understand the "over-engineering" bit):


![](/assets/img/blank-diagram.png)


Breaking it down it looks more or less like:


- Device come back from repair and is enrolled in Jamf Pro.


- A webhook on `DeviceAdded` trigger is fired off by Jamf Pro to AWS api gateway which then post the json content of the webhook to a Lambda.


- Lambda will process the webhook, check if a duplicate record of this device exist and if yes, will post a message to Slack, giving the details of the 2 device records and an option to delete the old one


- When an admin pick the "delete" button in Slack, a payload is sent to another AWS api gateway which invokes another Lambda which will parse the admin's choice


- An API call is made to Jamf Pro to delete the device record.


Lets analyze the first Lambda, which is written in Python.
On the webhook we get from Jamf Pro, we will have the SerialNumber and UDID of the device that has just enrolled.
We need then to check in Jamf Pro if another record exist with that same SerialNumber.
The Classic API (CAPI) is designed to return only 1 results, but here come to rescue the Jamf Pro API (JPAPI) which offers a nice RSQL filter which will return multiple results (if exist) for the given SerialNumber:


```
https:&#047;&#047;{jamf_url}.jamfcloud.com/api/v1/computers-inventory?section=HARDWARE&page=0&page-size=100&filter=hardware.serialNumber%3D%3D%22{serial_number}%22%22
```


Once we have data back from this endpoint, we can easily grab the JSS_ID (device identifier) and UDID out of the json blob:


```
id_multiple = list(map(int, id_multiple))
 id_multiple.sort()
 
 id1 = id_multiple&#091;0]
 id2 = id_multiple&#091;1]
 
 udid_multiple = &#091;]
 for result in output&#091;"results"]:
 udid = result&#091;"udid"]
 udid_multiple.append(udid)
 print(f"UDIDs : {udid_multiple}")
 
 udid_multiple.sort()
 
 udid1 = udid_multiple&#091;0]
 udid2 = udid_multiple&#091;1]
```


And that's pretty much if for the first function, we will just use `slack.com/api/chat.postMessage` to post the message into Slack, which will look like this:


![](/assets/img/screenshot-2023-04-13-at-16.14.51.png)


The next part of the job is to actually capture the user's input from Slack.


This is done with a pretty simple HTTP response where Slack will post a payload when a user will click any menu item button, and will be sent to our api gateway which will then pass the json blob to the Lambda to be worked on.
I have to admit I struggled a bit with Slack formatting of the payload and came up with the (incredibly in-elegant) code below to parse:


```
request_body = event&#091;"body"]
request_body_parsed = request_body.replace('payload=','', 1)
request_body_parsed = urllib.parse.unquote(request_body_parsed)
request_body_parsed = json.loads(request_body_parsed)
 
device_id = request_body_parsed&#091;"actions"]&#091;0]&#091;"value"]
serial_number = request_body_parsed&#091;"actions"]&#091;0]&#091;"name"]
udid = request_body_parsed&#091;"callback_id"]
username = request_body_parsed&#091;"user"]&#091;"name"]
```


So we now have the device SerialNumber, the "original" ID and the UDID, we can go ahead and delete.


I thought it would be nice to add some little validation of which user can actually delete the device record.
Given it's a no-going-back operation, we might want to restrict that to specific users only, even if you're posting those alerts in private Slack channels.


JPAPI come to rescue again as we can leverage the Cloud IdP integration (AzureAD, Google LDAP) to actually check if the user who clicked the button is member of an LDAP group or not:


```
url = f"https://{jamf_url}.jamfcloud.com/api/v1/cloud-idp/1001/test-user-membership"
 payload = {
 "username": f"{username}",
 "groupname": f"{control_group}"
 }
```


The `control_group` is a variable we can define in our script which will be an actual LDAP group we will check the user's membership against.


If the user is not member of the group, they will get a message advising them they're not allowed to perform the delete operation from Slack:


```
return {
 'statusCode': 200,
 'body': (f"Account:{username} attempted to delete the device record:{device_id} but is not authorized due to lack of permissions, contact your Jamf Pro administrator.")
 }
```


This is how it looks like in Slack:


![](/assets/img/screenshot-2023-04-13-at-16.49.25.png)


Otherwise, if the user is member of the LDAP group, they will be allowed to proceed with the deletion and a message in Slack will confirm the success of the operation:


```
url = f"https://{jamf_url}.jamfcloud.com/JSSResource/computers/id/{device_id}"
 headers = {"Authorization": f"Bearer {token}", "Accept": "application/json"}
 response = requests.delete(url, headers=headers)
 print(f"Status Code: {response.status_code}")
 print("Old device record successfully deleted from Jamf Pro")
 
 return {
 'statusCode': 200,
 'body': (f"A device record has been deleted via Slack integration, details as following:\n\nDevice Record ID {device_id}\n\nDuplicate Serial Number: {serial_number}\nUDID:{udid}\n\nDeleted from Jamf Pro by user: {username}")
 }
 
```


This is how it looks like in Slack:


![](/assets/img/screenshot-2023-04-13-at-16.15.04.png)


Time to see a demo:


https://www.youtube.com/embed/7nMBHHW7jcs


The python scripts for the lambdas are available on my GitHub:

https://github.com/matteo-bolognini/AWS-Projects/blob/main/webhook-processor.py
https://github.com/matteo-bolognini/AWS-Projects/blob/main/slack_responder.py


That's a wrap!
Hope you enjoyed this journey into over-engineering simple tasks :)
