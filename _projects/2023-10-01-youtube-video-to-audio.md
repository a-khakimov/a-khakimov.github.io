---
layout: post
title: Telegram-бот для скачивания аудио из YouTube
date: 2023-10-01 00:23:30 +0000
categories: [Телеграм бот]
tags: [Telegram, YouTube]
---

Хотел бы поделиться с еще одним из моих недавних pet-проектов - telegram-бот, который конвертирует YouTube видео в аудио. Делал я его чисто из-за любопытства. Идея проекта подпадает под главный критерий для моих пет-проектов - реализация не сложная и посильная для одного человека за относительно короткое время.

## Немного про детали реализации

Кажется, что механизм работы будет следующий:
- получили ссылку от пользователя
- скачали видео
- извлекли аудиодорожку
- отправили файл пользователю
- profit! 

Однако, если множество пользователей начнут одновременно дудосить запросами на скачивание, это может привести к длительной задержке и нагрузке на сервер.

Поэтому нужно воспользоваться более хитрым решением, которое позволит обрабатывать большое количество запросов, не нагружая при этом сервер. Я использовал модель основанную на очереди и воркерах, которая представлена на картинке.

![](assets/img/projects/youtube-audio-downloader/scheme.png){: .shadow }

И так:
- создаем очередь, в которую попадает каждая отправленная пользователем ссылка на видео
- эта очередь разбирается воркерами - отдельными фоновыми процессами, которые:
    - берут ссылку из очереди
    - скачивают файл
    - сохраняют в кэш
    - и отправляют файл пользователю

Число воркеров ограничено, чтобы предотвратить перегрузку сервера. Таким образом, даже при большом количестве запросов, система будет работать стабильно, обрабатывая ссылки по мере освобождения воркеров.

Кэш добавлен чтобы избежать повторного скачивания одного и того же видео. Интересный момент заключается в том, что файлы кешируются в Телеграм. Да, я использую Телеграм канал в качестве кэша, что позволяет мне не хранить файлы на моем сервере. Если пользователь пришлет ссылку на ранее загруженный файл, то бот перешлет ему сообщение из канала.

### Стек

- Scala
- Cats Effect 3
- [Telegramium](https://github.com/apimorphism/telegramium)
- [yt-dlp](https://github.com/yt-dlp/yt-dlp)

### Код

Репозиторий с кодом: [https://github.com/a-khakimov/purr](https://github.com/a-khakimov/purr)

### Ссылки

- [t.me/YoutubeAudioDownloadFreeBot](https://t.me/YoutubeAudioDownloadFreeBot) (UPD: на данный момент отключен)

