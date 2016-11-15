---
layout: page
title: Proactive bot
permalink: /en-us/azure-bot-service/templates/proactive/
weight: 12200
parent1: Azure Bot Service
parent2: Templates
---

* TOC
{:toc}

## What is a proactive bot?
The most common bot use case is when a user initiates the interaction by chatting with the bot via their favorite channel. What if you wanted to have the bot contact the user based on some triggered event, some lengthy job, or some external events changing state of system (e.g. pizza ready to pickup)? The proactive bot template is designed to illustrate just that! 

## The proactive bot scenario in the proactive template
The proactive bot template provides all the Azure resources you need to enable a very simple proactive scenario, in particular. Here is a diagram with brief overview of system in our example:

![Overview of Example Proactive Bot](/en-us/images/azure-bots/azure-bot-proactive-diagram.png)

When you create a proactive bot with the Azure Bot Service, you will find these Azure resources in your resource group:

- Azure Storage (used to create the queue)
- Azure Function App (a queueTrigger Azure Function) - Triggered whenever there is a message in queue, and communicates to Bot service via DirectLine. This function uses Bot binding to send out message as part of trigger payload. Our example function forwards user message as is from the queue.
- Azure Bot Service (your bot) - Contains logic of receiving message from user, enqueue message with required properties (Recipient, User's message) into to Azure Queue, and logic for receiving triggers from Azure Function sending back the message it received from trigger payload.

Everything is properly configured and ready to work.

### Receiving a message from the user and adding it to an Azure Storage Queue (Azure Bot Service)
Here's the snippet of code that receives the message from the user, adds it to an Azure Storage Queue, and finally sends back an acknowledgment to the user. Notice that the message is wrapped in an object that contains all the information needed to send the message back to the user on the right channel ([ResumptionCookie](/en-us/csharp/builder/sdkreference/dc/d2b/class_microsoft_1_1_bot_1_1_builder_1_1_dialogs_1_1_resumption_cookie.html){:target="_blank"} for C# and [session.message.address]() for Node.js).

<div id="thetabs1">
    <ul>
        <li><a href="#tab11">C#</a></li>
        <li><a href="#tab12">Node.js</a></li>
    </ul>

    <div id="tab11">

{% highlight csharp %}
    public virtual async Task MessageReceivedAsync(IDialogContext context, IAwaitable<IMessageActivity> argument)
    {
        var message = await argument;
        var queueMessage = new Message
        {
            ResumptionCookie = new ResumptionCookie(message),
            Text = message.Text
        };
        
        // AddMessageToQueue() is a utility method you can find in the template
        AddMessageToQueue(JsonConvert.SerializeObject(queueMessage));
        await context.PostAsync($"{this.count++}: You said {queueMessage.Text}. Message added to the queue.");
        context.Wait(MessageReceivedAsync);
    }

{% endhighlight %}

    </div>
    <div id="tab12">

{% highlight JavaScript %}
bot.dialog('/', function (session) {
    var queuedMessage = { address: session.message.address, text: session.message.text };
    session.sendTyping();
    var queueSvc = azure.createQueueService(process.env.AzureWebJobsStorage);
    queueSvc.createQueueIfNotExists('bot-queue', function(err, result, response){
        if(!err){
            var queueMessageBuffer = new Buffer(JSON.stringify(queuedMessage)).toString('base64');
            queueSvc.createMessage('bot-queue', queueMessageBuffer, function(err, result, response){
                if(!err){
                    session.send('Your message (\'' + session.message.text + '\') has been added to a queue, and it will be sent back to you via a Function');
                } else {
                    session.send('There was an error inserting your message into queue');
                }
            });
        } else {
            session.send('There was an error creating your queue');
        }
    });

});
{% endhighlight %}

    </div>  
</div>


### Triggering an Azure Function with the queue, and sending the message back to the user (Azure Functions)
Once the message has been added to the queue, the Function will be triggered, it will de-queue the message, and send it back to the user. If you inspect the Function configuration file, you will see that it contains an input binding of type "queueTrigger", and an output binding of type "bot".

This is the Functions configuration file functions.json.

{% highlight JSON %}
{
  "bindings": [
    {
      "name": "myQueueItem",
      "type": "queueTrigger",
      "direction": "in",
      "queueName": "bot-queue",
      "connection": ""
    },
    {
      "type": "bot",
      "name": "$return",
      "direction": "out",
      "botId": "yourbot"
    },
    {
      "type": "http",
      "name": "res",
      "direction": "out"
    }
  ]
}
{% endhighlight %} 

The Function code looks like this.



<div id="thetabs3">
    <ul>
        <li><a href="#tab31">C#</a></li>
        <li><a href="#tab32">Node.js</a></li>
    </ul>

    <div id="tab31">

{% highlight csharp %}
using System;
using System.Net;
using System.Net.Http;
using Microsoft.Azure.WebJobs.Host;

public class BotMessage
{
    public string Source { get; set; } 
    public string Message { get; set; }
}

public static HttpResponseMessage Run(string myQueueItem, out BotMessage message, TraceWriter log)
{
    message = new BotMessage
    { 
        Source = "Azure Functions (C#)!", 
        Message = myQueueItem
    };

    return new HttpResponseMessage(HttpStatusCode.OK); 
}{% endhighlight %}

    </div>
    <div id="tab32">

{% highlight JavaScript %}
module.exports = function (context, myQueueItem) {
    context.log('Sending Bot message', myQueueItem);

    var message = {
        'text': myQueueItem.text,
        'address': myQueueItem.address
    };

    context.done(null, message);
}
{% endhighlight %}

    </div>  
</div>

### Receiving the message back from the Azure Function (Azure Bot Service)

The following snippet of code, shows how to receive the message from the trigger function.

<div id="thetabs2">
    <ul>
        <li><a href="#tab21">C#</a></li>
        <li><a href="#tab22">Node.js</a></li>
    </ul>

    <div id="tab21">

{% highlight csharp %}
switch (activity.GetActivityType())
{
    case ActivityTypes.Trigger:
        // handle proactive Message from function
        var jactivity = JsonConvert.DeserializeObject<JObject>(jsonContent);
        var jobjectvalue = JsonConvert.DeserializeObject<JObject>(jactivity.GetValue("value").ToString());
        var message = JsonConvert.DeserializeObject<Message>(jobjectvalue.GetValue("Message").ToString());
        var messageactivity = (Activity)message.ResumptionCookie.GetMessage();
            
        using (var scope = DialogModule.BeginLifetimeScope(Conversation.Container, messageactivity))
        {
            var client = scope.Resolve<IConnectorClient>();
            var reply = messageactivity.CreateReply();
            reply.Text = $"This is coming back from the trigger! {message.Text}";
            await client.Conversations.ReplyToActivityAsync(reply);
        }
        
        break;
    default:
        break;
}

{% endhighlight %}

    </div>
    <div id="tab22">

{% highlight JavaScript %}
bot.on('trigger', function (message) {
    // handle message from trigger function
    var queuedMessage = message.value;
    var reply = new builder.Message()
        .address(queuedMessage.address)
        .text('This is coming from the trigger: ' + queuedMessage.text);
    bot.send(reply);
});
{% endhighlight %}

    </div>  
</div>


## Conclusion
This template should give you basic idea to enable a proactive scenarios for your bots. Leveraging the power of Azure Bot Service and Azure Functions, one can build really complex system really fast with fault-tolerant, independent pieces.

## Additional resources

* Learn more about [Azure Functions](https://azure.microsoft.com/en-us/documentation/services/functions/){:target="_blank"}
* Learn more about [Azure Functions Storage queue bindings](https://azure.microsoft.com/en-us/documentation/articles/functions-bindings-storage-queue/){:target="_blank"}