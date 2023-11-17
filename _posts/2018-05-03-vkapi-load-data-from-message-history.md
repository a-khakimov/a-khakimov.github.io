---
layout: post
title: 'vkapi: load data from message history'
date: 2018-05-03 00:30:00 +0000
categories: [Python]
tags: [python, vkapi]
---

## Problem

Bot [Public transport of Ekaterinburg](https://vk.com/transportekb) currently has hundreds of active users who every day send him a geo-positions. We decided to collect some statistics and see which areas of the city are the most active. For get it required to upload all the geotags from message history.

## Stack

* python
* vk api

## So, how do it?

First of all, need a method to get the message history. When reading messages, you should consider the following points:
* the maximum number of messages that can be read at a time should not exceed 200
* the message read request must be made no more than once every 0.3 seconds

Note: if you do not comply with these restrictions, the bot can block. For full confidence, I set the data acquisition time to 0.5 seconds.

To get the history you need to use the method **vkapi.messages.getHistory**.  In my case, run in a loop, getting 200 messages and then in a nested loop for check each message existence of a geo-position.

```python
def getAllGeoFromHistory(vkapi, userID, m_count=200, m_offset=0):
    do_it = 'yes'
    while do_it == 'yes':
        message = vkapi.messages.getHistory(user_id=userID, count=m_count, offset=m_offset)
        for message_count in range(m_count):
            if message_count >= message['count']:
                do_it = 'no'
                break
            if message['items'][message_count].get('geo'): # if geo-position finded in message
                coordinates_string = message['items'][message_count]['geo']['coordinates']
                coordiates_words = coordinates_string.split()
                jsonData = { 'x' : coordiates_words[0], 'y' : coordiates_words[1] }
                file.write(str(jsonData) + ',\n') # Write geo-position to json file
                print('ID:', message['items'][message_count]['user_id'], jsonData)
            if message['items'][message_count]['id'] < 2:
                do_it = 'no'
                break
        m_offset = m_offset + m_count
        time.sleep(0.5) # min time should be longer 0.3 s
```

After we learned to take all the geo-positions from the history of messages of a particular user, we need to apply this method to all the dialogs. Here the same restrictions apply:
* request frequency not more than 0.3 seconds
* maximum number of dialogs at once - 200

The principle is that need to take **user_id** from each dialog. For read the dialogs, call the **vkapi.messages.getDialogs** method. The next step is to call the method **getAllGeoFromHistory (vkapi, userID)** for each **user_id**.

```python
def openAllDialogsAndGetGeo(vkapi, max_count=200, cur_offset=0):
    do_it = 'yes'
    while do_it == 'yes':
        dialogs = vkapi.messages.getDialogs(count=max_count, offset=cur_offset)
        for dialog_count in range(max_count):
            if dialog_count  >= dialogs['count']:
                do_it = 'no'
                break
            userID = dialogs['items'][dialog_count]['message']['user_id']
            getAllGeoFromHistory(vkapi, userID)  # get all geo-positions from history          
        cur_offset = cur_offset + max_count
        time.sleep(0.5) # min time should be longer 0.3 s
```

After uploading all the geo-positions, I spread them on the map [OpenStreetMap](http://openstreetmap.ru).

![MAP1](assets/img/0001/1.jpg)

![MAP2](assets/img/0001/2.jpg)

This code can be modified and applied for other purposes. For example, it can be useful to search for specific phrases in messages or search phone numbers in history.

The source code can be found [here](https://github.com/techlinked/vkapi-download-all-message-history).

----------------

## How use it?

* Add token to **.token.json**

```json
{
    "token" : "user_token"
}
```

* Install **python3** and **vk api**

```bash
sudo apt install python3 python3-pip
pip3 install vk
```

* Run **geoget.py**

```bash
python3 geoget.py
```

* The geodata will be recorded to file **data.json**

----------------

