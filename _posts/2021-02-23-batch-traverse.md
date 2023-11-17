---
layout: post
title: Порционный запуск List[Future]
date: 2021-02-23 00:23:30 +0000
categories: [Scala]
tags: [Scala, Future]
---

![img-description](https://source.unsplash.com/hVcG4UdxNhM){: w="400" .shadow }
_Photo by [ainr](https://unsplash.com/@ainr) on [Unsplash](https://unsplash.com)_

Представим, что у нас есть некоторая функция, для выполнения которой требуется некоторое время.

```scala
def slowCalculation(i: Int)(implicit ec: ExecutionContext): Future[Int] = Future {
  log(s"slowCalculation $i")
  Thread.sleep(1000)
  i
}
```

С помощью `Future.traverse` мы можем применить данную функцию для некоторого списка данных. При этом выполнив конкурентный (это не значит паралельный) запуск фьюч.

```scala
val someData = List(1, 2, 3, 4)
Future.traverse(someData)(slowCalculation)
```

В функции `slowCalculation` вместо `Thread.sleep(1000)` может быть все что угодно. Например, обращение в базу данных или поход в некоторый сервис. Так же может произойти такая ситуация, что размер списка `someData` может быть равен не 4, а 400000. И поскольку `Future` жадная, то у нас запуститься 400000 прожорливых фьюч. Это может привести к непредвиденным последствиям (кстати, это отличный способ заддосить свой внутренний или внешний сервис). 

Чтобы обезопасить себя от такого поведения напишем функцию `batchTraverse` для порционного применения функции к списку данных.

```scala
def batchTraverse[A, B](input: Seq[A], batchSize: Int)(f: A => Future[B]): Future[Seq[B]] = {
  input.grouped(batchSize)
    .map(batch => () => Future.traverse(batch)(f))
    .foldLeft(Future.successful(Seq[B]())) {
      (accF, batchF) => for {
        acc <- accF
        batch <- batchF()
      } yield acc ++ batch
    }
}
```

После чего можем запустить функцию порционно по 10 штук пока не будет обработан весь список.

```scala
Await.result(batchTraverse(0 to 400000, 10)(slowCalculation), Duration.Inf)
```

* Примечание: в ходе экспериментов не забывайте о контексте, в котором будет производиться запуск фьюч. А лучше создать свой `ExecutionContext` с фиксированным размером пула потоков.

```scala
val threadPool = Executors.newFixedThreadPool(20)
implicit val ec: ExecutionContext = ExecutionContext.fromExecutor(threadPool)
```

