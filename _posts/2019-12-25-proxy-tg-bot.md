---
layout: post
title: Поднимаем телеграм бота через прокси
date: 2019-12-25 00:01:30 +0000
categories: [Telegram]
tags: [telegram, proxy, bot, tor, linux]
seo:
  date_modified: 2020-02-05 09:54:07 +0500
---

## Проблема

Проблема простая - телеграм заблокирован, а бота поднять нужно. Возможно это нужно сделать на виртуальном сервере или на Raspberry Pi - без разницы, проделывать нужно будет примерно одинаковые шаги:
* установить **tor** и прокси (в данном случае **privoxy**)
* задать необходимые конфигурации
* настроить и запустить бота

## Установка и конфигурация

Устанавливаем tor и pivoxy. На Ubuntu это делается просто.

```bash
$ apt install tor
$ apt install privoxy
```

Далее нужно залезть в конфигурации tor */etc/tor/torrc* и раскомментировать следующую строку.

```
SOCKSPort 127.0.0.1:9050
# SOCKSPolicy accept 
```

Задать следующие конфигурации pivoxy в файле */etc/privoxy/config*.

```
listen-address 127.0.0.1:8118
forward-socks5t / localhost:9050 .
```

Перезапустить сервисы.

```bash
$ sudo service privoxy restart 
$ sudo service tor restart
```

Убедиться, что открыты сконфигурированные порты для tor и privoxy.

```bash
$ sudo netstat -tlnp | grep tor
tcp        0      0 127.0.0.1:9050          0.0.0.0:*               LISTEN      2279/tor            
tcp        0      0 127.0.0.1:9051          0.0.0.0:*               LISTEN      2279/tor            
$ sudo netstat -tlnp | grep privoxy
tcp        0      0 127.0.0.1:8118          0.0.0.0:*               LISTEN      2306/privoxy        
tcp6       0      0 ::1:8118                :::*                    LISTEN      2306/privoxy   
```

## Проверка работоспособности

Сначала проверим работоспособноть на curl. Задаем переменные окружения:

```bash
$ export http_proxy=http://localhost:8118/
$ export https_proxy=$http_proxy
```

Делаем запрос и видим, что подключение есть.

```bash
$ curl -i -X GET https://api.telegram.org/bot<token>/getMe 
HTTP/1.1 200 Connection established

HTTP/1.1 200 OK
Server: nginx/1.16.1
Date: Sat, 11 Jan 2020 18:21:24 GMT
Content-Type: application/json
Content-Length: 107
Connection: keep-alive
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
Access-Control-Allow-Origin: *
Access-Control-Allow-Methods: GET, POST, OPTIONS
Access-Control-Expose-Headers: Content-Length,Content-Type,Date,Server,Connection

{"ok":true,"result":{"id":458886928,"is_bot":true,"first_name":"bot","username":"Bot"}}
```

## Поднимаем бота

Здесь все зависит от того на каком языке и с применением каких библиотек вы делаете бота. В моем случае, на scala опциональное использование прокси выглядит примерно следующим образом.

```scala
object Application extends App 
{
  val bot = if (proxyStatus == "enable") {
    new TgProxyBot(token, host, port)
  } else {
    new TgBot(token)
  }

  bot.run()
  // ...
}
```


