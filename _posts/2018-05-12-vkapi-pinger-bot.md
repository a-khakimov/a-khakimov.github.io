---
layout: post
title: 'vkapi: pinger-bot'
date: 2018-05-12 03:42:00 +0000
categories: [python, vkapi]
tags: [python, vkapi]
seo:
  date_modified: 2020-02-26 16:23:13 +0500
---

# Pinger-bot

## Problem

It so happened that we wrote a bot and he sometimes fell. This was due to several reasons:
* server was the most the usual notebook, which could hang.
* poor speed of the Internet

So we decided to write another bot that would control the main bot. If the main bot will fall, the pinger-bot will write that the bot fell and need to lift.
If the developer does not raise the bot, the pinger-bot will continue to notify after a specified period of time.

##  Stack

* python
* vk api

## How it works?

I decided to divide the program into two threads. The first thread constantly sends the message " Are you alive?".

```python
# thread for sender
def sender(vkapi, botID, waitTime=150):
    while True:
        print("sender: Are you alive?")
        vkapi.messages.send(user_id=botID, message='Are you alive?')
        time.sleep(waitTime)
```

The second will receive the answer, analyze and if the bot does not respond - notify the developers.

```python
# thread for receiver
def receiver(vkapi, botID, discussionID, waitTime=150):
    checkTimer = 0
    while True:
        if getResponse(vkapi, botID):
            checkTimer = 0            
        elif checkTimer > waitTime:            
            notifyTimer = 0
            while not getResponse(vkapi, botID):
                if notifyTimer > waitTime:
                    notifyTimer = 0
                    notifyDevelopers(vkapi, discussionID, '☠ Bot does not respond ☠')
                notifyTimer += 1
                time.sleep(1)
            checkTimer = 0
        time.sleep(1)
        checkTimer += 1
```

The answer analysis is very simple. You need to get a message, if the message is empty, then return False. If not empty, then check from whom received the message. If the message is from our bot, then return True.


```python
def getResponse(vkapi, botID):
    response = vkapi.messages.get(time_offset=1)
    if response['items']:
        userID = response['items'][0]['user_id']
        readState = response['items'][0]['read_state']             
        if userID == int(botID) and readState == 0:   #if this is my bot
            logging.info("getResponse: Bot alive")
            print("getResponse: Bot alive")
            return True
        else:
            logging.info("getResponse: Bot not alive")
            print("getResponse: Bot not alive")
            return False
```
Well, the function to send an notify to developers in chat.

```python
def notifyDevelopers(vkapi, discussionID, notification):
    print("notifyDevelopers: Bot has fallen")
    vkapi.messages.send(chat_id=discussionID, message=notification)
```

----------------

Source code available in project [ekaterinburg-transport](https://github.com/oybek/ekaterinburg-transport/tree/pinger-bot/pinger-bot)

----------------

## How use it?

Fill in the file **.authdata.json** with the necessary data.
To receive alerts, we created a chat that included project members and a pinger-bot. Therefore, you need to specify the id of the chat.

```json
{
    "app_id" : "id",
    "user_login" : "login",
    "user_password" : "pass",
    "bot_id" : "id",
    "discussion_id" : "id"
}
```

Next, just run the script.

```bash
python3 pinger-bot.py
```

----------------

