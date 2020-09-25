## Installation

The library is distributed via [NuGet](https://www.nuget.org/packages/gds-messages/) package. You can install this package with running this command in the Package Manager Console.

`Install-Package gds-messages -Version 1.1.2`

(The library was made by [this](https://github.com/neuecc/MessagePack-CSharp) messagepack C# implementation)



## Usage

Messages can be sent to the GDS via WebSocket protocol. The SDK contains a WebSocket client with basic functionalities, so you can use this to send and receive messages.
You can also find a GDS Server Simulator written in Java [here](https://github.com/arh-eu/gds-server-simulator). With this simulator you can test your client code without a real GDS instance.

A message can be sent as follows.

### Creating the client

First, we create the client object and connect to the GDS.
```csharp
GdsWebSocketClient client = new GdsWebSocketClient("ws://127.0.0.1:8888/gate", "user");
``` 

For simple password authentication an additional parameter can be passed to the client constructor:

```csharp
GdsWebSocketClient client = new GdsWebSocketClient("ws://127.0.0.1:8888/gate", "user", "u$€r_p4$$w0rD");
```

If you want to use secured connection you should invoke another constructor when you create the client. This one needs 4 parameters:

 - GDS URL
 - username
 - Path to the `PKCS12` formatted private key
 - Password to unlock the private key 
 
With these the connection will be encrypted over TLS. The GDS URL should be the one with the secure port/gate as well.

```csharp
GdsWebSocketClient client = new GdsWebSocketClient("wss://127.0.0.1:8443/gates", "tls_user", "tlsuser_private_key.p12", "very_secret_password_for_the_tls_certificate");
```


The library uses [log4net](https://logging.apache.org/log4net/) for logging. So the application needs to be configured accordingly.
Logging to the console is easy. Put this code in the beginning of your main() method.
```csharp
var logRepository = log4net.LogManager.GetRepository(System.Reflection.Assembly.GetEntryAssembly());    
log4net.Config.BasicConfigurator.Configure(logRepository);
```

### Subscribing to listeners

If you want to use async communication, there are some listeners you can subscribe.

To get the serialized message objects.
```csharp
client.MessageReceived += Client_MessageReceived;

static void Client_MessageReceived(object sender, Tuple<Message, MessagePackSerializationException> e)
{
    //...
}

```

Or to get the binary representation of the message.
```csharp
client.BinaryMessageReceived += Client_BinaryMessageReceived;

static void Client_BinaryMessageReceived(object sender, byte[] e)
{
	//...
}
```

If you would like to be notified of changes in the connection status, you can subscribe to the following listeners.
```csharp
client.Connected += Client_Connected;

static void Client_Connected(object sender, EventArgs e)
{
    Console.WriteLine("Connected");
}
```

```csharp
client.Disconnected += Client_Disconnected;

static void Client_Disconnected(object sender, EventArgs e)
{
    Console.WriteLine("Disconnected");
}
```

```csharp
client.Connect();
``` 

(During the connection, a connection type message is also sent after the websocket connection. If a positive acknowledgment message arrives, the IsConnected() method returns true.)

### Create messages

A message consists of two parts, a header and a data. You can create these objects through the `Gds.Messages.MessageManager` class.

The following example shows the process of creating messages by creating an attachment request type message.

First, we create the header part.
```csharp
MessageHeader header = MessageManager.GetHeader("user", "870da92f-7fff-48af-825e-05351ef97acd", 1582612168230, 1582612168230, false, null, null, null, null, DataType.AttachmentRequest);
```

After that, we create the data part.
```csharp
MessageData data = MessageManager.GetAttachmentRequestData("SELECT * FROM \"multi_event-@attachment\" WHERE id='ATID2006241023125470' and ownerid='EVNT2006241023125470' FOR UPDATE WAIT 86400");
```

Once we have a header and a data, we can create the message object.
```csharp
Message message = MessageManager.GetMessage(header, data);
```

### Send and receive messages

After you connected, you can send messages to the GDS. You can do that with the SendSync() and SendAsync() methods.

Let's see an event message for example.

```csharp
string operationsStringBlock = "INSERT INTO multi_event (id, plate, images) VALUES('EVNT2006241023125476', 'ABC123', array('ATID2006241023125470'));INSERT INTO \"multi_event-@attachment\" (id, meta, data) VALUES('ATID2006241023125470', 'some_meta', 0x62696e6172795f69645f6578616d706c65)";
Dictionary<string, byte[]> binaryContentsMapping = new Dictionary<string, byte[]> { { "62696e6172795f69645f6578616d706c65", new byte[] { 1, 2, 3 } } };
MessageData eventMessageData = MessageManager.GetEventData(operationsStringBlock, binaryContentsMapping);

client.SendAsync(eventMessageData);
```

Or if you want to define the header part explicitly.
```csharp
MessageHeader eventMessageHeader = MessageManager.GetHeader("user", "c08ea082-9dbf-4d96-be36-4e4eab6ae624", 1582612168230, 1582612168230, false, null, null, null, null, DataType.Event);
Message eventMessage = MessageManager.GetMessage(eventMessageHeader, eventMessageData);

client.SendAsync(eventMessage);
```

The response is available through the subscribed listener.
```csharp
static void Client_MessageReceived(object sender, Tuple<Message, MessagePackSerializationException> e)
{
    if (e.Item2 == null)
    {
        Message eventResponseMessage = e.Item1;
        if (eventResponseMessage.Header.DataType.Equals(DataType.EventAck))
        {
            EventAckData eventAckData = eventResponseMessage.Data.AsEventAckData();
            // do something with the event ack data...
        }
    }
}
```

Let's look at the same in synchronously.
```csharp
try
{
    Message response = client.SendSync(eventMessage, 3000);
    if (response.Header.DataType.Equals(DataType.EventAck))
    {
        EventAckData eventAckData = response.Data.AsEventAckData();
        // do something with the response data...
    }
}
catch (TimeoutException exception)
{
    // ...
}
catch (MessagePackSerializationException exception)
{
    // ...
}
```

At the end, we close the websocket connection as well.
```csharp
client.Close();
```
