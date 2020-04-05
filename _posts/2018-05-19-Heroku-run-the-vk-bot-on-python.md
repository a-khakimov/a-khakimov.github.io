---
layout: post
title: 'Heroku: run the vk bot on python'
date: 2018-05-19 00:01:27 +0000
categories: [python, vkapi, heroku]
tags: [python, vkapi, heroku, tutorial]
seo:
  date_modified: 2020-02-26 16:23:13 +0500
---

----------------

## Stack

* heroku
* python
* vk api

----------------

## Problem

Running the bot on a personal computer or laptop can lead to periodic crashes for various reasons, often due to problems with the Internet.

----------------

## Solution

One solution is to run the bot on the server. I found a service like [Heroku](https://heroku.com) for myself. It is possible to run up to 5 services for free. My bot is written in **python**.

1. First of all you need to register on **Heroku**. This is the easiest part - can be not to describe.

2. Next you need to download and install Heroku CLI

```bash
wget https://cli-assets.heroku.com/branches/stable/heroku-OS-ARCH.tar.gz
mkdir -p /usr/local/lib /usr/local/bin
tar -xvzf heroku-OS-ARCH.tar.gz -C /usr/local/lib
ln -s /usr/local/lib/heroku/bin/heroku /usr/local/bin/heroku
```

3. Need to log in via console

```bash
heroku login
```

4. Next you need to create an application

```bash
heroku create vkbot --buildpack http://github.com/heroku/heroku-buildpack-python.git
```

5. Application configuration

* Create the files:

**runtime.txt** - python version

```bash
python-3.6.0
```

**requirements.txt** - list libraries, with each new line

```bash
vk
```

**Procfile** - commands for run app

```bash
worker: python3 vkbot.py
```

* Push bot on heroku

```bash
git init
heroku git:remote -a vkbot
git add .
git commit -am "make it better"
git push heroku master
```

* For running app need open site Heroku and make free run for your bot.

![Run heroku app](/static/img/run-heroku-app.png)

* The site has the ability to view logs, it will be useful to analyze the behavior of the bot.

![Heroku app logs](/static/img/logs-heroku-app.png)

* It is also possible to open the console and execute some commands.

![Heroku app logs](/static/img/bash-heroku-app.png)

----------------
