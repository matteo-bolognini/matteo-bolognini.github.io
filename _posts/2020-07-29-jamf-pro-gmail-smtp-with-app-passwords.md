---
layout: post
title: "Jamf Pro GMail SMTP with App Passwords"
date: 2020-07-29
slug: jamf-pro-gmail-smtp-with-app-passwords
---

When using a GMail account as SMTP with Jamf Pro, we need to enable the "Less Secure Apps" configuration on Google side.
We can check this in our Google Account > Security > and verify if "Allow less secure apps" is turned ON
Google give us a tool we can use to specify a password for each application we want to use our GMail account to have access to. 
For example we can have multiple SMTP servers using the same GMail account and each of them have a "dedicated password.
So what is this Apps passwords?
"App passwords let you sign in to apps and devices that don't support 2-Step Verification", or OAuth2.0 we can add


Let's first check in our Google Account > Security > that Allow less secure apps is turned ON


![](/assets/img/1-1.png)


Next steps we will turn on 2FA following: [https://support.google.com/accounts/answer/185839?hl=en](https://support.google.com/accounts/answer/185839?hl=en)


![](/assets/img/2-1.png)


![](/assets/img/3-1.png)


![](/assets/img/4-1.png)


Once this is done we can go back into Security and create an App password following: [https://support.google.com/accounts/answer/185833?hl=en](https://support.google.com/accounts/answer/185833?hl=en)


![](/assets/img/screenshot-2020-07-29-at-17.54.15.png)


![](/assets/img/screenshot-2020-07-29-at-17.55.15.png)


We will not get a 16 characters random generated password, let's take note of the generated App Password as we won't be able to recover it later


![](/assets/img/screenshot-2020-07-29-at-17.55.26.png)


Once this is done we can go to Jamf Pro > SMTP Server and replace our current password with the randomized unique password we just obtained


![](/assets/img/screenshot-2020-07-29-at-18.13.49.png)
