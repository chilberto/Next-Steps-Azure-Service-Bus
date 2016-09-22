# Next Steps : Azure Service Bus Messaging with Queues using Sessions

## Introduction

This sample project builds upon the Azure SDK Template Azure Service Bus: Messaging With Queues by adding sessions and using the dead letter queue.

## Building the Project

The project should download all required packages when built.  You will first need to update the endpoint definition in the app.config of the project with a valid endpoint.

You can use the [How to use Azure Service Bus queues](https://azure.microsoft.com/en-us/documentation/articles/service-bus-dotnet-how-to-use-queues/) article to determine a suitable endpoint.

The following links are provided as an introduction:

[Getting Started: Messaging with Queues](https://code.msdn.microsoft.com/Getting-Started-Brokered-aa7a0ac3)

[Learn how to create a queue, place and read a message using Azure Service Bus Queues in 5 minutes](http://blogs.msdn.com/b/brunoterkaly/archive/2014/08/07/learn-how-to-create-a-queue-place-and-read-a-message-using-azure-service-bus-queues-in-5-minutes.aspx)

## Background
In scenarios where there are multiple Azure Service Bus clients and you want to route particular messages to particular consumers, sessions are an ideal feature of the Azure Service Bus Queue.  

There are many scenarios where this could be useful. Here are three examples:

* **Priority** - A low priority queue and a high priority queue could be created where the high priority items are worked before the low priority queue items.
* **Partitioning** - The distribution of the queue items across multiple message brokers could be used to distribute the load.
* **Aggregation** - Routing messages to specific message brokers could be used if specific messages need to be sent to specified consumers. 

## Description
To illustrate sessions, three messages will be posted to the service bus. Two will have the same session id and the third will have a different session id.

We will then use two different approches to consuming the messages.  The first will be to create two clients to consume the messages specifying a particularly session id.  The second will be to use an implemenation of IMessageSessionHandler.

## Queue Description

The queue must be created to require sessions.  This is done as follows:

```C#
var description = new QueueDescription(QueueName) 
{ 
    RequiresSession = true 
}; 
 
namespaceManager.CreateQueue(description);
```

## BrokeredMessage 

When a message created, a session must be specified otherwise the send will result in an exception.  This can be done when the message is constructed:

```C#
BrokeredMessage message = new BrokeredMessage(messageBody)  
                            { 
                                MessageId = messageId, 
                                SessionId = sessionId 
                            };
```

## QueueClient

The first approach to consuming the messages will be to use a QueueClient object.  As our application is the only consumer, I decided to use a foreach statement to loop through all the sessions and consume all the messages in each sessions.  

Note: to illustrate DeadLetter, I am sending the items from Session2 to DeadLetter.

The following is the modified method for consuming the items:

```C#
private static void ReceiveMessages() 
{ 
    Console.WriteLine("\nReceiving message from Queue..."); 
    BrokeredMessage message = null; 
 
    var sessions = queueClient.GetMessageSessions(); 
 
    foreach (var browser in sessions) 
    { 
        Console.WriteLine(string.Format("Session discovered: Id = {0}", browser.SessionId)); 
        var session = queueClient.AcceptMessageSession(browser.SessionId);                 
 
        while (true) 
        { 
            try 
            { 
                message = session.Receive(TimeSpan.FromSeconds(5)); 
 
                if (message != null) 
                { 
                    Console.WriteLine(string.Format("Message received: Id = {0}, Body = {1}", message.MessageId, message.GetBody<string>())); 
                     
                    if (session.SessionId == SessionId2) 
                        // if this is the second session then let's send to the dead letter for fun 
                        message.DeadLetter(); 
                    else                             
                        // Further custom message processing could go here... 
                        message.Complete(); 
                } 
                else 
                { 
                    //no more messages in the queue 
                    break; 
                } 
            } 
            catch (MessagingException e) 
            { 
                if (!e.IsTransient) 
                { 
                    Console.WriteLine(e.Message); 
                    throw; 
                } 
                else 
                { 
                    HandleTransientErrors(e); 
                } 
            } 
        } 
    } 
    queueClient.Close(); 
}
``` 

## IMessageSessionHandler

Using IMessageSessionHandler is a great way to consume the messages if the session id is not known.  You first register the session handler against the queue:

```C#
queueClient.RegisterSessionHandler(typeof(MyMessageSessionHandler), new SessionHandlerOptions { AutoComplete = false });
```

Messages are then received by the implementation in the OnMessage method:

```C#
public void OnMessage(MessageSession session, BrokeredMessage message) 
{ 
    Console.WriteLine(string.Format("MyMessageSessionHandler {3} received messaged on session Id = {0}, Id = {1}, Body = {2}", session.SessionId, message.MessageId, message.GetBody<string>(), WhoAmI)); 
 
    message.Complete(); 
}
```

## DeadLetter

There are several uses for the DeadLetter channel on a queue but it is primarily used when messages failed to be handled or expired before they were completed.

In the project, items are explicitly sent to DeadLetter:
```C#
message.DeadLetter();
```

And likewise read from the DeadLetter channel:

```C# 
var deadLetterPath = QueueClient.FormatDeadLetterPath(QueueName); 
 
var deadLetterClient = QueueClient.Create(deadLetterPath); 
 
while (true) 
{ 
    try 
    { 
        message = deadLetterClient.Receive(TimeSpan.FromSeconds(5));
```

## Summary
The Azure Service Bus offers a huge amount of scope that might be overlooked initially due to its overall simplicity when getting started.  Sessions and DeadLetter are just two features that extend the capability of the service bus.

Any feedback is appreciated!
