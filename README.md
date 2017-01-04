# amazonDRS

# Getting Started
This repo is intended to serve as an Arduino library aimed at getting started with Amazon's [Dash Replenishment Service](https://developer.amazon.com/dash-replenishment-service)! Amazon currently has some great [DRS documentation](https://developer.amazon.com/public/solutions/devices/dash-replenishment-service/) that I'll reference in this guide although currently offers nothing directed towards Arduino and the DIY hacker community. This guide is meant to supplement Amazon's documentation with a focus on implementing the API on [WiFi101](https://www.arduino.cc/en/Reference/WiFi101) enabled Arduinos! This includes the [ArduinoMKR1000](https://www.arduino.cc/en/Main/ArduinoMKR1000), and Arduino sporting the [WiFi101 shield](https://store-usa.arduino.cc/products/asx00001), an [Adafruit MO WiFi Feather](https://www.adafruit.com/product/3010), or really any Arduino with sufficient space, and Atmel's [ATWINC1500 WiFi module](http://www.atmel.com/images/atmel-42376-smartconnect-winc1500-mr210pa_datasheet.pdf).  Ports to other WiFi modules and libraries are possible but this seemed to be popular module to start with!

By the end of this guide you should have a functioning Dash Button similar to Amazon's [AWS IoT Button](https://aws.amazon.com/iotbutton/), with one small difference! Although the first example [amazonDashButton](https://github.com/andium/AmazonDRS/tree/master/examples/amazonDashButton) allows you to initiate purchases on amazon using a pushbutton, you now have the tools necessary to leverage the power of Arduino to initate these frictionless purchases with actuations beyond the motion of a simple button push! 

This guide is not meant to be a complete implementation of the Dash API and is by no means the most effecient or cost effective way to incorporate frictionless purchasing into your product. That being said, if you're looking to prototype and hack together your own dash button or frictionless purchase proof of concept, you've come to the right place!

# Initial Setup

## SNS Simple Notification Service
To start you're going to need some amazon accounts! It probably goes without saying that you'll need an amazon account to authorize your arduino to make purchaes on your behalf, but you'll also need to sign up for a free tier of AWS. 

* [Create a developer account with amazon web services](https://aws.amazon.com/free/?sc_channel=PS&sc_campaign=acquisition_US&sc_publisher=google&sc_medium=cloud_computing_b&sc_content=aws_account_bmm_control_q32016&sc_detail=%2Baws%20%2Baccount&sc_category=cloud_computing&sc_segment=102882724242&sc_matchtype=b&sc_country=US&s_kwcid=AL!4422!3!102882724242!b!!g!!%2Baws%20%2Baccount&ef_id=WGw8xgAABK9kcEZO:20170104000734:s). With your AWS account activated you'll be creating an SNS [Simple Notifications Service](https://aws.amazon.com/documentation/sns/) to handle the messaging for your device. This typically includes e-mails that are sent on behalf of your device when products or replenished, or if you were to authorize/deauthrize the device.

* Amazon has some great step-by-step instructions with screenshots on setting up the SNS Topic for Dash Replenishment! You can find those [here](https://developer.amazon.com/public/solutions/devices/dash-replenishment-service/docs/dash-create-an-sns-topic). Follow along and once you create the SNS topic you can pretty much forget about it. You won't touch it again during the dev process. 

##LWA Login With Amazon
Next you're going to need to create a [login with amazon](https://developer.amazon.com/login-with-amazon) security profile. 

* [Creating the security profile](https://developer.amazon.com/lwa/sp/create-security-profile.html) will allow you to authorize your Arduino to make purchases by doing just that, 'Logging in with Amazon!' Typically device manufacturers would inlcude this step as a flow in their product registration website or application. Since we're going for a bare bones proof of concept we're going to keep this as simple as possible. Fill out the required fields...

 * **Security Profile Name:**	Pick any name, this is the name you'll see when the login with amazon screen pops up
 * **Security Profile Description:**	anything goes
 * **Consent Privacy Notice URL:**  http://www.andium.com/thisdoesntexistyet (any url doesn't have to be valid yet)
 * **Consent Logo Image:** choose an img/not totally necessary, this appears as the thumbnail represeting your product in 'Your Account >> Manage Login With Amazon' where you can deauthorize your device and "un-tie" it from your amazon account.

* Click save and you're ready to move on! Your LWA security profile is created, time to save some important info for later.

* Click show Client ID and Client Secret. Save these details or keep the tab open, you're going to hard code them into your [AmazonTokens.h](https://github.com/andium/AmazonDRS/blob/master/src/AmazonTokens.h) file. More on this later when we get to setting up the Arduino. They should look something like this...
```
Client ID: amzn1.application-oa2-client.c0a2100c1af289b1bf4011c71f6029b3
Client Secret: ec48483c54c34a23a71aa8ccb2742902f7d3d00c2dd78fc5ac404eef0111e485
```

##Create a Dash Replenishment Device
Now it's time to create what amazon refers to as your Dash Replenishment Device. This [device creation wizard](https://developer.amazon.com/dash-replenishment/create_device.html) makes this process really simple. Essentially you're creating the infrastructure that will maintin what products you'd like to make available to your device. 
* First step is to create the Device Name, Model ID, and upload a transparent png img (<1MB)
* Next you'll create "Slots" for your device
 * It seems Slots are intended to represent discrete product types although you don't need to maintain this convention. For example a coffee maker may utilize Slot 1 for replacement coffee flavors, and Slot 2 for various coffee filters. When the user authorizes the device for replenishment they will choose one product from each slot to be eligible for replenishment.
  * In each slot you can add any number of ASINs. ASINs are unique product identifiers used by Amazon to identify a product. You can find the ASIN within the product details of an item on Amazon. For example these batteries [https://www.amazon.com/gp/product/B00MNV8E0C/ref=s9_dcacsd_bhz_bw_c_x_1_w](https://www.amazon.com/gp/product/B00MNV8E0C/ref=s9_dcacsd_bhz_bw_c_x_1_w) have an ASIN of B00MNV8EOC. Another thing to keep in mind is that not all products available on Amazon are eligible for dash replenishment. Currently only items that are "Shipped and Sold by Amazon" are eligible for DRS. 
 
* Upon completing these steps you have completed creating your device! Take note of your devices Model ID as well as the associated Slot IDs. We'll need the Model ID when we create our login with amazon link.

* It's also worth noting that you can create multiple devices for testing and utilize the same LWA profile for authorizing the device.

##Login With Amazon [Consent Request](https://developer.amazon.com/public/solutions/devices/dash-replenishment-service/docs/dash-lwa-web-api)/Authorization Code Grant
Phewf! At this point we have created everything we need from Amazon to get started! Now it's time to start hacking this together! Before we can start running some code on the Arduino we need to authorize the device to make purchases on behalf of our amazon account. Don't worry there is a [test flag](https://developer.amazon.com/public/solutions/devices/dash-replenishment-service/docs/dash-test-device-purchases) that we're going to set so we won't actually be ordering anything. (Even though we'll still get notifications that a purchase was made).

* It's time to construct the url that will bring us to the consent request screens! Here are the details we'll need...

 * **client_id:** amzn1.application-oa2-client.c0a2100c1af289b1bf4011c71f6029b3 (from the LWA Security Profile)
 * **scope:** dash:replenish. (DRS always uses this scope)
 * **scope_data:** ```{"dash:replenish":{"device_model":"Arduino_dash","serial":"arduino123", "is_test_device":"true"}}```
  * **device_model:** Arduino_dash (Model ID from the "Create Dash Replenishment Step")
  * **serial:** arduino123 (Make this up! It's supposed to uniquely identify your hardware.)
 * **response_type:** code (Code, because we're asking for an authorization_code which we'll use to gain tokens for API access)
 * **redirect_uri:** https://www.getpostman.com/oauth2/callback (This would be the uri of your customer facing app, but we're skipping the whole app part so we're just going to use this from [Postman](https://www.getpostman.com/))
 
 Before it's encoded, here's what an example request url to https://www.amazon.com/ap/oa should look like...
```
https://www.amazon.com/ap/oa?client_id=amzn1.application-oa2-client.c0a2100c1af289b1bf4011c71f6029b3&scope=
dash:replenish&scope_data={"dash:replenish":{"device_model":"Arduino_dash","serial":"arduino123", "is_test_device":"true"}}&response_type=code&redirect_uri=https://www.getpostman.com/oauth2/callback
```
Now with URI encoded fields
```
https://www.amazon.com/ap/oa?client_id=amzn1.application-oa2-client.c0a2100c1af289b1bf4011c71f6029b3&scope=dash%3Areplenish&scope_data=%7B%22dash%3Areplenish%22%3A%7B%22device_model%22%3A%22Arduino_dash%22%2C%22serial%22%3A%22arduino123%22%2C%20%22is_test_device%22%3A%22true%22%7D%7Dtest&response_type=code&redirect_uri=https%3A%2F%2Fwww.getpostman.com%2Foauth2%2Fcallback
```
You'll notice that not all field values change when they are URI encoded, only the scope and redirect_uri actually change. When composing your request url you can use this handy [encoder/decoder](http://meyerweb.com/eric/tools/dencoder/) to quickly encode your values.

If you've composed everything correctly paste the encoded url into the browser and it should bring you to an Amazon login page where you'll log in and authorize your device for replenishment! Notice your LWA Security Profile name displayed on the top. Follow the prompts and select which product you'd like to assign to each slot. On the last screen you confirm your shipping and billing information and click 'complete setup'! Keep an eye on the url bar at the top of the browser, you should get a response back that looks simillar to this...

```
https://app.getpostman.com/oauth2/callback?code=ANdNAVhyhqirUelHGEHA&scope=dash%3Areplenish
```
Now you've got your authorization_code! Jot down the string following code= "ANdNAVhyhqirUelHGEHA". Time to move on to the Arduino and use this authorization code to gain access and refresh tokens to access the Dash API.
 
## authCodeGrant
 
 
 
 
 
 
 
 
 
 
 
 


