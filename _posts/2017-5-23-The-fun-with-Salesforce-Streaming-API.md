---
layout: post
title: The fun with Salesforce Streaming API
crosspost_to_medium: false
excerpt_separator: <!--read more-->
---

Few weeks ago I had a time to play with a Salesforce Streaming API, the API that allows you to receive notification to Salesforce data that match a SOQL query you define. Unfortunately, the way you should use the API isn't very well defined (or at least I couldn't found anything helpful), so I needed to dig into Comet (a protocol that is used by Salesforce Streaming API) documentation and learn by doing stuff. After that, I decided to write a blog post to share what I learned.

<!--read more-->
**Prerequisites**

Before I started playing with Salesforce Streaming API, there were some things I needed to do first:


- create a Salesforce Developer Organisation. You can do it on [that page](https://developer.salesforce.com/signup)
- [create Salesforce Connected App](https://help.salesforce.com/articleView?id=connected_app_create.htm) and enable OAuth for it.
- [unrestrict your IP address for a user that you will use to obtain OAuth token](https://help.salesforce.com/articleView?id=login_ip_ranges.htm). For some reasons obtaining OAuth token didn't work for me without that step
- [enable sample console app](https://trailhead.salesforce.com/en/modules/service_basics/units/service_basics_configure_console)

When you will perform all steps described above, you can go to the next section of this blog post.

**Salesforce Streaming API**

[Salesforce Streaming API](https://developer.salesforce.com/docs/atlas.en-us.api_streaming.meta/api_streaming/intro_stream.htm) use Comet, a communication patter that allows for pushing data from a server to a client using long-pooling, callback-pooling or websocket. In Salesforce case, only long-pooling is supported. The long-pooling mechanism works like this: client establish a long lived GET request to the server. When the server don't have the data the connection stays open until it timeounts. If the server have some data, it is returned in a GET response. In both cases, client immediately reconnects to the server for next piece of the data.

Before the client will be able to perform long-pooling on the server, it's need to exchange some Bayeux protocol messages with the server in order to register to the topics it wants to listen to.

That's the theory, but how it exaclty works? Lets look at some plain POST/GET examples to see what kind of data is exchanged between client and server and what is happening during long-pooling.

**Examples**

First, let's create a PushTopic in Salesforce that will produce an event every time a new Lead is created. Go to DeveloperConsole, then Debug->Open Execute Anonymous Window:

![My helpful screenshot]({{site.url}}/assets/DeveloperConsoleCreatePushTopic.png)

and enter following code in the window:
{% highlight csharp %}
PushTopic pushTopic = new PushTopic();
pushTopic.Name = 'LeadCreation';
pushTopic.Query = 'SELECT Id, FirstName, LastName, Company FROM Lead';
pushTopic.ApiVersion = 39.0;
pushTopic.NotifyForOperationCreate = true;
insert pushTopic;
{% endhighlight %}

it will create a Salesforce Push Topic named *LeadCreated* that will produce an event every time new lead is created.

After that obtain a new OAuth token by issuing a POST request to *services/oauth2/token* endpoint.

{% highlight http %}
https://login.salesforce.com/services/oauth2/token?grant_type=password&username=<YOUR_SALESFORCE_USERNAME>&password=<YOUR_SALESFORCE_PASSWORD>&client_id=<CLIENT_ID_FROM_CONNECTED_APP_YOU_CREATED>&client_secret=<CLIENT_SECRET_FROM_CONNECTED_APP_YOU_CREATED>&redirect_uri=https%3A%2F%2Fwww.google.com%2F
{% endhighlight %}

The Salesforce server will responde with response that looks similar to the one below:

{% highlight http linenos %}
{
  "access_token": <YOUR_ACCESS_TOKEN>
  "instance_url": <INSTANCE_URL>,
  "id": <ID>,
  "token_type": "Bearer",
  "issued_at": <ISSUED_AT>,
  "signature": <SIGNATURE>
}
 {% endhighlight %}

The key fields here are *access_token* , which contains the token you will use to authorize next requests and *instance_url* which is the server you will need to use in your next requests. They reffered as {access_token} and {instance_url}, so every time you see those strings, you need to put the *access_token* and *instance_url* you received in token response.

 After obtaining the token, you need to perform a handshake with a Salesforce CometD server by sending following POST request:

{% highlight http linenos %}
 POST /cometd/39.0/ HTTP/1.1
 Host: {instance_url}
 Authorization: OAuth {access_token}
 Content-Type: application/json
 Cache-Control: no-cache
 [{
     "channel": "/meta/handshake",
     "version": "1.0",
     "minimumVersion": "1.0",
     "supportedConnectionTypes": ["long-polling", "callback-polling", "iframe", "flash"]
 }]
{% endhighlight %}

In the request body, */meta/handshake* is the channel for performing handshake - each client always needs to send first message to that channel. *version* and *minimumVersion* inform about expected protocol version used by the client and minimum protocol version client supports and *supportedConnectionTypes* informs about supported connection types by the client.

If the handshake procedure will be successful, the server will create a client session and respond with a request that should be similar to the one below:

{% highlight Json linenos %}
 [
   {
     "ext": {
       "replay": true
     },
     "minimumVersion": "1.0",
     "clientId": "<YOUR_CLIENT_ID>",
     "supportedConnectionTypes": [
       "long-polling"
     ],
     "channel": "/meta/handshake",
     "version": "1.0",
     "successful": true
   }
 ]
{% endhighlight %}

The most important part of this response it "clientId", which identify client in a request we will send next. The server will also respond with *supportedConnectionTypes* set to *long-polling*, which means Salesforce CometD server supports only that type of connection.

After the successfull handshake procedure, we can establish a connection to the server by sending a message to a */meta/connect* channel.

{% highlight http linenos %}
POST /cometd/39.0/ HTTP/1.1
Host: {instance_url}
Authorization: OAuth {access_token}
Content-Type: application/json

{
    "channel":"/meta/connect",
    "clientId":"{{client_id}}",
    "connectionType":"long-polling"
}
{% endhighlight %}

The message must define a *channel* and set it to */meta/connect*, it must provide a *client_id* received in handshake response in order to identify client and it also must specify *connectionType* that is supported by the server (remember, Salesforce support only long-pooling connection)

Response to this message may be:
{% highlight Json linenos %}
[
  {
    "clientId": "client_id",
    "advice": {
      "interval": 0,
      "timeout": 110000,
      "reconnect": "retry"
    },
    "channel": "/meta/connect",
    "successful": true
  }
]
{% endhighlight %}

If the connection was successful, *successful* flag will be set to *true*. Server also can sent an *advice* object. It may contain *interval* which informs after how long client can send another request to the server and *timeout* which informs client how much time will take the server to respond for the next connect request.

Important thing to note here is that */meta/connect* requests are heart-beat mechanism and a channel on which we receive updates for our subscriptions, which means client MUST send those request periodically, according to advice he gets from the server.

While we're connected to the server, we can send a subscribe request by issuing POST containt the information abount topic that we want to subscribe for:

{% highlight http linenos %}
POST /cometd/39.0/ HTTP/1.1
Host: {instance_url}
Authorization: Oauth {access_token}
Content-Type: application/json

[{
    "channel": "/meta/subscribe",
    "clientId": "{{client_id}}",
    "subscription": "/topic/LeadCreation"
}]
{% endhighlight %}

In the above example we're trying to subscribe to previously created LeadCreation push topic.

You should receive following response:

{% highlight http linenos %}
[
  {
    "clientId": "{{client_id}}",
    "channel": "/meta/subscribe",
    "subscription": "/topic/LeadCreation",
    "successful": true
  }
]
{% endhighlight %}

which will mean that you successfully subscribed to the channel.

Now, you need to start sending GET request to the connect channel in order to get updates when new Lead is created. After you do that, go to Salesforce Service Console and create new Lead. Immediately you should get a notification over connect channel that looks similary to this one below:

{% highlight json linenos %}
[
  {
    "data": {
      "event": {
        "createdDate": "2017-05-23T17:30:14.939Z",
        "replayId": 2,
        "type": "created"
      },
      "sobject": {
        "Company": "Master",
        "FirstName": null,
        "LastName": "Your",
        "Id": "00QB0000003JGw6MAG"
      }
    },
    "channel": "/topic/LeadCreation"
  },
  {
    "clientId": "{{client_id}}",
    "channel": "/meta/connect",
    "successful": true
  }
  {% endhighlight %}
]

In that particular case, it was an event send when I created a Lead with Company set to "Master", FirstName not set, LastName set to "Your" and "Id" set by the Salesforce.

After receiving and update, reconnect to connect channel, otherwise your session will timeout and your client session on the server will be deleted.

That's all for that post. I hope that it was helpful to you and allowed you to better understand how the Salesforce Streaming API works under the hood. If you want to dig more into the details, look at the [CometD Reference Book](https://docs.cometd.org/current/reference/) which nicely explains Bayeux protocol.
