
# homebridge-http-switch Plugin

`homebridge-http-switch` is a [Homebridge](https://github.com/nfarina/homebridge) plugin with which you can configure 
HomeKit switches which forward any requests to a defined http server. This comes in handy when you already have home 
automated equipment which can be controlled via http requests. Or you have built your own equipment, for example some sort 
of lightning controlled with an wifi enabled Arduino board which than can be integrated via this plugin into Homebridge.

`homebridge-http-switch` supports three different type of switches. A normal `stateful` switch and two variants of 
_stateless_ switches (`stateless` and `stateless-reverse`) which differ in their original position. For stateless switches 
you can specify multiple urls to be targeted when the switch is turned On/Off.   
More about on how to configure such switches can be read further down.

## Installation

First of all you need to have [Homebridge](https://github.com/nfarina/homebridge) installed. Refer to the repo for 
instructions.  
Then run the following command to install `homebridge-http-switch`

```
sudo npm install -g homebridge-http-switch
```

## Updating the switch state in HomeKit

The _'On'_ characteristic from the _'switch'_ service has the permission to `notify` the HomeKit controller of state 
changes. `homebridge-http-switch` supports two ways to send state changes to HomeKit.

#### The 'pull' way:

The 'pull' way is probably the easiest to set up and supported in every scenario. `homebridge-http-switch` requests the 
state of the switch in an specified interval (pulling) and sends the value to HomeKit.  
Look for `pullInterval` in the list of configuration options if you want to configure it.

#### The 'push' way:

When using the 'push' concept the http device itself sends the updated value itself to `homebridge-http-switch` whenever 
the value changes. This is more efficient as the new value is updated instantly and `homebridge-http-switch` does not 
need to make needless requests when the value didn't actually change. However because the http device needs to actively 
notify the `homebridge-http-switch` plugin there is more work needed to implement this method into your http device.  
How to implement the protocol into your http device can be read in the chapter [**Notification Server**](#notification-server)

## Configuration:

The configuration can contain the following properties:

#### Basic configuration options:

- `name` \<string\> **required**: Defines the name which is later displayed in HomeKit
- `switchType` \<string\> **optional** \(Default: **"stateful"**\): Defines the type of the switch:
    * **"stateful"**: A normal switch and thus the default value.
    * **"stateless"**: A stateless switch remains in only one state. If you switch it to on, it immediately goes back to off. 
    Configuration example is further [down](#stateless-switch).
    * **"stateless-reverse"**: Default position is ON. If you switch it to off, it immediately goes back to on. 
    Configuration example is further [down](#reverse-stateless-switch).
    * **"toggle"**: The toggle switch is a stateful switch however does not use the `statusUrl` to determine the current 
    state. It uses the last set state as the current state. Default position is OFF.
    * **"toggle-reverse""**: Same as **"toggle"** but switch default position is ON.

* `onUrl` \<string | \[string\] | [urlObject](#urlobject) | [[urlObject](#urlobject)]\> **required**: Defines the url 
(and other properties when using an urlObject) which is called when you turn on the switch.
* `offUrl` \<string | \[string\] | [urlObject](#urlobject) | [[urlObject](#urlobject)]\> **required**: Defines the url 
(and other properties when using an urlObject) which is called when you turn off the switch.
* `statusUrl` \<string | [urlObject](#urlobject)\> **required**: Defines the url 
(and other properties when using an urlObject) to query the current state from the switch. By default it expects the http 
server to return **'1'** for ON and **'0'** for OFF leaving out any html markup.  
You can change this using `statusPattern` option.  

#### Advanced configuration options:

- `statusPattern` \<string\> **optional** \(Default: **"1"**\): Defines a regex pattern which is compared to the body of the `statusUrl`.
When matching the status of the switch is set to ON otherwise OFF. [Some examples](#examples-for-custom-statuspatterns).
- `auth` \<object\> **optional**: If your http server uses basic authentication you can specify your credential in this 
object. It uses those credentials for all http requests and thus overrides all possibly specified credentials inside 
an urlObject for `onUrl`, `offUrl` and `statusUrl`.  
The object must contain the following properties:
    * `username` \<string\>
    * `password` \<string\>
- `httpMethod` _**deprecated**_ \<string\> **optional**: If defined it sets the http method for `onUrl` and `offUrl`. 
This property is deprecated and only present for backwards compatibility. It is recommended to use an 
[[urlObject](#urlobject)] to set the http method per url.

* `timeout` \<integer\> **optional** \(Default: **1000**\): When using a stateless switch this timeout in 
**milliseconds** specifies the time after which the switch is reset back to its original state.
* `pullInterval` \<integer\> **optional**: The property expects an interval in **milliseconds** in which the plugin 
pulls updates from your http device. For more information read [pulling updates](#the-pull-way).  
(This option is only supported when `switchType` is **"stateful"**)

- `multipleUrlExecutionStrategy` \<string\> **optional** \(Default: **"parallel"**\): Defines the strategy used when 
executing multiple urls. The following are available:
    - **"parallel"**: All urls are executed in parallel. No particular order is guaranteed. Execution as fast as possible.
    - **"series"**: All urls are executed in the given order. Each url must complete first before the next one is executed.  
    When using series execution you can also have a look at the [delay url](#the-delay-url).

* `debug` \<boolean\> **optional**: If set to true debug mode is enabled and the plugin prints more detailed information.

Below are two example configurations. One is using simple string urls and the other is using simple urlObjects.  
Both configs can be used for a basic plugin configuration.

```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Switch",
          
          "switchType": "stateful",
          
          "onUrl": "http://localhost/api/switchOn",
          "offUrl": "http://localhost/api/switchOff",
          
          "statusUrl": "http://localhost/api/switchStatus"
        }   
    ]
}
```

```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Switch",
          
          "switchType": "stateful",
          
          "onUrl": {
            "url": "http://localhost/api/switchOn",
            "method": "GET"
          },
          "offUrl": {
            "url": "http://localhost/api/switchOff",
            "method": "GET"
          },
          
          "statusUrl": {
            "url": "http://localhost/api/switchStatus",
            "method": "GET"
          }
        }   
    ]
}
```

#### UrlObject

A urlObject can have the following properties:
* `url` \<string\> **required**: Defines the url pointing to your http server
* `method` \<string\> **optional** \(Default: **"GET"**\): Defines the http method used to make the http request
* `body` \<string\> **optional**: Defines the body sent with the http request
* `strictSSL` \<boolean\> **optional** \(Default: **false**\): If enabled the SSL certificate used must be valid and 
the whole certificate chain must be trusted. The default is false because most people will work with self signed 
certificates in their homes and their devices are already authorized since being in their networks.
* `auth` \<object\> **optional**: If your http server uses basic authentication you can specify your credential in this 
object. When defined the object must contain the following properties:
    * `username` \<string\>
    * `password` \<string\>
* `headers` \<object\> **optional**: Using this object you can define any http headers which are sent with the http 
request. The object must contain only string key value pairs.  
  
Below is an example of an urlObject containing all properties:
```json
{
  "url": "http://example.com:8080",
  "method": "GET",
  "body": "exampleBody",
  
  "strictSSL": false,
  
  "auth": {
    "username": "yourUsername",
    "password": "yourPassword"
  },
  
  "headers": {
    "Content-Type": "text/html"
  }
}
```

### Stateless Switch

Since **OFF** is the only possible state you do not need to declare `offUrl` and `statusUrl`

```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Switch",
          
          "switchType": "stateless",
          
          "timeout": 1000,
          
          "onUrl": "http://localhost/api/switchOn"
        }   
    ]
}  
```

### Reverse Stateless Switch

Since **ON** is the only possible state you do not need to declare `onUrl` and `statusUrl`

```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Switch",
          
          "switchType": "stateless-reverse",
          
          "timeout": 1000,
          
          "offUrl": "http://localhost/api/switchOff"
        }   
    ]
}
```

### Multiple On or Off Urls
If you wish to do so you can specify an array of urls or urlObjects (`onUrl` or `offUrl`) when your switch is a 
**stateless switch** or a **reverse-stateless switch**.  
**This is not possible with a normal stateful switch.**

Below are two example configurations of an stateless switch with three urls. 
One is using simple string array and the other is using simple urlObject arrays. 

```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Switch",
          
          "switchType": "stateless",
          "onUrl": [
            "http://localhost/api/switch1On",
            "http://localhost/api/switch2On",
            "http://localhost/api/switch3On"
          ]
        }   
    ]
}
```
```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Switch",
          
          "switchType": "stateless",
          "onUrl": [
            {
              "url": "http://localhost/api/switch1On"
            },
            {
              "url": "http://localhost/api/switch2On"
            },
            {
              "url": "http://localhost/api/switch3On"
            }
          ]
        }   
    ]
}
```

#### The 'delay(...)' url

When using multiple urls and **"series"** as `multipleUrlExecutionStrategy` you can also specify so called delay urls in the 
`onUrl` or `offUrl` arrays. This could be used to guarantee a certain delay between two urls.  
The delay url has the following pattern: **"delay(INTEGER)"** where 'INTEGER' is replaced with the delay in milliseconds.

Here is an example:
```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Delayed Switch",
          
          "switchType": "stateless",
          "multipleUrlExecutionStrategy": "series",
          
          "onUrl": [
            "http://localhost/api/switch1On",
            "delay(1000)",
            "http://localhost/api/switch2On"
          ]
        }   
    ]
}
```

### Examples for custom statusPatterns

The `statusPattern` property can be used to change the phrase which is used to identify if the switch should be turned on 
or off. So when you want the switch to be turned on when your server sends **"true"** in the body of the http response you
could specify the following pattern:
```json
{
    "statusPattern": "true"
}
```

<br>

However using Regular Expressions much more complex patterns are possible. Let's assume your http enabled device responds 
with the following json string as body, where one property has an random value an the other indicates the status of the 
switch:
```json
{
    "perRequestRandomValue": 89723789,
    "switchState": true
}
```
Then you could use the following pattern:
```json
{
    "statusPattern": "{\n    \"perRequestRandomValue\": [0-9]+,\n    \"switchState\": true\n}"
}
```
More on how to build regex patterns: https://www.w3schools.com/jsref/jsref_obj_regexp.asp

## Notification Server

`homebridge-http-switch` can be used together with 
[homebridge-http-notification-server](https://github.com/Supereg/homebridge-http-notification-server) in order to receive
updates when the state changes at your external program. For details on how to implement those updates and how to 
install and configure `homebridge-http-notification-server`, please refer to the 
[README](https://github.com/Supereg/homebridge-http-notification-server) of the repository first.

Down here is an example on how to configure `homebridge-http-switch` to work with your implementation of the 
`homebridge-http-notification-server`.

```json
{
    "accessories": [
        {
          "accessory": "HTTP-SWITCH",
          "name": "Switch",
          
          "notificationID": "my-switch",
          "notificationPassword": "superSecretPassword",
          
          "onUrl": "http://localhost/api/switchOn",
          "offUrl": "http://localhost/api/switchOff",
          
          "statusUrl": "http://localhost/api/switchStatus"
        }   
    ]
}
```

* `notificationID` is an per Homebridge instance unique id which must be included in any http request.  
* `notificationPassword` is **optional**. It can be used to secure any incoming requests.

To get more details about the configuration have a look at the 
[README](https://github.com/Supereg/homebridge-http-notification-server).

**Available characteristics (for the POST body)**

Down here are all characteristics listed which can be updated with an request to the `homebridge-http-notification-server`

* `characteristic` "On": expects a boolean `value`