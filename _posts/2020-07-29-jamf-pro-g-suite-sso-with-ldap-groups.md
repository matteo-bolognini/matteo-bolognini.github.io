---
layout: post
title: "Jamf Pro G Suite SSO with LDAP groups"
date: 2020-07-29
slug: jamf-pro-g-suite-sso-with-ldap-groups
---

Jamf Pro offers a pretty seamless SSO integration with G Suite but when it comes to granting access based on Groups instead of single User accounts there’s a few gotchas that need to be taken into consideration and we’ll look into those in this article.**By default, G Suite is NOT passing user groups membership attributes into the SAML message; this means that no attributes pertaining to the user group’s membership is sent over.


As a prerequisite before we move forward let’s make sure in Jam Pro we have already configured SSO with G Suite as SAML 2.0 Identity Provider as per this KB: [Configuring Single Sign-On with G Suite](https://www.jamf.com/jamf-nation/articles/440/configuring-single-sign-on-with-g-suite)


Let’s start logging in to GSuite account with an Admin user: [https://admin.google.com/my_domain.com/](https://admin.google.com/enziantech.com/)
Then we head to Users > More > Manage Custom Attributes


![](/assets/img/1.png)


and in there we Select > *ADD CUSTOM ATTRIBUTE*  


![](/assets/img/2.png)


Category:** the category in which you would like the custom extension attribute to be listed (in the below example we created a new one)


**Name: **Enter the label you want to display on the user’s account page - in the example we used 


**Info type: **Select **Text**


**Visibility: **Visible to domain 


**No. of values**: Single Value


![](/assets/img/3.png)


Now we need to go to > Apps > SAML Apps


![](/assets/img/4.png)


Then click on the SAML app we created and in


![](/assets/img/5.png)


Here we can click on *Attribute Mapping > ADD NEW MAPPING*


![](/assets/img/6-1.png)


![](/assets/img/7.png)


In the “***Enter the application attribute***” field we can pretty much insert anything we’d like. The important part here is that whatever we provide here will need to be matched exactly in Jamf Pro into the *IDENTITY PROVIDER GROUP ATTRIBUTE NAME* in the SSO configuration (we’ll review this later).**By default Jamf Pro uses an URL like the below
 [http://schemas.xmlsoap.org/claims/Group](http://schemas.xmlsoap.org/claims/Group)
We can either provide this or customize it. I’ll change this and will call my Attribute “IT-jamf-admins”


*Select a category***: we can both select the same category as the one we assigned the custom attribute or choose a different category (like Employee Details for example)


***Select user field***: if we used the Category of the custom attribute we created we should have here our custom name. If we selected “Employee Details” as a category we’ll have here some options.**We could map for example to the Department field


Let’s not forget to *SAVE*** after we’re done here!**A couple of screenshots as example of what could look like this setup


![](/assets/img/8.png)


![](/assets/img/9.png)


Now we're ready to tie up our user and it's group memberhsip to the custom attribute we created.
This is what will make G Suite send this custom Attribute in the SAML message


Let's go to > Users > "my_test_user" > User information


![](/assets/img/10.png)


Once clicked on the User Information we can scroll down to the bottom and there we should see the custom attribute we created before, called Jamf custom attribute** 


![](/assets/img/11.png)


In there we will add the G Suite group that we want to be passed in the SAML message, in my example the LDAP group I want to grant access to Jamf Pro based on is called “IT-administrators”


![](/assets/img/12.png)


Ok, we’re almost done now and we just need to configure Jamf Pro to "read" the values we're passing in the SAML message accordingly to our new setup.


Let's go to Jamf Pro > Settings > System Settings > Single Sign-On and populate the **IDENTITY PROVIDER GROUP ATTRIBUTE NAM**E with the following: IT-administrators


![](/assets/img/13.png)


Next, let’s double-check Jamf Pro can lookup that group**In my example, as I have a Cloud LDAP I’ll go there and test a lookup from Jamf Pro to G Suite for the “IT-administrators” group


![](/assets/img/14.png)


As I didn’t have this group yet imported in Jamf Pro I’ll now go ahead and add it but if you already have LDAP groups in Jamf Pro this step can be skipped
In Jamf Pro > Jamf Pro User Accounts & Groups** > New
Add LDAP Group


![](/assets/img/15.png)


![](/assets/img/16.png)


![](/assets/img/17.png)


And that’s pretty much it!
We're now ready to test the login to Jamf Pro using GSuite as SSO and we will grant access based on it's group membership.


We can go to our Jamf Pro webpage and authenticate now via SSO without the need to import every single user into Jamf Pro.


Bonus: how can we be sure everything is good here? Let’s check into Jamf Pro debug logs!
We can see the full SAML message in the logs and specifically the username that logged in AND now the group that we’re passing from G Suite to Jamf Pro where we see our group passed as


>IT-administrators</saml2:AttributeValue>


![](/assets/img/18.png)
