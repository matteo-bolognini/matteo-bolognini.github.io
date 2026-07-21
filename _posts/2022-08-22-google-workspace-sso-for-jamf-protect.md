---
layout: post
title: "Google Workspace SSO for Jamf Protect"
date: 2022-08-22
slug: google-workspace-sso-for-jamf-protect
---

This will be a quick post to help navigate Google Workspace to setup a SAML app to be used with Jamf Protect to enable SSO on the tenant.


First, let's login to [https://admin.google.com ](https://admin.google.com)and expand the ***Apps*** section and locate and open ***Web and mobile apps***:


![](/assets/img/1-1.png)


**Next we want to select *Add app*** and then ***Add a custom SAML app***:


![](/assets/img/2-1.png)


**We can now start configuring our SAML app.
Lets give it an App name, that would be the display name of the app, eventually a description to help keep track of what this SAML app is linked to, and an icon (optional):


![](/assets/img/3-2.png)


We can then hit Continue and move to the next page.


In here, we will download the metadata file.
This is a unique per SAML app file, which contains the information of this app.
You want to save this file somewhere safe as it is the one you will need to send Jamf to enable SSO for the Protect tenant:


![](/assets/img/4-1.png)


The next step after we move on clicking Continue, is to configure the ACS URL and the Entity ID.


For those, we can refer Jamf's documentation:


[https://docs.jamf.com/jamf-protect/documentation/Integrating_with_Google_Cloud.html](https://docs.jamf.com/jamf-protect/documentation/Integrating_with_Google_Cloud.html)


For example, my tenant is https://matteoistesting.protect.jamfcloud.com/
So the <ConnectorName> to replace for both ACS URL and Entity ID will be:


matteoistesting-protect-jamfcloud-com


Note the dashes in here, not the dots.


This test tenant I'm using is located in Europe so my ACS URL will be
EU—`https://eu-auth.protect.jamfcloud.com/login/callback?connection=matteoistesting-protect-jamfcloud-com`


and my Entity ID:


`urn:auth0:eu-jamf:matteoistesting-protect-jamfcloud-com`


![](/assets/img/7-2.png)


Once we've configured ACS URL and Entity ID accordingly to our tenant, we want to setup the` `*Name ID format*** and set that to ***EMAIL***:


![](/assets/img/8-3.png)


Lets hit continue and move to the last step of configuring the SAML app. We will leave the default configuration here so we can click on ***Finish*** to have Google Workspace build the SAML app:


![](/assets/img/9-2.png)


Once the SAML app have been successfully crated, note that by default, it will not be enabled, see the ***OFF for everyone***:


![](/assets/img/10-3.png)


Lets click on it and enable it by setting it to ***ON for everyone***:


![](/assets/img/11-2.png)


The very last step now is to assign user groups to our SAML app:


![](/assets/img/12-3.png)


**This is a very important step as, assigning those groups, will translate in granting access to Jamf Protect only to Google Workspace users that are member of such groups.


Once we've finished, we can reach out to Jamf and provide:


- Google Workspace domain
- IdP metadata file


Once Jamf will confirm the SSO has been setup for our tenant, we should see a new *Continue with Google*** button to show up under the /login URL of our tenant:


![](/assets/img/screenshot-2022-08-22-at-14.55.40-2.png)


And that's pretty much it.
We can click on it and we'll be redirected to Google Workspace for authentication.

Keep in mind, by default Jamf disables IdP initiated logins, for good reasons: https://auth0.com/docs/authenticate/protocols/saml/saml-sso-integrations/identity-provider-initiated-single-sign-on

This means we can't launch Protect console from our Google Workspace profile.
If you're ok with having IdP initiated login enabled, just let Jamf know and they will enable it for you.
