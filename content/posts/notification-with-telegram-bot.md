---
title: "Free and Secure Messaging on Raspberry Pi with Telegram Bots"
date: 2021-07-25T01:00:00Z
tags: ["telegram", "notifications", "raspberry pi", "home automation", "nodejs"]
---

Telegram is a free and cross-platform instant messaging software. Telegram also
offers a way to set up your own messaging bots. With Telegram bots, it's easy
to set up a tight interface with which you can talk to your home automation Raspberry Pi
on the go without having to expose your whole Raspberry Pi as server.

<em>P.S. Telegram bots can do [a lot more](https://core.telegram.org/bots/), though we only
focus on a small use case here.</em>

# Use Case

1. Push notification: the Raspberry Pi can sent a notification through a bot when home
   conditions change, e.g. door opened, or temperature dropped. The API will even allowed
   sending files like photos or videos, which make for a simple home surveillance system
2. Simple control interface: the Raspberry Pi can respond to a message sent to the bot
   and perform programmed actions, e.g. respond with the current condition at home

# Why use telegram bots

1. It's free! Both telegram the service/app and setting up a telegram bot is free. There are
   also virtually no concerns with rate limiting if you are only using the bot for yourself.
2. It's (more) secure and therefore simpler. Instead of exposing a server to the internet and
   have to worry about defending your tiny server against all sorts of attacks, the
   communication strictly happens through the bot API via long polling.

# Walkthrough

## Setting up the bot

First of all, set up a telegram account if you haven't done so.

Then, head over to [Bot Father](https://t.me/botfather) (yes, really) and send him (it?) a message
`/newbot`. He will respond to help you set up the bot. Note down the token (and don't share it with
anyone!).

## Quick start

First, let's set up a quick project. Here we will use the `node-telegram-bot-api` project.

```bash
npm init
npm i node-telegram-bot-api
npm i -D typescript @types/node-telegram-bot-api ts-node
npx tsc --init
```

Create a new file, `app.ts` and start with

```typescript
import TelegramBot from "node-telegram-bot-api";

const bot = new TelegramBot("<your access token from bot father", {
  polling: true,
});

bot.onText(/.*/, (msg, match) => {
  console.log("chat id", msg.chat.id);
  console.log("content", match![0]);
});
```

This basically logs every message that the bot receives.

Run this script with

```bash
npx ts-node app.ts
```

Now, try to start a conversation and send a message to your bot. I sent a simple 'hello'.
You should see something like this

```
node-telegram-bot-api deprecated Automatic enabling of cancellation of promises is deprecated.
In the future, you will have to enable it yourself.
See https://github.com/yagop/node-telegram-bot-api/issues/319. node:internal/modules/cjs/loader:1092:14
chat id 1234
content /start
chat id 1234
content hello
```

For the rest of this post, I will use `1234` as the chat ID. Substitute with your own.

Note down the chat id - it refers to the chat that you have started with the bot. We can use this as
an authorization mechanism by limiting the bot to only respond to our chat. Let's update
the script to

```typescript
bot.onText(/.*/, (msg, match) => {
  console.log("chat id", msg.chat.id);
  if (msg.chat.id !== 1234) {
    console.log("Ignoring message as chat ID mismatched");
    return;
  }
  console.log("content", match![0]);
});
```

You can test this out by asking your friend to try to contact the bot, or use a different telegram account.
You should see something like this:

```
chat id 2345
Ignoring message as chat ID mismatched
```

Someone can still compromise telegram or your telegram account and thus gain access to your bot, but at least
it's easier to defend against that. In the case that they do gain access, they will also only able to perform
actions that you have programmed the bot to support.

## Sending messages

We will also use this chat ID as the 'channel' to send any push notifications. As a demo, let's set up a simple
pinging notification.

```typescript
const chatId = 1234;

setInterval(() => {
  bot.sendMessage(chatId, `ping ${new Date().toString()}`);
}, 5000);
```

## Responding to messages

We can also program the bot to respond to messages. Put this below the first `onText` code snippet:

```typescript
bot.onText(/^What is ([a-z\s]+)\?/, (msg, match) => {
  console.log("chat id", msg.chat.id);
  if (msg.chat.id !== chatId) {
    console.log("Ignoring message as chat ID mismatched");
    return;
  }
  if (match![1] === "the time") {
    bot.sendMessage(chatId, `The time is ${new Date().toTimeString()}`);
  } else {
    bot.sendMessage(chatId, `I don't know what ${match![1]} is`);
  }
});
```

This sets up a very simple 'AI'. It should be easy to extend this to perform arbitrary actions
based on the messages that the bot receives.

## Complete code:

```typescript
import TelegramBot from "node-telegram-bot-api";

const bot = new TelegramBot("<access token from bot father>", {
  polling: true,
});

const chatId = 1234; // get your chat ID from the log

bot.onText(/^What is ([a-z\s]+)\?/, (msg, match) => {
  console.log("chat id", msg.chat.id);
  if (msg.chat.id !== chatId) {
    console.log("Ignoring message as chat ID mismatched");
    return;
  }
  if (match![1] === "the time") {
    bot.sendMessage(chatId, `The time is ${new Date().toTimeString()}`);
  } else {
    bot.sendMessage(chatId, `I don't know what ${match![1]} is`);
  }
});

bot.onText(/.*/, (msg, match) => {
  console.log("chat id", msg.chat.id);
  if (msg.chat.id !== chatId) {
    console.log("Ignoring message as chat ID mismatched");
    return;
  }
  console.log("content", match![0]);
});

setInterval(() => {
  bot.sendMessage(chatId, `ping ${new Date().toString()}`);
}, 5000);
```
