---
layout: post
title: "Jamf Pro LAPS"
date: 2023-05-03
slug: jamf-pro-laps
---

With the release of Jamf Pro 10.46, LAPS have been introduced into the product: https://learn.jamf.com/bundle/jamf-pro-release-notes-current/page/New_Features_and_Enhancements.html


Straight from the Admin's guide the TL;DR:


```
LAPS allows you to:
Automatically rotate local administrator passwords

Store local administrator passwords in a secure online location

Audit who has accessed local administrator passwords
```


In this first phase of the launch, LAPS is available via API only.


So I thought would be fun to poke around to learn a bit more about it by making a Slack slash command to interact with.


No fancy diagram this time but here's a brief overview of how it all works together:


- a Slack app will make available a slash command


- the command will hit an AWS API gateway


- API gateway invoke a lambda


- Lambda will retrieve the LAPS credentials from Jamf Pro and store them in Secrets Manager for an easy retrieval


This last bit is where I asked myself it it was the best delivery method possible. We have to keep in mind we're dealing with administrator credentials to our end devices, so we want to store and share them as securely as possible.


An idea could be to integrate a password manager and wire it up to the Lambda via API, but that's maybe for a future post.
For now, we'll use Secret Manager which provides at rest and in transit encryption with Amazon's managed KMS.


To start, we need to enable LAPS.
For this we can use the `/v1/local-admin-password/settings` endpoint.


```
url = f"https://{jamf_url}.jamfcloud.com`/v1/local-admin-password/settings`"
payload = {
 "autoDeployEnabled": true,
 "passwordRotationTime": 3600,
 "autoExpirationTime": 7776000
}

headers = {
 "Authorization": f"Bearer {token}",
 "accept": "application/json",
 "content-type": "application/json"
	 }
	
response = requests.put(url, json=payload, headers=headers)
```


We can then do a GET request against `/v1/local-admin-password/settings` to verify LAPS has been enabled correctly.


We can now move on to the AWS side of things.
I've covered in previous posts how to setup the API gateway + Lambda so we'll skip that part.
The only addition in this case is that we'll need to give our Lambda IAM user permissions not only to read but also to write to Secrets Manager.


The way Slack sends back to your APIs the payload data is... fun... so I did come up with an extremely inelegant solution to parse out some data:


```
#Parse the Slack message content
output = f"""{event}"""
data = ast.literal_eval(output)
body_value = data&#091;'body']
decoded_body = urllib.parse.unquote(body_value)
query_params = urllib.parse.parse_qs(decoded_body)
 
#Parse extract device Serial Number from Slack
serial_from_slack = query_params.get('text', &#091;None])&#091;0]
user_name = query_params&#091;'user_name']
user_name = user_name&#091;0]
user_name = user_name.strip("&#091;]'")

#Parse extract user's details from Slack
user_id = query_params&#091;'user_id']
user_id = user_id&#091;0]
user_id = user_id.strip("&#091;]'")
```


I'm extracting the user who invoked the slash command in Slack so we can then use the Jamf Pro Azure AD API endpoint /`v1/cloud-idp/1002/test-user-membership` to do some access based checks.
In our script we will define an Azure AD group (or multiple) that are allowed to use LAPS. This way we can lock down the use of LAPS to only administrators or whoever is deemed to get access to it.


Unauthorized users will get a message in Slack to inform them they're not allowed to use LAPS:


![](/assets/img/screenshot-2023-05-03-at-14.24.29.png)


A quick video to show how that would look like:


https://youtu.be/Agqbpb1sJGs


When instead the user requesting the LAPS workflow is member of the Azure AD group, and hence authorized:

First we display a message to let the admin know we got the request and are working on it:


![](/assets/img/screenshot_2023-05-03_at_13_46_34.png)


Then the Slack bot will send a message to the user with a link to AWS Secrets Manager that will contain the credentials:


![](/assets/img/screenshot_2023-05-03_at_13_46_22.png)


Here's how it will play together:


https://youtu.be/sTPRBWRK0XA


And that's pretty much it.


The python script for the Lambda is available here: https://github.com/matteo-bolognini/Jamf-Pro/blob/main/scripts/python/laps.py


As a disclaimer, none of this is something production ready and should not be blindly implemented without proper testing.


This is just a first step exploring the capabilities of the newly released LAPS function, I'm curious to see how others would use it and also can't wait for this to make it to the Jamf Pro UI.
