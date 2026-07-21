---
layout: post
title: "Jamf Now - password sync Preview"
date: 2021-12-07
slug: jamf-now-password-sync-preview
---

On 6 December 2021 Jamf has released a Preview of a new feature in Jamf Now that will enable and facilitate password sync on macOS, leveraging Jamf Connect.


First of all, where do we start from? Of course the documentation!**https://docs.jamf.com/jamf-now/documentation/Jamf_Connect_Deployment.html


Before we dig into, I'd like to stress this feature is currently on Preview and as per the documentation it states:
"*Previews give you a first look at upcoming features and functionality, and allow you to provide feedback and submit defects to our software developers. Preview features and documentation are provided for testing purposes and should not be considered final."*


This feature is currently supported for Okta and AzureAD. Similar to the "complete suite" that is Jamf Connect, this new iteration on Jamf Now allows to synchronize the local account password on a Mac with the Okta/AzureAD password so both of them are in sync and, most importantly, if there is a mismatch of the local account password, prompt the user to change the local password so it matches the IdP password and get FileVault password updates as well.


Ok time to start with the setup. In this example I will use Okta but pretty much the same steps applies for AzureAD ref: https://docs.jamf.com/jamf-connect/2.7.0/documentation/Integrating_with_Microsoft_Azure_AD.html#ID-00001cea).


First of all, again documentation on how to create the Okta app:
https://docs.jamf.com/jamf-connect/2.7.0/documentation/Integrating_with_Okta.html

Once logged in the Admin console, we'll head over to 


- Applications** > **Create App Integration**


In the next screen we're gonna select:


2. **OIDC - OpenID Connect** as the sign-in method.**3.Native Application** as the application type.


![](/assets/img/screenshot-2021-12-07-at-14.34.42.png)


4. Give the app a name in the Application name field.


5. Set **Authorization Code** to **Implicit (hybrid)** in the **Grant type** section


![](/assets/img/screenshot-2021-12-07-at-14.36.55.png)


6. For this example, we will assing this app to ALL users within the org


![](/assets/img/screenshot-2021-12-07-at-14.40.22.png)


7. Most important step of all, click **Save**.


We now have configured our Okta app, so it's time to head over to Jamf Now.


Within our Jamf Now Blueprint lets head to the **Security** section and enable the **Enable Jamf Connect** checkbox.


![](/assets/img/screenshot-2021-12-07-at-14.43.39.png)


This will bring up the configuration menu. Lets switch to Okta in the **Identity** **Provider** field and provide our Okta tenant (domain)


![](/assets/img/screenshot-2021-12-07-at-14.42.37-1.png)


There is a built in webui to test the configuration. Most likely you will get an error like


![](/assets/img/screenshot-2021-12-07-at-14.47.34.png)


This is just expected as, by default, Okta is not allowed to be embedded in an iFrame.**I don't recommend this but you can change this setting by logging as admin and going to Settings** -> **Customization** -> **IFrame** **Embedding** and checking “**Allow** **IFrame** **embedding**”


![](/assets/img/screenshot-2021-12-07-at-14.50.31.png)


Once this is done, if you revisit the Blueprint and click again on **Test**, you'll notice the Okta UI login should now be displayed.


Once we're at this point, the configuration is pretty much completed we can save and wait for the Blueprint to sync on the device.

Digging into a bit we'll find 2 components will be installed on the device:
1. A Configuration Profile, containing the Okta setup and config


2. An application in /Applications. This is the actual app used to keep password in sync.
On this first iteration of the Preview, the app won't auto-launch and needs to by launched manually by the user


![](/assets/img/screenshot-2021-12-07-at-15.00.57-1.png)


Finally, lets have a look at how it all works.
In this short video we'll see
- a user opening the Jamf Connect application
- getting a prompt to provide the Okta password
- a second prompt to provide the local account password
This is the point where Connects kicks in, identify the user's Okta password is not matching the local account password and proceed to change the local account password to match the Okta one and update the FileVault password as well.


https://youtu.be/sgZ4pApj3dw


The Connect app also gives the user ability to change/reset their Okta password via the menu bar app. This ensures that when changing the Okta password, the local account and FV passwords are all kept in sync as well.


Hope it helps :)
