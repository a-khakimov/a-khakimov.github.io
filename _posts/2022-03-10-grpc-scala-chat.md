---
layout: post
title: Пишем простой чат с консольным интерфейсом используя трубно-ориентированное программирование с котами
date: 2022-03-10 00:23:30 +0000
categories: [Scala]
tags: [gRPC, Scala, FS2]
---

![](assets/img/grpc.scala.chat.gif){: w="700" : .normal : .shadow }

Если в процессе изучения [gRPC](https://grpc.io/) хотите попрактиковаться с [Bidirectional Streaming](https://grpc.io/docs/what-is-grpc/core-concepts/#bidirectional-streaming-rpc) (или так называемый двунаправленная потоковая передача данных), c запросами в рамках одного соединения, инициированием событий со стороны сервера, то создание простенького чата может быть отличным способом.

Проект будем писать на языке [Scala](https://www.scala-lang.org/) с использованием библиотеки [fs2-grpc](https://github.com/typelevel/fs2-grpc). Будем использовать клиент-серверную архитектуру, где клиенты могут отправлять сообщения на сервер, который будет ретранслировать их всем подключенным клиентам.

## gRPC

Но прежде чем начать, давайте вспомним, что такое [gRPC](https://grpc.io/) и как он связан с [HTTP/2](https://http2.github.io/) не углубляясь в подробности (на эту тему и так достаточно статей).

[gRPC](https://grpc.io/) - это RPC-фреймворк (Remote Procedure Call), который позволяет создавать клиент-серверные приложения для обмена данными. gRPC использует под капотом протокол [HTTP/2](https://http2.github.io/), который позволяет ускорить передачу данных, уменьшить объем передаваемых данных и снизить задержку. Важно упомянуть о том, что gRPC использует [Protobuf](https://protobuf.dev/) чтобы определить методы и структуру сообщений с помощью специального языка описания интерфейсов, а затем сгенерировать код для работы с этими сообщениями на различных языках программирования. Protobuf обеспечивает эффективную сериализацию/десериализацию данных в компактный бинарный формат.

## Механизм работы чата

### Ограничения

Во первых, чтобы не усложнять проект, я решил делать чат консольным и не хранить сообщения на стороне сервера.

### Bidirectional Streaming

Поговорим про механизм работы чата с Bidirectional Streaming в gRPC. Процесс обмена сообщениями будет работать таким образом, что сервер и клиент обмениваются потоками сообщений в рамках одного соединения. Клиент отправляет событие, сервер его получает, обрабатывает, а затем отправляет ответное событие. Клиент и сервер обмениваются сообщениями асинхронно. Таким образом, данные передаются между клиентом и сервером в реальном времени и в обе стороны.

![](https://habrastorage.org/getpro/habr/upload_files/433/a6c/87f/433a6c87f5957f4ab5abb4f154a38f30.png)

### Мультикастинг событий внутри сервера

Когда клиенты подключаются к серверу, каждый из них может отправлять события на сервер. Однако, возникает проблема - как переслать сообщения от одного клиента остальным подключенным клиентам.

![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/a2b/c3e/f65/a2bc3ef65864f09a938ff6a565500ba6.png)

Для решения этой проблемы можно использовать механизм мультикастинга с помощью топика. Топик - это объект, который позволяет отправлять сообщения одновременно нескольким подписчикам. То есть, если один клиент отправляет событие, то полученное событие на стороне сервера будет направлено на этот топик, а оттуда автоматически пересылается всем клиентам, подписанным на этот топик.

![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/cf5/63d/eaf/cf563deaf2132dd87b66a356ffca66cb.png)

Для реализации мультикастинга я использовал [Topic](https://fs2.io/#/concurrency-primitives?id=topic) из библиотеки [fs2](https://fs2.io/) (Functional Streams for Scala).


Таким образом, визуально механизм взаимодействия клиентов в сервером выглядит примерно так.

![](https://habrastorage.org/r/w1560/getpro/habr/upload_files/e73/6ee/1f0/e736ee1f0dd2f83b982d379ea81cc853.png)
_Клиент генерирует событие и отправляет на сервер, который, в свою очередь, раскидывает это событие по клиентам_

## Реализация

Для языка __Scala__ есть несколько библиотек для работы с _gRPC_. Я использую [fs2-grpc](https://github.com/typelevel/fs2-grpc), который является оберткой над [ScalaPB](https://scalapb.github.io/docs/grpc/) и сделана на основе функциональной библиотеки для работы со стримами - [fs2](https://fs2.io/).

[fs2-grpc](https://github.com/typelevel/fs2-grpc) поддерживает все типы RPC-вызовов - [Unary](https://grpc.io/docs/what-is-grpc/core-concepts/#unary-rpc), [Server Streaming](https://grpc.io/docs/what-is-grpc/core-concepts/#server-streaming-rpc), [Client Streaming](https://grpc.io/docs/what-is-grpc/core-concepts/#client-streaming-rpc) и [Bidirectional Streaming](https://grpc.io/docs/what-is-grpc/core-concepts/#bidirectional-streaming-rpc). Она также предоставляет механизмы обработки ошибок и управления ресурсами, такие как [Resource](https://typelevel.org/cats-effect/docs/std/resource) и [Bracket](https://typelevel.org/cats-effect/docs/2.x/typeclasses/bracket). [fs2-grpc](https://github.com/typelevel/fs2-grpc) интегрируется со стеком функциональных библиотек для работы с эффектами (cats-effect, zio, monix). В моем примере используется [Cats Effect 3](https://typelevel.org/cats-effect/).

### Proto

И так, приступим. В первую очередь нужно накидать прото-файл, в котором опишем контракт взаимодействия клиента и сервера.

Создадим некоторый `ChatService` с методом `eventsStream`, у которого на входе и на выходе потоковые данные с типом `Events` (то есть будем события через стримы туда-сюда делать).

```proto
service ChatService {

  rpc eventsStream(stream Events) returns (stream Events) { }
}
```

`Events` содержит данные обернутые в тип события, которые могут быть инициированы как на стороне клиента, так и сервера (в нашем случае только на стороне клиентов).

```proto
message Events {

  oneof event {
    Login    client_login    = 1;
    Logout   client_logout   = 2;
    Message  client_message  = 3;
    Shutdown server_shutdown = 4;
  }
```

### Реализация сервера

Ранее мы говорили, что сервер должен получать события от клиентов и транслировать их остальным клиентам.

После компиляции прото-файла будет сгенерирован базовый код для работы с _gRPC_, среди которого будет интерфейс `ChatServiceFs2Grpc`. Он должен быть имплементирован на стороне сервера. Моя реализация имеет следующий вид.

```scala
object ChatService {

  def apply[F[_]: Concurrent: Console](
      eventsTopic: Topic[F, Events]
  ): ChatServiceFs2Grpc[F, Metadata] = new ChatServiceFs2Grpc[F, Metadata] {

    val eventsToClients: Stream[F, Events] =
      eventsTopic
        .subscribeUnbounded
        .evalTap(event => Console[F].println(s"From topic: $event"))

    override def eventsStream(
        eventsFromClient: fs2.Stream[F, Events],
        ctx: Metadata
    ): fs2.Stream[F, Events] = {
      eventsToClients.concurrently(
        eventsFromClient
          .evalTap(event => Console[F].println(s"Event from client: $event"))
          .evalMap(eventsTopic.publish1)
      )
    }
  }
}
```

Мы видим метод `eventsStream`, который описывали в proto-файле. Из потока `eventsFromClient` получаем события от клиентов. На выходе отдаем некоторый поток событий `eventsToClients`. Если посмотреть выше, то видим, что `eventsToClients` это подписка на топик `eventsTopic: Topic[F, Events]`, в который публикуются события от клиента для отправки остальным клиентам.

### Сборка и запуск сервера

Собираем все компоненты, которые представляют собой основу серверного приложения.

```scala
object ChatServerApp extends IOApp {

  private def runServer(service: ServerServiceDefinition): IO[Nothing] = {
    NettyServerBuilder
      .forPort(50053)
      .keepAliveTime(5, TimeUnit.SECONDS)
      .addService(service)
      .resource[IO]
      .evalMap(server => IO(server.start()))
      .useForever
  }

  override def run(args: List[String]): IO[ExitCode] = for {
    topic <- Topic[IO, Events]
    serviceResource: Resource[IO, ServerServiceDefinition] =
      ChatServiceFs2Grpc.bindServiceResource[IO](ChatService(topic))
    _ <- serviceResource.use(runServer)
  } yield ExitCode.Success
}
```

В функции `runServer` создается и запускается новый сервер с помощью `NettyServerBuilder`, который прослушивает порт 50053. `NettyServerBuilder` предоставляется библиотекой gRPC для создания серверов, использующих Netty в качестве транспорта и позволяет настроить параметры сервера (порт, keepAliveTime и т.д.)

В методе `run` создается топик, который будет использоваться для мультикастинга событий по клиентам. Создаем инстанс сервиса `ChatService` и биндим его к серверу. Затем запускаем наш сервер.

```bash
$ sbt "runMain org.github.ainr.chat.server.ChatServerApp"
```

В итоге, когда сервер запущен, клиенты смогут подключаться к нему, отправлять сообщения и получать их в режиме реального времени.

### Реализация клиента

Что должен делать клиент? Клиент может показаться чутка сложнее, но на самом деле тут тоже все просто. Клиент делает несколько простых вещей:
- Читает ввод в консоль
- Отправляет события серверу
- Получает события от сервера и обрабатывает их
- Печатает полученные сообщения в консоль

Со стороны клиента тоже все сделано на стримах (Stream).

#### Чтение ввода из консоли

Для чтения ввода из консоли снова прибегаем к помощи стримов. Создаем класс `InputStream` с методом `read`, который возвращает поток сообщений напечатанных клиентом - `Stream[F, String]`.

```scala
object InputStream {

  def apply[F[_]: Async: Console](bufSize: Int): InputStream[F] = {

    new InputStream[F] {

      override def read: Stream[F, String] = {
        fs2.io
          .stdinUtf8(bufSize)
          .through(fs2.text.lines)
          .evalTap(erase)          // удалить из консоли ввод
          .filter(_.nonEmpty)      // фильтруем пустые строки
      }

      private def erase: PartialFunction[String, F[Unit]] = {
        _ => Console[F].print("\u001b[1A\u001b[0K")  // удаляет то, что мы напечатали в консоль путем ввода спец-символов
      }
    }
  }
}
```

По коду видно, что он берет поток символов, преобразует их в строки и фильтрует пустые. Магическим может показаться только лишь  метод `erase`, который печатает что-то непонятное в консоль.

На самом деле никакой магии нет. Все, что он делает - это удаляет то, что мы напечатали в консоль путем ввода [спец-символов ANSI](https://demmel.com/ilcd/help/JavaDocumentation/generalInfo/ILCDANSISupport/java-standard-ansi-sequences.htm) чтобы сообщения не дублировались.

#### Логика клиента

Далее введенный пользователем в консоль текст нужно преобразовать в тип события `Event` и отправить серверу.

В целом, логика клиента довольно простая и описана путем композиции стримов в методе `start`. Здесь снова фигурирует `chatService: ChatServiceFs2Grpc[F, Metadata]` с методом `eventsStream` сгенерированный библиотекой `fs2-grpc` на вход которого отправляем события из консоли (`InputStream`), генерируемые пользователем.

```scala
object ChatClient {

  def apply[F[_]: Concurrent: Console](
      clientName: String,
      inputStream: InputStream[F],
      chatService: ChatServiceFs2Grpc[F, Metadata]
  ): ChatClient[F] = new ChatClient[F] {

    private val grpcMetaData = new Metadata() // empty

    override def start: F[Unit] = {
      chatService
        .eventsStream(
          login(clientName) ++ inputStream.read.through(handleInput),
          grpcMetaData
        )
        .through(processEvent)  // обрабатываем полученные события от сервера
        .through(writeToConsole)   // пишем в консоль
        .compile
        .drain
    }

private def login(clientName: String): fs2.Stream[F, Events] =
  fs2.Stream(Events(ClientLogin(Login(clientName))))

    // ...
```

Metadata в gRPC - это способ передачи дополнительных метаданных между клиентом и сервером, которые представляет собой пары ключ-значение и могут быть добавлены к любому запросу.

На выходе `eventsStream` ловим события с сервера, сгенерированные другими клиентами, обрабатываем их методом `processEvent`, который преобразовывает события в строки.

```scala
private def processEvent: Pipe[F, Events, String] =
  _.map { data =>
    data.event match {
      case event: ClientLogin   => s"${Color.Green(event.value.name).overlay(Bold.On)} entered the chat."      case event: ClientLogout  => s"${Color.Blue(event.value.name).overlay(Bold.On)} left the chat."      case event: ClientMessage => s"${Color.LightGray(s"${event.value.name}:").overlay(Bold.On)} ${event.value.message}"
      case _: ServerShutdown    => s"${Color.LightRed("Server shutdown")}"
      case unknown              => s"${Color.Red("Unknown event:")} $unknown"
    }
  }
```

Для форматированного вывода текста в консоли используется библиотека [fansi](https://github.com/com-lihaoyi/fansi) от  [lihaoyi](https://www.lihaoyi.com/), предназначенная для работы с цветами и стилями текста в консольном приложении. Она позволяет добавлять цветовые и стилевые эффекты к тексту, что делает консольный вывод более информативным и привлекательным. Далее сообщения будут напечатаны в консоль методом `writeToConsole`.

### Сборка и запуск клиента

Собираем все компоненты, которые представляет собой основу клиентского приложения.

```scala
object ChatClientApp extends IOApp {

  private def buildChatService(channel: Channel): Resource[IO, ChatServiceFs2Grpc[IO, Metadata]] =
    ChatServiceFs2Grpc.stubResource[IO](channel)

  private def resources: Resource[IO, ChatServiceFs2Grpc[IO, Metadata]] =
    NettyChannelBuilder
      .forAddress("127.0.0.1", 50053)
      .usePlaintext()
      .resource[IO]
      .flatMap(buildChatService)

  override def run(args: List[String]): IO[ExitCode] =
    resources.use { chatServiceFs2Grpc =>
      ChatClient(
        args.headOption.getOrElse("Anonymous"),
        InputStream[IO](bufSize = 1024),
        chatServiceFs2Grpc
      ).start
    }.as(ExitCode.Success)
}
```

`NettyChannelBuilder` - это класс, предоставляемый библиотекой gRPC для создания клиентов, использующих `Netty` в качестве транспорта. Он позволяет настроить параметры клиента.

В функции `buildChatService` создается ресурс, который представляет собой клиент для обращения к серверу чата. Для его создания используется метод `stubResource` из `ChatServiceFs2Grpc`.

Запускаем клиент через sbt, передав в аргументы имя клиента.

```bash
$ sbt "runMain org.github.ainr.chat.client.ChatClientApp Username"
```

И можем общаться :)

![](assets/img/grpc.scala.chat.gif){: w="700" : .normal : .shadow }

## Вместо заключения

Создание небольших, простых проектов - это отличный способ попрактиковаться и углубить свои знания в технологиях. Это может быть что-то, что вы можете написать быстро и без особых усилий, но в то же время дает возможность изучить какой-то новый аспект технологии или языка программирования.

Простые проекты могут быть очень разнообразными. Например, вы можете написать небольшой веб-сервер, создать небольшую игру, написать скрипт для автоматического сбора данных, или же написать чат на базе gRPC, как мы обсуждали ранее.

Преимущество создания небольших проектов заключается в том, что вы можете более глубоко изучить технологию и применить знания на практике. Вы также можете быстро увидеть результат своей работы и получить удовлетворение от завершения проекта.

Не бойтесь начинать с чего-то простого и постепенно увеличивать сложность - это поможет вам стать более опытным и уверенным программистом.

## Исходники

Код проекта можно посмотреть на гитхабе - [https://github.com/a-khakimov/simple-fs2-grpc-chat](https://github.com/a-khakimov/simple-fs2-grpc-chat).

{% include embed/youtube.html id='Y0tQIVXPwh0' %}
