---
layout: post
title: "A beginners approach to serverless and python to solve real life scenarios."
date: 2022-08-24
slug: a-beginners-approach-to-serverless-and-python-to-solve-real-life-scenarios
---

Lets start with a premise: this post is not about setting up anything even remotely close to be "production-ready".**This is just me, tinkering with AWS and python.


Lets start with why I'm even writing this post.


A few weeks ago I got faced with an interesting dilemma.


Say that you configure Conditional Access for your Jamf Pro instance and enroll via Company Portal your iOS devices.
You then have a Conditional Access policy to require your iOS devices to be on the latest version.
Example: your CA policy has a pass code policy, or a minimum iOS version.


Scenario: you enroll a device, register via Company Portal and shows the end user get a "Your device is non compliant" message.
The user then takes action (in our example say updating the pass-code to be compliant with minimum requirements, or update iOS version).
Once the user then browse again to any O365 resource or Conditional Access gated resources, will still get a "you're not compliant" message.


This is because the device inventory in Jamf Pro haven't been updated, and as consequence, neither the data forwarded from Jamf Pro to Microsoft Endpoint Manager for compliance.


TL;DR
I thought it would be nice to have a way to request an Inventory Update from their iOS devices in a simple and secure way, so enough with the theory, lets move to practice.


From an end user prospective, this will be deployed as a WebClip, via MDM.


Overview of the general setup below:


![](/assets/img/blank-diagram4.png)


Lets break it down:


- User launch the WebClip which makes an HTTP call to AWS API Gateway
- API Gateway invoke a Lambda function
- Lambda grabs the API username and password of a Jamf Pro API account from Secret Manager (data is encrypted at rest and in transit using KMS)
- Lambda send an API call to Jamf Pro to request an UpdateInventory command to be issued for the specific device_id which requested it.
- Jamf Pro receives the API request and send the UpdateInventory command to the device.


AWS Console - Lambda - part1****
Let's start creating a Lambda function:


![](/assets/img/screenshot-2022-08-24-at-12.31.40.png)


Select "*Author from scratch*", give it a friendly name and choose your runtime, in this case I'll choose python as it's the language I'll use to write the API call for Jamf Pro.

For now, we want to leave this Lambda as is, just create it and we'll come back to it later to configure it.


AWS Console - S3 bucket**


This step is optional but I think it adds more value to the end user experience to return an image/GIF when they hit the WebClip, without this step we would only be able to return raw/html formatted text.


Go to S3 section and hit Create Bucket:


![](/assets/img/screenshot-2022-08-24-at-13.41.04.png)


**Give it a friendly name and then for this demo we're ok keeping the rest of the setup to the default one:


![](/assets/img/screenshot-2022-08-24-at-13.41.28.png)


Once the bucket has been created, click on hit and go to Properties and copy the ARN:


![](/assets/img/screenshot-2022-08-24-at-13.42.45.png)


AWS Console - Secret Manager**


The Lambda itself will not be directly exposed to the internet but given it will be invoked by a public API, lets avoid storing username/password in clear text in it.**What we can do, is instead to store them in Secret Manager, where they will be encrypted with KMS.
On execution, Lambda will decrypt those at code level and keep them safe.


Search for Secret Manager and hit *Store a new secret***:


![](/assets/img/screenshot-2022-08-24-at-13.57.51.png)


In the secret type choose Other type of secret:


![](/assets/img/screenshot-2022-08-24-at-13.59.11.png)


Provide the key/value as:


username


password


and the relatively correspondent values, leave encryption to KMS and hit Next:


![](/assets/img/screenshot-2022-08-24-at-14.00.20.png)


Give it a secret name and keep it saved, we will use this later in the Lambda code.**Save the secret hitting *Store Secret***


**AWS Console - IAM****We now need to grant a few permissions in here.
First, we need to give Lambda permissions to read the username/password from Secret Manager.
We also need to allow the Lambda to read the images/gif we're storing on the S3 bucket.
Lastly, we want to make sure our Lambda has permissions to log into CloudWatch.
If when you created the Lambda function you didn't specify an execution role, AWS would have created an IAM role for you and defaulted to grant CloudWatch permissions.


You can check in AWS Console > IAM > Roles you should find an IAM role with a name starting with the name of your Lambda function:


![](/assets/img/screenshot-2022-08-24-at-13.46.48.png)


If you expand the generated policy, it should include CloudWatch permissions:


![](/assets/img/1-2.png)


We can now create 2 new policies to grant permission respectively for S3:


```
{
 "Version": "2012-10-17",
 "Statement": &#091;
 {
 "Effect": "Allow",
 "Action": &#091;
 "s3:GetObject"
 ],
 "Resource": "arn:aws:s3_bucket_here"
 }
 ]
}
```


And for retrieving data from Secret Manager:


```
{
 "Version": "2012-10-17",
 "Statement": &#091;
 {
 "Sid": "AllowLambdafucntionReadSecretManager" 
 "Effect": "Allow",
 "Action": &#091;
 "secretsmanager:GetSecretValue",
 ],
 "Resource": &#091;
 "arn:ARN_of_Secret_Manager_Secret_here"
 ]
 }
 ]
}
```


In both cases replace Resource with the correct ARN of your bucket and secret.

Next we can either create a new IAM role, assign those permissions and run the Lambda with it, or, for simplicity as this is just a demo, I'll just edit the execution IAM role that was automatically created.
Go to the role, click Add permissions and attach the 2 policies we just created.


AWS Console - API Gateway**


We'll build a simple HTTP API Gateway:


![](/assets/img/screenshot-2022-08-24-at-12.31.04.png)


Click on Add integration:


![](/assets/img/screenshot-2022-08-24-at-14.10.53.png)


Select Lambda and pick the Lambda function we've created before:


![](/assets/img/2-2.png)


Set the method type to GET:


![](/assets/img/screenshot-2022-08-24-at-14.12.55.png)


Given this API Gateway will be called via an iOS WebClip, the only HTTP method we have available is GET.


Click on Next on the next steps and complete the API Gateway setup.


Once its done, you should see an invoke ULR similar to this:


![](/assets/img/3-3.png)


Save the invoke URL as we'll be using that in the WebClip.


**AWS Console - Lambda - part2**


It's time now to finally configure the Lambda function.**A side note, we need the requests module which by default is not included in python 3.8 or later shipped with the AWS SKD.
I found this blog post helpful to get *requests*** available: https://medium.com/@cziegler_99189/using-the-requests-library-in-aws-lambda-with-screenshots-fa36c4630d82


We can now copy and paste in the python code for our lambda:


```
import requests
import json
import boto3
import base64
s3 = boto3.client('s3')

def lambda_handler(event, context):

 #Define the Jamf Pro URL
 jamf_url = "https://myinstance.jamfcloud.com"
 
 #Define the S3 bucket name(bucket) and filename(key)
 S3_Bucket = "mybucket"
 S3_Key = "myimage.png"
 SecretManagerId = "mysecret"
 ErrorText = "We hit a snag! Please open a ticket to HelpDesk"
 
 #######################################################################################
 ################################# DO NOT MODIFY BELOW #################################
 #######################################################################################
 
 #Extract the device id from the API call HTTP header
 jss_id = event['queryStringParameters']['device_id']
 
 #Invoke Secrets Manager to obtain Jamf Pro API username and password
 client = boto3.client("secretsmanager")
 response = client.get_secret_value(SecretId = (SecretManagerId))
 secretDict = json.loads(response["SecretString"])
 jamf_api_username = secretDict["username"]
 jamf_api_password = secretDict["password"]
 
 #Invoke Jamf Pro API /token enpoint
 token_url = f"{jamf_url}/api/v1/auth/token"
 headers = {"Accept": "application/json"}
 resp = requests.post(token_url, auth=(jamf_api_username, jamf_api_password), headers=headers)
 resp.raise_for_status()
 resp_data = resp.json()
 print(f"Jamf Pro Access Token granted, valid until {resp_data['expires']}.")
 data = resp.json()
 token = data["token"]
 
 jamf_pro_resp = requests.post(
 f"{jamf_url}/JSSResource/mobiledevicecommands/command/UpdateInventory/id/{jss_id}",
 headers={"Authorization": f"Bearer {token}", "Accept": "application/json"},
 )
 
 if jamf_pro_resp.status_code != 401:
 response = s3.get_object(
 Bucket = (S3_Bucket),
 Key = (S3_Key),
 )
 image = response['Body'].read()
 return {
 'headers': { "Content-Type": "image/png" },
 'statusCode': 200,
 'body': base64.b64encode(image).decode('utf-8'),
 'isBase64Encoded': True
 
 }
 print(f"Inventory Update successfully requested - HTTP response: {jamf_pro_resp.status_code}")
 else:
 return {
 'headers': { "Content-type": "text/html" },
 'statusCode': 200,
 'body': "<h1>(ErrorText)</h1>",
 
 } 
 print(f"Check your credentials <POST failed> Jamf Pro API HTTP response: {jamf_pro_resp.status_code}")

```


Edit the script line 9 to 17 and provide the details of your environment.


At this point, we're done with the AWS config, all what's left is to create and deploy the WebClip via Jamf Pro.


Create a Configuration Profile for Mobile Devices.


Within the URL field we will provide the API Gateway URL constructed this way:


```
https:&#047;&#047;your_unique_identifier.execute-api.eu-central-1.amazonaws.com/yourAPIGatewayName?device_id=$JSSID
```


We are adding the 


```
?device_id=$JSSID
```


so that when the Configuration Profile is deployed, Jamf Pro will insert each device_id into the URL.
The device_id is then fetch in Lambda.


The Config Profile should then look like this:


![](/assets/img/10-4.png)


Scope as needed and Save.


Now let's wrap it up and see if it works:


https://youtu.be/l_By0cFaMrI


Not the most elegant notification in the WebClip but for this demo would suffice.
We can see the Update Inventory command being issued successfully.


Well, that's a wrap!
