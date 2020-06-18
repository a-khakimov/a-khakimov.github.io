---
layout: post
title: Простой способ сборки Android/Qt приложений
date: 2020-05-25 00:23:30 +0000
categories: [qt, android, docker]
tags: [qt, android, docker]
seo:
  date_modified: 2020-06-18 09:14:06 +0500
---

Для сборки android приложений написанных на Qt можно воспользоваться двумя способами:
* настроить окружение на компьютере/сервере и собирать у себя локально
* использовать готовые docker-контейнеры (этот способ я для себя обнаружил недавно)

Недостатки первого способа:
* Способ довольно нетривиален и чтобы у вас все получилось нужно приложить некоторое упорство (именно, поэтому как минимум один раз я предлагаю пойти этим путем). Потому, что нужно установить и настроить Android SDK, NDK, Qt Creator, в общем, нужно сделать кучу [телодвижений](https://doc.qt.io/qt-5/android-getting-started.html) чтобы получить конечный apk-файл. 
* И еще один минус вытекающий с такого подхода - если ты, например, будешь переезжать на другой компьютер, тебе придется настраивать окружение в нем заново.

Второй способ заключается в использовании готовых docker контейнеров (например этих - [https://hub.docker.com/r/a12e/docker-qt](https://hub.docker.com/r/a12e/docker-qt)) или приготовить контейнер самому (в этом случае настроить окружение нужно будет только один раз, после чего зафиксировать изменения и использовать его в других проектах).

Для своего [мини-проекта](https://github.com/a-khakimov/SmartTap) я создал четыре докерфайла:

```bash
$ ls Dockerfiles/
android.arm64v8a.Dockerfile
android.armv7.Dockerfile
android.x86_64.Dockerfile
android.x86.Dockerfile
...
```

Которые имеют примерно следующее содержание.


```
FROM a12e/docker-qt:5.13-android_arm64_v8a

RUN git clone https://github.com/a-khakimov/SmartTap.git; \
cd SmartTap/Game; \
mkdir build; \
cd build; \
qmake -r .. ANDROID_EXTRA_LIBS+=$ANDROID_DEV/lib/libcrypto.so \
            ANDROID_EXTRA_LIBS+=$ANDROID_DEV/lib/libssl.so; \
make; \
make install INSTALL_ROOT=../dist; \
cd ..; \
androiddeployqt --input build/android-libSmartTap.so-deployment-settings.json \
                --output dist/ --aab --deployment bundled --gradle --release
```

Что тут происходит:
* Вытягивается нужный образ (a12e/docker-qt:5.13-android_arm64_v8a)
* Забираем из гитхаба проект, который нужно собрать
* И собственно сам процесс сборки

Для выгрузки apk-файла вытягиваем контейнер и копируем из него файл.

Преимущество такого подхода:
* избавили себя настройки окружения
* автоматизировали процесс сборки


