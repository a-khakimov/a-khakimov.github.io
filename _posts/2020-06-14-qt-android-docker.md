---
layout: post
title: Сборка Android/Qt приложений с помощью Docker-контейнеров
date: 2020-05-25 00:23:30 +0000
categories: [Qt, Android, Docker]
tags: [qt, android, docker]
---

![](https://cdn.shopify.com/s/files/1/0099/4220/4513/products/QtAndroidTools_logo_256x.png)

Для сборки android приложений написанных на Qt можно воспользоваться двумя способами:
* настроить окружение на компьютере и собирать у себя локально
* использовать готовые docker-контейнеры с настроенным для сборки окружением

Настройка окружения может оказаться довольно нетривиальным занятием и чтобы у вас все получилось нужно приложить некоторое упорство, поскольку нужно установить и настроить Android SDK, NDK, Qt Creator и т.д. Нужно сделать кучу [телодвижений,](https://doc.qt.io/qt-5/android-getting-started.html) чтобы получить конечный apk-файл (именно, поэтому как минимум один раз я предлагаю пойти этим путем). И если ты, например, будешь переезжать на другой компьютер, тебе придется настраивать окружение в нем заново.

Второй способ заключается в использовании готовых docker контейнеров (например этих - [https://hub.docker.com/r/a12e/docker-qt](https://hub.docker.com/r/a12e/docker-qt)) или приготовить контейнер самому (в этом случае настроить окружение нужно будет только один раз, после чего зафиксировать изменения и использовать его в других проектах).

Для проекта необходимо будет создать докерфайл с примерно следующим содержанием:

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
