---
layout: post
title: Длительные операции в API (long-running operations) 
date: 2024-11-12 00:23:30 +0000
categories: [Заметки из книг]
tags: [Паттерны проектирования API, Заметки из книг]
---

![](assets/img/memes/lro.jpg){: w="300" .shadow }

Это пост с кратким пересказом очередной главы книги ["API Design Patterns" Дж. Дж. Гивакса](https://www.oreilly.com/library/view/api-design-patterns/9781617295850/), посвящённой длительным операциям.

**Для чего нужны длительные операции?** Синхронный API отлично справляется с быстрыми задачами. Но для ресурсоемких операций, таких как обработка больших данных, простое ожидание ответа может оказаться проблемой. Появляется масса вопросов: сколько ждать? как понять, что запрос обрабатывается? что делать, если результат больше не нужен? Клиент остаётся в подвешенном состоянии и для таких случаев напрашивается другое решение. Вот тут и появляется паттерн длительные операции.

LRO (long-running operations) — это аналог конструкций вроде [Promise и Future](https://en.wikipedia.org/wiki/Futures_and_promises) из языков программирования, с тем отличием, что LRO больше завязано на инфраструктуру. С помощью этого паттерна запускают операцию, отслеживают прогресс выполнения, хранят метаданные (процент выполнения, время) и позволяют клиенту прерывать или возобновлять процесс.

Для реализации LRO создается отдельный ресурс Operation, содержащий ID задачи, ее статус (например, выполнена или приостановлена) и метаданные с информацией о ходе выполнения. Интерфейс API должен поддерживать методы для работы с длительными операциями.

**Способы получения конечного результата**
- Опрос (Polling). Клиент периодически запрашивает статус выполнения с фиксированным или, например, с экспоненциальным интервалом. Это просто и стабильно, но требует регулярных запросов, что увеличивает трафик.
- Ожидание (Wait). Сервер поддерживает открытое соединение и сообщает клиенту о завершении операции. Это экономит трафик, но требует устойчивых подключений, так как возникнут проблемы при потере соединения (например, клиент — мобилка, которая уехала в глухую деревню, где сеть ловит только на крыше соседнего сарая).

**Обработка ошибок.** HTTP-коды ошибок недостаточны для точной обработки на стороне клиента, так как они не различают ошибки на этапе извлечения ресурса и ошибки, вызванные самой операцией. Лучше возвращать код, описание ошибки и дополнительные детали в структуре `OperationError`.

**Отслеживание прогресса.** Прогресс можно получить с помощью метода `GetOperation`, который возвращает актуальные метаданные о ходе выполнения операции. Метаданные, могут содержать: процент выполнения, предполагаемое время завершения или количество обработанных данных.

**Отмена операции** нужна если, например, она запущена по ошибке или результат клиенту больше не нужен — API освобождает ресурсы, удаляя промежуточные данные. Возможность отмены не обязательна для всех операций, так как не всегда приносит выгоду и причем отмена не всегда возможна.

**Приостановка и возобновление операции** полезны, когда выполнение можно временно остановить без потерь. Это тоже не обязательная опция и следует учитывать, что не все операции можно и имеет смысл приостанавливать - это зависит от типа задачи (к примеру, попробуйте приостановить операцию, запускающую ракету, когда ракета уже оторвалась от земли).

**Инспектирование операций** необходимо, чтобы отслеживать состояние списка LRO. Для этого в API можно реализовать метод `GET /operations`, который вернет список операций с возможностью фильтрации по статусу и метаданным. Это позволит, например, увидеть какие операции еще выполняются или приостановлены.

**Хранить или чистить?** Длительные операции (имеется ввиду ресурсы в БД) сразу после завершения становятся практически бесполезными, поскольку весь смысл в их результате. Простое решение, конечно, хранить ресурсы постоянно, но это может быть ресурсозатратно. Ресурсы можно очищать через скользящее окно по полю `expireTime`, удаляя их, скажем, через 30 дней после завершения. Более сложные политики хранения могут привести к усложнению системы, поэтому лучше придерживаться простого, единого срока хранения для всех LRO.