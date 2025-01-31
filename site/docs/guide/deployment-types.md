---
prev: ./games.md
next: /advanced/
---

# Long Polling vs. Webhooks

There are two ways how your bot can receive messages from the Telegram servers.
They are called _long polling_ and _webhooks_.
grammY supports both of these two ways, while long polling is the default.

This section first describes what long polling and webhooks actually are, and in turn outlines some of the advantages and disadvantes of using one or the other deployment method.
It will also cover how to use them with grammY.

## Introduction

You can think of the whole webhooks vs. long polling discussion as a question of what _deployment type_ to use.
In other words, there are two fundamentally different ways to host your bot (run it on some server), and they differ in the way how the messages reach your bot, and can be processed by grammY.

This choice matters a lot when you need to decide where to host your bot.
For instance, some infrastructure providers only support one of the two deployment types.

Your bot can either pull them in (long polling), or the Telegram servers can push them to your bot (webhooks).

> If you already know how these things work, scroll down to see how to use [long polling](#how-to-use-long-polling) or [webhooks](#how-to-use-webhooks) with grammY.

## How Does Long Polling Work?

_Imagine you're getting yourself a scoop of ice cream in your trusted ice cream parlor.
You walk up to the employee and ask for your favorite type of ice cream.
Unfortunately, he lets you know you that it is out of stock._

_The next day, you're craving that delicious ice cream again, so you go back to the same place and ask for the same ice cream.
Good news!
They restocked over night so you can enjoy your salted caramel ice cream today!
Yummy._

**Polling** means that grammY proactively sends a request to Telegram, asking for new updates (think: messages).
If no messages are there, Telegram will return an empty list, indicating that no new messages were sent to your bot since the last time you asked.

When grammY sends a request to Telegram and new messages have been sent to your bot in the meantime, Telegram will return them as an array of up to 100 update objects.

```asciiart:no-line-numbers
______________                                   _____________
|            |                                   |           |
|            |   <--- are there messages? ---    |           |
|            |    ---       nope.         --->   |           |
|            |                                   |           |
|            |   <--- are there messages? ---    |           |
|  Telegram  |    ---       nope.         --->   |    Bot    |
|            |                                   |           |
|            |   <--- are there messages? ---    |           |
|            |    ---  yes, here you go   --->   |           |
|            |                                   |           |
|____________|                                   |___________|
```

It is immediately obvious that this has some drawbacks.
Your bot only receives new messages every time it asks, i.e. every few seconds or so.
To make your bot respond faster, you could just send more requests and not wait as long between them.
We could for example ask for new messages every millisecond! What could go wrong…

Instead of deciding to spam the Telegram servers, we will use _long polling_ instead of regular (short) polling.

**Long polling** means that grammY proactively sends a request to Telegram, asking for new updates.
If no messages are there, Telegram will keep the connection open until new messages arrive, and then respond to the request with those new messages.

_Time for ice cream again!
The employee already greets you with your first name by now.
Asked about some ice cream of your favorite kind, the employee smiles at you and freezes.
In fact, you don't get any response at all.
So you decide to wait, firmly smiling back.
And you wait.
And wait.
Some hours before the next sunrise, a truck of a local food delivery company arrives and brings a couple of large boxes into the parlor's storage room.
They read_ ice cream _on the outside.
The employee finally starts to move again.
“Of course we have salted caramel!
Two scoops with sprinkles, the usual?”
As if nothing had happened, you enjoy your ice cream while leaving the world's most unrealistic ice cream parlor._

```asciiart:no-line-numbers
______________                                   _____________
|            |                                   |           |
|            |   <--- are there messages? ---    |           |
|            |   .                               |           |
|            |   .                               |           |
|            |   .     *both waiting*            |           |
|  Telegram  |   .                               |    Bot    |
|            |   .                               |           |
|            |   .                               |           |
|            |    ---  yes, here you go   --->   |           |
|            |                                   |           |
|____________|                                   |___________|
```

> Note that in reality, no connection would be kept open for hours.
> Long polling requests have a default timeout of 30 seconds (in order to avoid a number of [technical problems](https://tools.ietf.org/id/draft-loreto-http-bidirectional-07.html#timeouts)).
> If no new messages are returned after this period of time, then the request will be cancelled and resent—but the general concept stays the same.

Using long polling, you don't need to spam Telegram's servers, and still you get new messages immediately!
Nifty.
This is what grammY does by default when you run `bot.start()`.

## How Do Webhooks Work?

_After this terrifying experience (a whole night without ice cream!), you'd prefer not to ask anyone about ice cream at all anymore.
Wouldn't it be cool if the ice cream could come to you?_

Setting up a **webhook** means that you will provide Telegram with a URL that is accessible from the public internet.
Whenever a new message is sent to your bot, Telegram (and not you!) will take the initiative and send a request with the update object to your server.
Nice, heh?

_You decide to walk to the ice cream parlor one very last time.
You tell your friend behind the counter where you live.
He promises to head over to your apartment personally whenever new ice cream is there (because it would melt in the mail).
Cool guy._

```asciiart:no-line-numbers
______________                                   _____________
|            |                                   |           |
|            |                                   |           |
|            |                                   |           |
|            |         *both waiting*            |           |
|            |                                   |           |
|  Telegram  |                                   |    Bot    |
|            |                                   |           |
|            |                                   |           |
|            |    ---  hi, new message   --->    |           |
|            |   <---    thanks dude     ---     |           |
|____________|                                   |___________|
```

## Comparison

**The main advantage of long polling over webhooks is that it is simpler.**
You don't need a domain or a public URL.
You don't need to fiddle around with setting up SSL certificates in case you're running your bot on a VPS.
Use `bot.start()` and everything will work, no further configuration required.
Under load, you are in complete control of how many messages you can process.

Places where long polling works well include:

- during development on your local machine,
- on all VPS', and
- on hosted “backend” instances, i.e. machines that actively run your bot 24/7.

**The main advantage of webhooks over long polling is that they are cheaper.**
You save a ton of superfluous requests.
You don't need to keep a network connection open at all times.
You can use services that automatically scale your infrastructure down to zero when no requests are coming.
If you want to, you can even [make an API call when responding to the Telegram request](#webhook-reply), even though this has [a number of drawbacks](https://doc.deno.land/https/deno.land/x/grammy/mod.ts#ApiClientOptions).

Places where webhooks work well include:

- on VPS' with SSL certificate,
- on hosted “frontend” instances that scale according to their load, and
- on serverless platforms, such as cloud functions or programmable edge networks.

## I Still Have No Idea What to Use

Then go for long polling.
If you don't have a good reason to use webhooks, then note that there are no major drawbacks to long polling, and—according to our experience—you will spend much less time fixing things.
Webhooks can be a bit nasty from time to time.

Whatever you choose, if you ever run into serious problems, it should not be too hard to switch to the other deployment type after the fact.
With grammY, you only have to touch a few lines of code.
The setup of your [middleware](./middleware.md) is the same.

## How to Use Long Polling

Call

```ts
bot.start();
```

to run your bot with a very simple form of long polling.
It processes all updates sequentially.
This makes your bot very easy to debug, and all behavior very predictable, because there is no concurrency involved.

If you want your messages to be handled concurrently by grammY, or you worry about throughput, check out the section about [grammY runner](/plugins/runner.md).

## How to Use Webhooks

If you want to run grammY with webhooks, you can integrate your bot into a web server.
We therefore expect you to be able to start a simple web server with a framework of your choice.

Every grammY bot can be converted to middleware for a number of web frameworks, including `express`, `koa`/`oak`, and more.
You can import the `webhookCallback` function from grammY to convert your bot to middleware for the respective framework.

<CodeGroup>
 <CodeGroupItem title="TS">

```ts
import express from "express";

const app = express(); // or whatever you're using
app.use(express.json()); // parse the JSON request body

// 'express' is also used as default if no argument is given
app.use(webhookCallback(bot, "express"));
```

</CodeGroupItem>
 <CodeGroupItem title="JS">

```js
const express = require("express");

const app = express(); // or whatever you're using
app.use(express.json()); // parse the JSON request body

// 'express' is also used as default if no argument is given
app.use(webhookCallback(bot, "express"));
```

</CodeGroupItem>
 <CodeGroupItem title="Deno">

```ts
import { Application } from "https://deno.land/x/oak/mod.ts";

const app = new Application(); // or whatever you're using

// make sure to specify the framework you use
app.use(webhookCallback(bot, "oak"));
```

</CodeGroupItem>
</CodeGroup>

Be sure to read [Marvin's Marvellous Guide to All Things Webhook](https://core.telegram.org/bots/webhooks) written by the Telegram team if you consider running your bot on webhooks on a VPS.

### Webhook Reply

When a webhook request is received, your bot can call up to one method in the response.
As a benefit, this saves your bot from making up to one HTTP request per update. However, there are a number of drawbacks to using this:

1. You will not be able to handle potential errors of the respective API call.
   This includes rate limiting errors, so you won't actually be guaranteed that your request has any effect.
2. More importantly, you also won't have access to the response object.
   For example, calling `sendMessage` will not give you access to the message you send.
3. Furthermore, it is not possible to cancel the request.
   The `AbortSignal` will be disregarded.
4. Note also that the types in grammY do not reflect the consequences of a performed webhook callback!
   For instance, they indicate that you always receive a response object, so it is your own responsibility to make sure you're not screwing up while using this minor performance optimization.

If you want to use webhook replies, you can specify the `canUseWebhookReply` option in the `client` option of your `BotConfig` ([API reference](https://doc.deno.land/https/deno.land/x/grammy/mod.ts#BotConfig)).
Pass a function that determines whether or not to use webhook reply for the given request, identified by method.

```ts
const bot = new Bot(token, {
  client: {
    // We accept the drawback of webhook replies for typing status
    canUseWebhookReply: (method) => method === "sendChatAction",
  },
});
```
