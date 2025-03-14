# Jolie_Mosquitto

Jolie connector for Mosquitto framework

- Jolie: https://www.jolie-lang.org/
- Mosquitto: https://mosquitto.org/

## Architecture

The MQTT protocol is based on the principle of publishing messages and subscribing to topics, or "pub/sub". Multiple clients connect to a broker and subscribe to topics that they are interested in. Clients also connect to the broker and publish messages to topics. Many clients may subscribe to the same topics and do with the information as they please. The broker and MQTT act as a simple, common interface for everything to connect to.

![Architecture](https://github.com/jolie/jolie-mosquitto/blob/master/architecture.png)

JavaService uses the Paho library to create both the publisher and the subscriber.
Jolie's client service communicates with the JavaService, which takes care of creating a publisher client, creating the connection with the Mosquitto broker and finally transmitting the message.
Jolie's server service communicates with JavaService, which is responsible for creating a subscriber client, creating the connection with the Mosquitto broker and finally subscribing to the desired topics.

To install the Mosquitto broker on your computer follow the instructions provided by the official Eclipse Mosquitto website at the link: https://mosquitto.org/download/

## Example

To be able to use the connector correctly, be sure to add both the file _mosquitto.jap_ and _org.eclipse.paho.client.mqttv3-1.2.5.jar_ to your project folder.
- ```mosquitto.jap``` contains both the connector and the interfaces necessary for the Jolie service to communicate with the Mosquitto broker.
- ```org.eclipse.paho.client.mqttv3-1.2.5.jar``` is the dependency on the Paho library that JavaService uses to create the publisher and subscriber.

### CLIENT - SERVER :

#### server.ol

```java
from mosquitto.mqtt.mqtt import MQTT
from mosquitto.mqtt.mqtt import MosquittoReceiverInterface
from console import Console

service serverMQTT {

embed Console as Console
embed MQTT as Mosquitto

execution: concurrent

inputPort Server {
    Location: "local"
    Interfaces: MosquittoReceiverInterface
}

init {
    request << {
        brokerURL = "tcp://mqtt.eclipseprojects.io:1883",
        subscribe << {
            topic = "jolie/test"
        }
        // I can set all the options available from the Paho library
        options << {
            setAutomaticReconnect = true
            setCleanSession = false
            setConnectionTimeout = 25
            setKeepAliveInterval = 0
            setMaxInflight = 200
            setUserName = "username"
            setPassword = "password"
            setWill << {
                topicWill = "home/LWT"
                payloadWill = "server disconnected"
                qos = 0
                retained = true
            }
        }
    }
    setMosquitto@Mosquitto (request)()
}

main {
    receive (request)
    println@Console("topic :     "+request.topic)()
    println@Console("message :   "+request.message)()
}

}
```

You can modify all options values and the topic you want to subscribe in. The string "home/#" is used to subscribe on every subtopic of "home".
An example of launch of this client is:  
    ```jolie server.ol```.

#### client.ol

```java
from mosquitto.mqtt.mqtt import MQTT

service clientMQTT {

embed MQTT as Mosquitto

init {
    req << {
        brokerURL = "tcp://mqtt.eclipseprojects.io:1883"
        // I can set all the options available from the Paho library
        options << {
            setAutomaticReconnect = true
            setCleanSession = false
            setConnectionTimeout = 20
            setKeepAliveInterval = 20
            setMaxInflight = 150
            setUserName = "username"
            setPassword = "password"
            setWill << {
                topicWill = "home/LWT"
                payloadWill = "client disconnected"
                qos = 0
                retained = true
            }
        }
    }
    setMosquitto@Mosquitto (req)()
}

main {
    request << {
        topic = "jolie/test",
        message = args[0]
    }
    sendMessage@Mosquitto (request)()
}

}
```

You can modify all options values and the topic you want to publish in.
An example of launch of this client is:  
    ```jolie client.ol "hello"```.

#### MosquittoInterface.iol respectively mqtt/mqtt.ol

```java
type SetMosquittoRequest: void {
    clientId?: string
    brokerURL: string
    options?: void {
        setAutomaticReconnect?: bool
        setCleanSession?: bool
        setConnectionTimeout?: int
        setKeepAliveInterval?: int
        setMaxInflight?: int
        setServerURIs?: string
        setUserName?: string
        setPassword?: string
        setWill?: void {
            topicWill: string
            payloadWill: string
            qos: int
            retained: bool 
        }
        setSocketFactory?: void {
            caCrtFile: string
            crtFile: string
            keyFile: string
            password: string
        }  
	    debug?: bool
    }
    subscribe?: void {
        topic[1,*]: string
    }
}

type MosquittoMessageRequest: void {
    topic: string
    message: string
}

interface MosquittoPublisherInterface {
    RequestResponse:
        sendMessage (MosquittoMessageRequest)(void)
}

interface MosquittoInterface {
    RequestResponse:
        setMosquitto (SetMosquittoRequest)(void),
}

interface MosquittoReceiverInterface {
    OneWay:
        receive (MosquittoMessageRequest)
}

service MQTT {
    inputPort ip {
        location: "local"
        interfaces: MosquittoPublisherInterface, MosquittoInterface, MosquittoReceiverInterface
    }
    foreign java {
        class: "org.jolielang.connector.mosquitto.MosquittoConnectorJavaService"
    }
}
```

The ```MosquittoPublisherInterface``` exposes a method called ```sendMessage``` which receives an input request of the ```MosquittoMessageRequest``` type. This type requires two fields: a ```topic``` (to publish the message) and a ```message``` (to publish).

The ```MosquittoInterface``` exposes a method called ```setMosquitto``` which receives in input a request of the type ```SetMosquittoRequest```. This type requires two mandatory fields: a ```brokerURL``` (which is used to connect to the broker Mosquitto) and a ```clientId``` (if not specified a random one is generated).
You can also specify some values of the ```options``` to customize the connection (if not specified, the default values are used).

The ```MosquittoReceiverInterface``` exposes a method called ```receive``` which receives a ```MosquittoMessageRequest``` request, described above.

The interface is located inside the ```moquitto.jap``` file (or ```JolieMosquittoConnector.jar``` for the older API).

### CHAT :

In this example I wanted to apply the MQTT communication protocol to a simple chat.
Always exploiting the JavaService explained in the previous example, and exploiting Leonardo (a Web Server written in Jolie: https://github.com/jolie/leonardo) is sufficient to launch the command ```jolie leonardo.ol``` and subsequently open a browser to the page ```localhost:16000``` to observe its operation.

To test the chat with your friends without necessarily having a server with Mosquitto installed, just use the sandbox provided at the link https://iot.eclipse.org/projects/sandboxes/

#### frontend.ol

```java
from mosquitto.mqtt.mqtt import MQTT
from mosquitto.mqtt.mqtt import MosquittoReceiverInterface
from console import Console
from json_utils import JsonUtils

type SendChatMessageRequest: void {
    message: string
}

type GetChatMessagesResponse: void {
    messageQueue*: void {
        message: string
        username: string
        session_user: string
    }
}

type SetUsernameRequest: void {
    username: string
}

interface FrontendInterface {
    RequestResponse:
		sendChatMessage( SendChatMessageRequest )( void ),
        getChatMessages( void )( GetChatMessagesResponse ),
        setUsername( SetUsernameRequest )( void )
}

service Frontend {

embed MQTT as Mosquitto
embed Console as Console
embed JsonUtils as JsonUtils

execution: concurrent

inputPort Frontend {
    Location: "local"
    Interfaces: MosquittoReceiverInterface, FrontendInterface
}

init {
    request << {
        brokerURL = "tcp://mqtt.eclipseprojects.io:1883",
        subscribe << {
            topic = "jolie/test/chat"
        }
        // I can set all the options available from the Paho library
        options << {
            setUserName = "joliechat"
            setPassword = "joliechat"
        }
    }

    setMosquitto@Mosquitto (request)()
    println@Console("SUBSCRIBER connection done! Waiting for message on topic : "+request.subscribe.topic)()
}

main {
    [ receive (request) ] {
        getJsonValue@JsonUtils(request.message)(jsonMessage)
        println@Console("RECEIVE ["+jsonMessage.username+"] done! Message correctly received: "+jsonMessage.message)()
        synchronized( username ) {
            session_user = global.username
        }
        synchronized( messageQueue ) {
            global.messageQueue[#global.messageQueue] << {
                message = jsonMessage.message
                username = jsonMessage.username
                session_user = session_user
            }
        }
    }

    [ getChatMessages( GetChatMessagesRequest )( GetChatMessagesResponse ) {
        synchronized( messageQueue ) {
            for (i=0, i<#global.messageQueue, i++) {
                GetChatMessagesResponse.messageQueue[i] << global.messageQueue[i]
            }
            undef(global.messageQueue)
        }
    }]

    [ sendChatMessage( messageRequest )( response ) {
        synchronized( username ) {
            json << {
                username = global.username
                message = messageRequest.message
            }
        }
        getJsonString@JsonUtils(json)(jsonString)
        req << {
            topic = "jolie/test/chat",
            message = jsonString
        }
        sendMessage@Mosquitto (req)()
        println@Console("PUBLISH ["+json.username+"] done! Message correctly sent: "+json.message)()
    }]

    [ setUsername( usernameRequest )( usernameResponse ) {
        synchronized( username ) {
            global.username = usernameRequest.username
            println@Console("Username set for the current session: "+global.username)()
        }
    }]
}

}
```

The ```frontend.ol``` service presents an ```inputPort``` which communicates both with Mqtt (```MosquittoReceiverInterface```) and with the ```FrontendInterface``` interface.
In the ```init``` method, it prepares the request (setting all the desired parameters) to send to the embedded ```Mosquitto``` service to open a connection with the broker Mosquitto.
In the ```main``` method, instead, it develops four operations:
- **setUsername:** sets the username of the user who wants to connect to the chat.
- **sendChatMessage:** publish the messages sent in the chat at the broker Mosquitto to the topic set in the connection. The message is converted into a json to allow the sending of additional information along with the message text.
- **receive:** receives the messages from broker Mosquitto and saves them in a global variable ```messageQueue```.
- **getChatMessages:** this operation reads all the messages in the queue, sends them to the WebService which prints them in the chat and empties the ```messageQueue```. This operation is called cyclically every 0.5 seconds by the web page.

#### Options

In order to let you customize your communications, you can modify some options (these descriptions are taken from the official Paho library documentation):
https://www.eclipse.org/paho/files/javadoc/org/eclipse/paho/client/mqttv3/MqttConnectOptions.html

- **setAutomaticReconnect : bool**
Sets whether the client will automatically attempt to reconnect to the server if the connection is lost.
    - If set to **false**, the client will not attempt to automatically reconnect to the server in the event that the connection is lost.
    - If set to **true**, in the event that the connection is lost, the client will attempt to reconnect to the server. 
    It will initially wait 1 second before it attempts to reconnect, for every failed reconnect attempt, the delay will double until it is at 2 minutes at which point the delay will stay at 2 minutes.

- **setCleanSession : bool**
Sets whether the client and server should remember state across restarts and reconnects.
    - If set to **false** both the client and server will maintain state across restarts of the client, the server and the connection. As state is maintained:
        - Message delivery will be reliable meeting the specified QOS even if the client, server or connection are restarted.
        - The server will treat a subscription as durable.
    - If set to **true** the client and server will not maintain state across restarts of the client, the server or the connection. This means:
        - Message delivery to the specified QOS cannot be maintained if the client, server or connection are restarted
        - The server will treat a subscription as non-durable.

- **setConnectionTimeout : int**
Sets the connection timeout value. This value, measured in seconds, defines the maximum time interval the client will wait for the network connection to the MQTT server to be established. The default timeout is 30 seconds. 
A value of 0 disables timeout processing meaning the client will wait until the network connection is made successfully or fails.

- **setKeepAliveInterval : int**
Sets the "keep alive" interval. This value, measured in seconds, defines the maximum time interval between messages sent or received. It enables the client to detect if the server is no longer available, without having to wait for the TCP/IP timeout. The client will ensure that at least one message travels across the network within each keep alive period. In the absence of a data-related message during the time period, the client sends a very small "ping" message, which the server will acknowledge. A value of 0 disables keepalive processing in the client.
The default value is 60 seconds.

- **setMaxInflight : int**
Sets the "max inflight". please increase this value in a high traffic environment.
The default value is 10.

- **setUserName : string**
Sets the user name to use for the connection.

- **setPassword : char[]**
Sets the password to use for the connection.

- **setServerURIs : string[]**
Set a list of one or more serverURIs the client may connect to.
Each serverURI specifies the address of a server that the client may connect to. Two types of connection are supported tcp:// for a TCP connection and ssl:// for a TCP connection secured by SSL/TLS. For example:
    tcp://localhost:1883
    ssl://localhost:8883
If the port is not specified, it will default to 1883 for tcp://" URIs, and 8883 for ssl:// URIs.
If serverURIs is set then it overrides the serverURI parameter passed in on the constructor of the MQTT client.
When an attempt to connect is initiated the client will start with the first serverURI in the list and work through the list until a connection is established with a server. If a connection cannot be made to any of the servers then the connect attempt fails.
Specifying a list of servers that a client may connect to has several uses:
    - **High Availability and reliable message delivery**
    Some MQTT servers support a high availability feature where two or more "equal" MQTT servers share state. An MQTT client can connect to any of the "equal" servers and be assured that messages are reliably delivered and durable subscriptions are maintained no matter which server the client connects to.
    The cleansession flag must be set to false if durable subscriptions and/or reliable message delivery is required.
    - **Hunt List**
    A set of servers may be specified that are not "equal" (as in the high availability option). As no state is shared across the servers reliable message delivery and durable subscriptions are not valid. The cleansession flag must be set to true if the hunt list mode is used.

- **setWill : void**
Sets the "Last Will and Testament" (LWT) for the connection. In the event that this client unexpectedly loses its connection to the server, the server will publish a message to itself using the supplied details.
**Parameters**:
    - **topic** - the topic to publish to : ```string```
    - **payload** - the byte payload for the message : ```string```
    - **qos** - the quality of service to publish the message at (0, 1 or 2) : ```int```
    - **retained** - whether or not the message should be retained : ```bool```

- **setSocketFactory : void**
Sets the SocketFactory to use. This allows an application to apply its own policies around the creation of network sockets. If using an SSL connection, an SSLSocketFactory can be used to supply application-specific security settings.
**Parameters**:
    - **caCrtFile** - the path to the CA Certificate file : ```string```
    - **crtFile** - the path to the Server Certificate file : ```string```
    - **keyFile** - the path to the Server Key Pair file : ```string```
    - **password** - the password you used to encrypt the certificates : ```string```

#### SSL configuration

So far we have seen how to implement the MQTT protocol for data transmission without worrying about communication security.

In order to properly run the connector with an SSL encrypted connection, you need to add dependencies to your project to the **Bouncy Castle** library https://www.bouncycastle.org/java.html.
In particular, just add the following two jar files:
- ```bcpkix-jdk15to18-166.jar```;
- ```bcprov-ext-jdk15to18-166.jar```;
to download the jars go to https://www.bouncycastle.org/latest_releases.html.

#### Username and Password

The MQTT protocol provides different degrees of security.
The simplest is authentication with **username** and **password**. In this case the communication is not secure (the transmitted data are not encrypted) but it is a good way to limit the access to the broker only to clients who know the username and password authentication.
To implement this behavior you need to perform two steps:
- create a password file;
- modify the ```mosquitto.conf``` file to force the use of the password;
To create a pasword file, open a text editor and enter your username and password, one for each line, separated by two dots as shown below:
```java
admin:adminPassword
test:testPassword
```

Now you need to convert the password file that encrypts passwords, go to a command line and type:
```mosquitto_passwd -U passwordfile```

At this point you must copy the newly encrypted password file to the mosquitto installation folder.
Now you only have to edit the ```mosquitto.conf``` file in which you will have to set the following commands:
```java
allow_anonymous false
password_file C:\mosquitto\passwords.txt
```
In order to take advantage of the changes made it will be necessary to restart the execution of the Mosquito broker.

After performing all the above steps, in the examples (```client_server``` and ```chat```) you can take advantage of the ```setUserName``` and ```setPassword``` connection options displayed in the ```MosquittoInterface.iol``` interface to create a connection to your Mosquitto broker as shown below:
```java
    request << {
        brokerURL = "ssl://localhost:8883",
        subscribe << {
            topic = "jolie/test/chat"
        }
        // I can set all the options available from the Paho library
        options << {
            setUserName = "test"
            setPassword = "testPassword"
        }
    }
    setMosquitto@Mosquitto (request)()
```

#### Certified based encryption

Below is a text extracted from Mosquitto's official documentation (https://mosquitto.org/man/mosquitto-conf-5.html#):

_When using certificate based encryption there are three options that affect authentication. The first is require_certificate, which may be set to true or false. If false, the SSL/TLS component of the client will verify the server but there is no requirement for the client to provide anything for the server: authentication is limited to the MQTT built in username/password. If require_certificate is true, the client must provide a valid certificate in order to connect successfully. In this case, the second and third options, use_identity_as_username and use_subject_as_username, become relevant. If set to true, use_identity_as_username causes the Common Name (CN) from the client certificate to be used instead of the MQTT username for access control purposes. The password is not used because it is assumed that only authenticated clients have valid certificates. This means that any CA certificates you include in cafile or capath will be able to issue client certificates that are valid for connecting to your broker. If use_identity_as_username is false, the client must authenticate as normal (if required by password_file) through the MQTT options. The same principle applies for the use_subject_as_username option, but the entire certificate subject is used as the username instead of just the CN._

In order to implement this type of encryption it is necessary to create a series of certificates. This is a fairly delicate step so I suggest you to follow the tutorial at the following link: https://mcuoneclipse.com/2017/04/14/enable-secure-communication-with-tls-and-the-mosquitto-broker/

Once you have created the certificates, properly modified the ```mosquitto.conf``` file and restarted the Mosquitto broker as shown in the tutorial above, you are ready to implement an encrypted communication in the ```client_server``` and ```chat``` examples.

To do this you need to set the certificate paths in the ```setSocketFactory``` option exposed by the ```MosquittoInterface.iol``` interface as shown below:

```java
    request << {
        brokerURL = "ssl://localhost:8883",
        subscribe << {
            topic = "jolie/test/chat"
        }
        // I can set all the options available from the Paho library
        options << {
            setSocketFactory << {
                caCrtFile = "C:\\Program Files\\mosquitto\\certs\\m2mqtt_ca.crt"
                crtFile = "C:\\Program Files\\mosquitto\\certs\\m2mqtt_srv.crt"
                keyFile = "C:\\Program Files\\mosquitto\\certs\\m2mqtt_srv.key"
                password = "password" //the password you used to encrypt the certificates
            }
        }
    }
    setMosquitto@Mosquitto (request)()
```

### MOSQUITTO PLUGIN :

This plugin was developed to allow the Mosquitto connector to be transparent to the developer.
This way a developer can develop his application and then choose to use MQTT as a data transmission protocol.
The plugin will be embedded to the client-side service and will capture all requests that pass through the outputPort connected to the server-side service.
Once captured the request it will send it to broker Mosquitto on a well defined topic.
On the server side the plugin will subscribe to all the topics coming from the client and every time a message arrives it will be delivered to the server side service through its inputPort.
Finally the plugin, in case of RequestResponse operations, will transmit the response to the client side service always passing through Mosquitto broker.
In this way the developer will be able to design his services as if the communication was done in a direct way, while in the middle the plugin will use the broker Mosquitto to transmit data from one service to another.

In the case of a simple ```client.ol``` service that communicates with a ```server.ol``` service, the resulting architecture will be as follows:

![Plugin_Architecture](https://github.com/jolie/jolie-mosquitto/blob/master/plugin_diagram.png)

To achieve this behavior, the plugin uses the potential of ```MetaRender``` and ```MetaJolie``` services offered by the Jolie language.
Launching the service ```mqttTransformPort.ol``` in the same directory of the desired service and passing it as parameters (```service_name```, ```port_name```, ```ouput/input```) will automatically generate a ```mosquittoPlugin.ol``` file that will be able to intercept requests and responses in or out of the port and divert them to the Mosquitto broker.

Once this process is performed on both client and server side, it will be enough to embed the ```mosquittoPlugin.ol``` files to the respective client-side and server-side services.

The generated ```mosquittoPlugin.ol``` file has the same name but different content depending on whether it is created for the client-side service (passing as last argument ```output```), or for the server-side service (passing as last argument ```input```).

Let's see now in the detail the example in the ```mosquitto_plugin``` directory.

This is an example taken from Jolie's official documentation, where a ```client.ol``` service requires the user to enter two operands and the operation you want to perform [sum|mul|div|sub].
This request is sent to the ```calculator.ol``` service that using the principles of dynamic embedding, calculates the result of the operation and sends the response to the client.

As you can see, the example has been divided into two folders ```/client``` and ```/server``` to simulate a real case.
In both folders you need to add the ```mosquitto.jap``` connector and the jar of the Eclipse Paho library in a subdir called ```/lib```.

At this point, inside both folders you have to launch the file ```mqttTransformPort.ol``` passing as arguments:

- ```service_name```: the name of the service you want to communicate via mqtt;
- ```port_name```: the name of the port that makes this service communicate with the other service;
- ```input/output```: input in case it is an inputPort, output in case it is an outputPort.

In this case the two calls will be:

- in the ```/client``` folder: ```jolie ../mqttTransformPort.ol client.ol Calculator output```
- in the ```/server``` folder: ```jolie ../mqttTransformPort.ol calculator.ol Calculator input```

After these calls in both folders a ```mosquittoPlugin.ol``` file will be generated automatically.
Each of these files will have to be embedded to its corresponding service.
In the ```client.ol``` file I added a static embedding using the following lines of code:

```java
embedded {
    Jolie: "mosquittoPlugin.ol" in Calculator
}
```

In the ```calculator.ol``` file I used runtime embedding in the init section of the service:

```java
emb << {
    filepath = "mosquittoPlugin.ol",
    type = "Jolie"
}
loadEmbeddedService@Runtime( emb )(  )
```

Now if you launch the ```calculator.ol``` service and then the ```client.ol``` service you will get the desired result, but in reality the messages have been transferred through the broker Mosquitto.

As last thing, I would like to explain how the topics are automatically generated.
If you look at the client-side generated ```mosquittoPlugin.ol``` file, you will notice that:

```java
init {
    clientToken = new
    topicRootResponse = "mqttTransform4Jolie/response/socket://localhost:8000/"+clientToken
    topicRootRequest = "mqttTransform4Jolie/request/socket://localhost:8000/"+clientToken
    req << {
        brokerURL = "tcp://mqtt.eclipseprojects.io:1883"
        subscribe << {
            topic = topicRootResponse+"/#"
        }
    }
    setMosquitto@Mosquitto (req)()
}
```

The client-side plugin subscribes to the topic ```"mqttTransform4Jolie/response/socket://localhost:8000/"+clientToken```:

- ```mqttTransform4Jolie```: is a string useful to understand that the topic was generated by the plugin;
- ```response```: indicates that responses are published on this topic;
- ```socket://localhost:8000```: is the location of the outputPort that communicates with the ```calculator.ol``` service;
- ```clientToken```: is a token that uniquely identifies me the client.

In this way the client-side plugin receives from Mosquitto all messages published as response and referred to the desired client.

Within each operation, the request sent by ```client.ol``` and directed to ```calculator.ol``` is captured, transformed into a json string and sent to Mosquitto publishing on the topic ```topicRootRequest+"/<operation_name>/"+csets.session_token```:

As you can see in the ```init```: ```topicRootRequest = "mqttTransform4Jolie/request/socket://localhost:8000/"+clientToken```

The message is then published on a topic that indicates that it is a ```request``` and also inserts a ```session token``` that identifies the session to which the request refers.
In this way the plugin can wait to receive the response from Mosquitto for the correct session. 

The server-side plugin subscribes to the request topic.
Based on the operation name contained in the topic, it sends the message to the corresponding operation of the ```calculator.ol``` service.
Finally, in case of ```RequestResponse``` operations, it captures the ```response``` and publishes it on Mosquitto to the ```response``` topic with the corresponding ```session token```.
In this way client-side plugin is able to correlate the response to the right request.
