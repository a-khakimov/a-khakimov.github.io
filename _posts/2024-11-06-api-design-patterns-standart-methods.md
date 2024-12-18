---
layout: post
title: Стандартные методы API
date: 2024-11-06 00:23:30 +0000
categories: [Заметки из книг]
tags: [Паттерны проектирования API, Заметки из книг]
---

Это пост с обзором очередной главы книги **"API Design Patterns"** Дж. Дж. Гивакса, посвящённой стандартным методам API.

**Зачем нужны стандартные методы?** API спроектированное с применением стандартных методов помогает пользователям быстрее освоить его, поскольку используются знакомые методы для стандартных операций. Это особенно важно для RESTful API, где единообразие позволяет работать с различными ресурсами без необходимости каждый раз учить новые подходы.

Одним из важнейших аспектов при проектировании API является согласованность поведения методов. Например, вопрос: что возвращать, если пытаемся удалить несуществующий ресурс? Некоторые выберут `200 OK`, другие — `404 Not Found`. Но что важнее — это обеспечивать одинаковые ответы для подобных случаев, чтобы минимизировать неоднозначность и повысить безопасность.

Не все ресурсы должны поддерживать каждый стандартный метод. Некоторые могут быть избыточными для определённого ресурса. В таких случаях API должен возвращать статус `405 Method Not Allowed`, а не `404 Not Found`, чтобы показать, что проблема не в ресурсе, а в неподдерживаемом методе. Стандартные методы стоит реализовывать для большинства ресурсов, чтобы сохранялась предсказуемость работы API.

**Идемпотентность и побочные эффекты** Стандартные методы должны быть предсказуемыми, не вызывающими побочных эффектов. Методы, такие как Create, не должны запускать дополнительные процессы, вроде отправки email. Метод Get, в свою очередь, должен лишь возвращать данные, не изменяя их состояние.

**Методы**:

- **GET**: Метод Get предназначен для возвращения данных ресурса по его идентификатору, работая по принципу «ключ — значение». Этот метод должен быть идемпотентным и не вызывать побочных эффектов, возвращая один и тот же результат при повторных запросах, если данные не изменялись параллельно.
- **LIST**: Этот метод позволяет получить список ресурсов в коллекции. Он учитывает наличие доступов пользователя к ресурсу и возвращает только те данные, к которым у него есть права. Для повышения эффективности желательно избегать точного подсчёта результатов, предпочитая приближённые оценки. Фильтрация в методе List полезна, так как позволяет избежать затратных операций по извлечению всех данных и их последующей обработке. Рекомендуется использовать строковые выражения для фильтров, которые сервер затем парсит и применяет, избегая сложных и типизированных фильтрующих структур.
- **CREATE**: Используется для создания новых ресурсов. Обычно ID для нового ресурса генерируется на сервере, однако в некоторых случаях можно разрешить пользователям задавать его самостоятельно. В распределенных системах может возникнуть проблема с согласованностью: данные могут быть обновлены, но еще не реплицированы по всем узлам, что может привести к ошибке 404 при извлечении ресурса. API должно поддерживать строгую согласованность, чтобы созданный ресурс был доступен сразу после его создания. Если это невозможно, лучше использовать пользовательские методы или длительные операции для учета времени репликации данных.
- **UPDATE**: Метод Update используется для изменения существующего ресурса с помощью HTTP-метода PATCH, который частично обновляет ресурс, не заменяя его полностью. Он обновляет только те поля, которые явно указаны пользователем. Однако для изменений, связанных с переходом состояния ресурса (например, архивирование), лучше использовать пользовательские методы, такие как `ArchiveChatRoom()`.
- **DELETE**: Этот метод удаляет ресурс из API. Метод может быть идемпотентным или неидемпотентным в зависимости от того, рассматриваем ли мы его как декларативный (удаление ресурса, независимо от его существования) или императивный (удаление только существующего ресурса). В ресурсно-ориентированных API метод Delete обычно неидемпотентен, и попытка удалить несуществующий ресурс приводит к отказу.
- **REPLACE**: Метод Replace используется для полной замены ресурса, включая все его поля, даже если они не известны клиенту. Он использует HTTP-метод PUT и гарантирует, что ресурс будет заменен точно так, как указано в запросе. Этот метод можно использовать для создания новых ресурсов, но он не должен заменять стандартный метод Create, поскольку не позволяет точно различать создание и обновление.

Стандартные методы API подходят для 90% случаев. Обеспечивают понятный интерфейс для пользователей. Для уникальных сценариев можно создавать пользовательские методы. Преимущество стандартных методов — это их предсказуемость и совместимость. Пользовательские методы требуют больше усилий.
