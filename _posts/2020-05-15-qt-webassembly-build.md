---
layout: post
title: Запукаем приложение написанное на C++/Qt в браузере
date: 2020-05-15 00:01:30 +0000
categories: [Qt, WebAssembly]
tags: [qt, qml, webassembly]
---

Не имея никакого опыта разработки фронтенда веб-приложений, хотя имею поверхностное представление об *html/javascript/css*, я столкнулся с такой штукой как *webassembly* и узнав, что приложение написанное на *c++/qt/qml* можно запустить в браузере решил, что эту технологию обязательно нужно попробовать оседлать. 

## Сборка

Тут вроде все довольно [просто](https://doc.qt.io/qt-5/wasm.html). 
* Качаем [emscripten](https://emscripten.org/docs/introducing_emscripten/index.html)
* Устанавливаем
```bash
$ ./emsdk install sdk-fastcomp-1.38.30-64bit
$ ./emsdk activate --embedded sdk-fastcomp-1.38.30-64bit
$ source emsdk_env.sh
```
* Собираем наше приложение используя **wasm qmake**
```bash
$ ~/.prog/Qt/5.14.2/wasm_32/bin/qmake ..
$ make
```

## Запуск

Для локального запуска можно воспользоваться утилитой **emrun**

```bash
$ emrun --no_browser --port 7373 SmartTap.html
```

Далее в браузере стучимся по адресу *http::/localhost:7373/SmartTap.html*

Для запуска приложения на сервере, или например на *github pages* достаточно вытянуть следующие файлы из сборочной директории и положить туда, откуда клиент сможет получить к ним доступ.

```bash
$ ls build_wasm/
SmartTap.wasm   qtloader.js    qtlogo.svg    
SmartTap.html   SmartTap.js
```

К примеру, мой тестовый проект валяется [тут](https://a-khakimov.github.io/projects/smarttap/SmartTap.html). Да, приложение дейсвительно запускается. Придется конечно немного подождать пока подгрузиться файл с расширением *wasm*, в моем случае он весит примерно 20 мегабайт.

## Все ли так хорошо?

Нет, не все. Например, у меня (на момент написания поста) не работает звук, По всей видимости из-за отсутсвия поддержки *qtmultimedia*. Об ограничениях написано [здесь](https://wiki.qt.io/Qt_for_WebAssembly)

