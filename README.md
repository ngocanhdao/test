[TOC]



#Introduction
App360SDK provides easiest way to integrate user management and payment methods (including sms, phone card and e-banking) into your application.

App360 SDK 's Integration Flow
![App360 SDK 's Integration Flow](https://camo.githubusercontent.com/c1f062a8627f539dbb4d1668f712d1c675b17142/687474703a2f2f692e696d6775722e636f6d2f505864436d62332e706e67)


#Getting started with Demo project

Firstly, clone or download this repository to your machine.

git clone https://github.com/app360/app360-wp-sdk.git
or, download from https://github.com/app360/app360-wp-sdk/release

>The App360SDK supports Windows Phone 8 and 8.1 silverlight project.

#Add App360 SDK to your project
There are two ways to add App360SDK to your project.

**Manual**
RightClick to your project -> Add -> Reference choose App360SDK.dll from /app360-wp-sdk/release folder.
Add a new file m360.xml into your project with content below:
```xml
<?xml version="1.0" encoding="utf-8" ?>
<config>
  <channel>mwork</channel>
  <subchannel></subchannel>
</config>
```
![enter image description here](https://lh5.googleusercontent.com/-4nJTQD_u7S0/VP1ZzouNwGI/AAAAAAAAAA8/q3WnOK26noQ/s0/m36xml.png "m36xml.png")

**Using [Nuget](https://www.nuget.org/packages/App360SDK)**
Go to Tools -> Nuget Package Manager -> Package Manager Console

`Install-Package App360SDK -Pre`



# Intergrate your app with App360SDK for windows phone

##Document


 - Getting Started Guide: this README

## Application Id & Secret Key
To using any function of App360SDk, you need to configure *Application Id* and *Secret key* which is used for authorize your app(game) with App360's server.

Go to https://developers.app360.vn/ to create your application.
![enter image description here](https://camo.githubusercontent.com/165a524062d77d5e9205d159f24268cee4c114d5/687474703a2f2f692e696d6775722e636f6d2f34785a386659632e706e67)
**Application ID** and **Secret Key** is available in application details page, Information tab.


#App-scoped ID
##Channeling
Change configure of channel and subchannel in m360.xml

![enter image description here](https://lh3.googleusercontent.com/0v-0aZOiaDpL0ndmXav9WW9J5d1E9VGKtDph_VqH_Q=s0 "m360Config.png")
    


##SDK Initialization
 In your Application constructor, call App360SDK.Initialize() with your application id and the secret key.
```csharp
public App()
{
    // App.xaml initialization
    
	App360SDK.Initialize(<application_id>, <secret>);	
}
```
During initialization, the SDK will try to find and re-opened any cached app-scoped ID session. If the application hasn't opened any session, MOGSessionManager.ActiveSession will return null and the application should open a new session.

##Session initialization
App-scoped ID is logged in via creating a new session for such ID. There are two ways to open a session (both via MOGSessionManager class):

Anonymously

When session is created anonymously and scopedId exists, a new session will be created for that app-scoped ID. If scopedId doesn't exist, a new app-scoped ID with the same scopedId will be created first. If scopedId is null, it will be generated randomly.
```csharp
MOGSession sessionInfo = await MOGSessionManager.OpenActiveSessionWithScopeIdAsync(null, null);
```

>Note that there's no authentication mechanism (password, token, etc) for anonymous login. This login type should be used when the application has already implemented its own authentication mechanism (in this case, scopedId should be set to app's username, for example); or when the application doesn't require the user to login immediately. Anonymously logged-in user could later be linked with Facebook/Google account for better security and portability (e.g. login the same app-scoped ID on different devices).

With Facebook/Google access token
```csharp 
// MOGSession class contains informations about Scoped Session such as id, channel, client id, etc.
MOGSession sessionInfo = await 
// Replace your access token with <AccessToken>
MOGSessionManager.OpenActiveSessionWithSocialIDAsync(SocialDServiceName.FACEBOOK,"<AccessToken>");

```
When session is created via Facebook/Google access token, the SDK will verify the token with Facebook/Google servers then try to find an existing app-scoped ID that is already linked with the corresponding social account. If there's no such ID, a new one will be generated randomly then link with the account and its session is returned. Otherwise, a new session is generated for the existing app-scoped ID. In this way, the authentication mechanism is provided by Facebook/Google.


Success or not?
“sessionInfo.IsSuccess” will return true, if the user log in successfully and otherwise. If there is any error occurs, the detail of error will be return to “sessionInfo.Error”
```csharp
if (sessionInfo.IsSuccess)
{
	var messageBoxResult = MessageBox.Show("Đăng nhập thành công");	
}
else
{                
	MessageBox.Show("Đăng nhập không thành công", "\n -Lỗi \n" + sessionInfo.Error, MessageBoxButton.OK);
}
```



## Linking app-scoped ID with Facebook/Google


To link current app-scoped ID with Facebook/Google
```csharp
\\ replace <facebook_access_token> with Facebook access token of user
MOGScopedUser.CurrentUser.LinkWithFacebookToken("<facebook_access_token>");
```
If done, value of MOGScopedUser.CurrentUser.SocialID will be updated.

#Payment
##Payment flow
In order to secure payment flow, your application might choose to integrate with our SDK on both client-side and server-side, in which case the payment flow is depict in the following diagram:
![enter image description here](https://camo.githubusercontent.com/0ace684f85e4b82195f68c92b6f00e71e33609c7/687474703a2f2f7777772e77656273657175656e63656469616772616d732e636f6d2f6367692d62696e2f63647261773f6c7a3d64476c306247556755474635625756756443427a5a5846315a57356a5a516f4b523246745a53302d5530524c4f67415543584a6c6358566c633351674b444570436c4e455379302d523246745a546f6756484a68626e4e6859335270623234676157517349484e3059585231637977675957317664573530494367794b514247423064686257556763325679646d56794f69427a5a57356b494851414d41746b5958526849475a7663694270626e4e775a51424a4269677a4146384c4144494a41436f4d61575141595167734948567a5a584a66615751674b445141624159415a5163416754384859574e72494368495646525141494561427941794d44417049436731414230504149455944574e76626d5a70636d30674b44594145784d41464173334b5126733d726f7365)

 1. The application (client-side) calls payment API from the SDK, optionally with a custom payload (documented below).    
 2. The SDK returns transaction id and other details (if available) 
 3. The application  client sends transaction data to its server awaiting confirmation
 4. SDK server calls a pre-registered endpoint of the application server
    to notify about transaction status when it completes
    - Note that there is no guarantee about the order of (3) and (4) (i.e. (4) may happen before (3)) 
 5. Application server acknowledges SDK server's call by
    responding with HTTP status code 200.
 6. Application server validates the transaction based on the information it has (transaction ID,
    payload, etc.)
 7. Application server confirms/notifies game client
    about the status of the transaction

Note:

To register your application's server endpoint, go to [https://developers.app360.vn/](https://developers.app360.vn/) set Payment callback endpoint in application details page, Information tab.
Before using any payment methods, the application must first initialize the SDK. See section SDK Initialization above.
There are two methods to integrate payment functionality of App360 SDK into your application:

Using the SDK's existing UI
Implement your own UI and using the SDK's basic payment request classes.
## Using payment classes
 Create a new instance of MOGPaymentSDK
```csharp
MOGPaymentSDK paymentSDK = new MOGPaymentSDK();
```
To make a **banking** charging request
```csharp
// make a banking transaction
MOGBankingTransaction transaction = await paymentSDK.MakeBankingTransaction("100000","payload");
```

To make a **SMS** charging request
```csharp
// make a sms charging transaction
// you can pass more SMS's Amount 
MOGSMSTransaction smsTransaction=await paymentSDK.GetSMSSyntax("payload", MOGPaymentSDK.SMS_AMOUNT.AMOUNT_1000,MOGPaymentSDK.SMS_AMOUNT.AMOUNT_500);
```
To make a **Card** charging request

```csharp
// replace card_type, serial> and <payload> with your value
MOGPhoneCardTransaction phonecardTransaction = await paymentSDK.MakePhoneCardTransaction(MOGCardVendor.VINAPHONE, "<card_code>", "<serial>", "<payload>");
```
## Checking transaction status
To check the status of a transaction
```
\\ checking status by transaction_id
 MOGStatusTransaction status = await paymentSDK.CheckStatusOfTransaction("<transaction_id>");
```
#Release Notes

##Version 1.0.0
**Release Date:** 


# Knows issues
There's no known issues for now
# FAQ
What is a application id and client key?

They are a pair of key, used to authorize your app (game) with SDK's server.

How can i get my application id and client key?

Goto https://developers.app360.vn
Login if you already have an account or register a new one
Open your application in App360 dashboard, select Information tab
All key you need will be there
# Support
Please contact [us](mailto:support@app360.vn) for general inquiries.
#For a technical issue
n case you have a technical issue, you can reach our technical support team. Please provide the following information when you reach out, it'll allow us to help you much more quickly.

 - The steps to reproduce the problem. 
 - If possible, some pieces of code, or even a project.
>For more information, please go to https://developers.app360.vn.
