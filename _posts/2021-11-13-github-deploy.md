---
layout: post
title: Деплой приложения из github на виртуальный хостинг
date: 2021-11-13 00:23:30 +0000
categories: [CI/CD]
tags: [github, ci/cd]
---

![img-description](https://source.unsplash.com/VNOFgRMyons){: w="400" .shadow }
_Photo by [ainr](https://unsplash.com/@ainr) on [Unsplash](https://unsplash.com)_

Так сложилось, что мои пет-проекты хранятся на гитхабе, а разворачиваю я их на арендованном виртуальном сервере (VPS).

В этом посте хочу вкратце описать то, как я делаю настройку деплоя и запуска/перезапуска приложения.

## Настройки на стороне сервера

Как правило, для каждого приложения я создаю отдельного пользователя. Для этого нужно подключится к серверу от рута и запустить следующую команду.

```bash
$ adduser botbek
```

Для управления приложением я использую стандартный линуксовый менеджер служб **systemd**, поскольку через него можно достаточно быстро и гибко настроить запуск/перезапуск/контроль приложения.

Для добавления нового сервиса нужно создать файл, в моем случае **/etc/systemd/system/botbek.service**, с примерно следующим содержимым.

```bash
Description=botbek
# Указываем набор зависимостей сервиса
Requires=postgresql.service

[Service]
Type=simple
User=botbek
Group=botbek
# Команда для запуска
ExecStart=scala /home/botbek/builds/botbek.jar
ExecStop=/bin/kill -HUP $MAINPID
Restart=on-abort
# Набор переменных окружений
Environment=TELEGRAM_TOKEN=Foo

TimeoutSec=300
Restart=always
```

Видно, что я указал необходимые зависимости, переменные для запуска приложения, а также команды для запуска и остановки.

И последний штрих - нужно дать возможность нашему пользователю вызывать `systemctl` . Для этого нужно выдать права поправив файлик __sudoers__. Это делается через команду **visudo**
 
 ```bash
$ visudo
```

Где нужно добавить строку

```
botbek ALL=NOPASSWD: /usr/bin/systemctl status botbek.service, /usr/bin/systemctl start botbek.service, /usr/bin/systemctl restart botbek.service
```

Где говорим, что пользователь **botbek** может вызывать systemctl для проверки статуса и старта/рестарта приложения.

## Настройки на стороне GitHub

В репозитории проекта, для настройки/запуска пайплайнов существует **Github Actions**. В зависимости от того на каком языке вы пишете настройка может отличаться, но суть одна - нужно собрать проект, (опционально) прогнать тесты, загрузить сборку на сервер и при необходимости запустить/перезапустить приложение. В моем случае для проекта написанного на языке **Scala** у меня будет создан файл `.github/workflows/scala.yml` в который добавляю следующие правила.

```yaml
name: Scala CI  
  
on:  
  push:  
    branches: [ master ]  
  pull_request:  
    branches: [ master ]  
  
jobs:  
  build:  
  
    runs-on: ubuntu-latest  
  
    steps:  
      - uses: actions/checkout@v3  
  
      - name: Set up JDK 17  
        uses: actions/setup-java@v2  
        with:  
          java-version: '17'  
          distribution: 'temurin'  
  
      - name: Assembly  
        run: sbt assembly  
  
      - name: Deploy to remote  
        uses: garygrossgarten/github-action-scp@release  
        with:  
          local: target/scala-2.13/botbek.jar  
          remote: /home/botbek/builds/botbek.jar  
          host: ${{ secrets.SSH_HOST }}  
          username: ${{ secrets.SSH_USER }}  
          password: ${{ secrets.SSH_PASSWORD }}
```

По содержимому видно, что пайплайн запускается в ветке **master** при каждом пуше в него. _Джоба_ состоит из трех шагов:
- сначала настраивается java с требуемой версией
- далее выполняется сборка
- и, собственно, деплой на виртуальный сервер по **scp** 

Так же видим, что для подключения к серверу используются секреты - хост (`SSH_HOST`), пользователь (`SSH_USER`) и пароль (`SSH_PASSWORD`). Их можно добавить в настройках репозитория в следующей вкладке `Settings -> Security -> Secrets -> Actions`.

## Проверка

После успешного завершения пайплайна, подключимся на сервер по _ssh_.

```bash
$ ssh botbek@host.com
```

И проверим, что файл был успешно загружен.

```bash
botbek@host:~$ ls builds/
  botbek.jar
```

Запускаем и проверяем, что служба активна

```bash
$ systemctl start botbek.service
$ systemctl status botbek.service
* botbek.service - botbek
     Loaded: loaded (/etc/systemd/system/botbek.service; static; vendor preset: enabled)
     Active: active (running) since ...; 2 weeks 5 days ago
   Main PID: 126095 (bash)
      Tasks: 22 (limit: 1108)
     Memory: 256.7M
     CGroup: /system.slice/botbek.service
             |-126095 bash /usr/bin/scala /home/botbek/builds/botbek.jar
```

