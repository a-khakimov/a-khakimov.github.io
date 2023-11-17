---
layout: post
title: Пару слов об ODB ORM 
date: 2020-06-25 00:23:30 +0000
categories: [Qt, Postgresql, C/C++]
tags: [Qt, PostgreSQL, Database, C/C++]
---

Некоторое время назад встретил на своем пути замечательную ORM-библиотеку [ODB](https://www.codesynthesis.com/products/odb/). Приглянулась она мне довольно подробными [примерами](https://www.codesynthesis.com/products/odb/examples.xhtml) и [документацией](https://www.codesynthesis.com/products/odb/doc.xhtml). А так же мое почтение разработчикам за максимально наглядное изображение, демонстрирующее механизм работы с библиотекой.

![](https://www.codesynthesis.com/products/odb/doc/odb-flow.png)

Далее я решил применить ее в реальном проекте и столкнулся с парочкой проблем о которых напишу тут.

И так, при использовании библиотеки в чистом С++ проекте проблем возникнуть не должно. В моем случае развлечения начались в процессе прикручивания к проекту с использованием Qt библиотек, в частности при генерации кода для структуры с полем **QDateTime** для хранения таймстемпов *(Это все я делал на Ubuntu)*. Структура, для которой нужно было сгенерировать код выглядела примерно так.

```cpp
#include <string>
#include <odb/core.hxx>
#include <QtCore/QDateTime>

#pragma db object
class player
{
public:
    player (const QDateTime datetime,
    	    ...

    QDateTime datetime () const { return timestamp_; }
    ...

private:
    friend class odb::access;
    player () {}

#pragma db id auto
    unsigned long id_;
    QDateTime timestamp_;
    ...
};

```

Окей. В документации указано, что для генерации кода при использовании Qt достаточно использовать опцию **--profile qt/date-time**. Пытаемся сгенерировать:

```bash
$ odb --database pgsql --generate-query --generate-schema --profile qt/date-time ../player.hxx 
```

И ловим ошибку.

```
In file included from <odb-prologue-2>:1:
/usr/include/odb/qt/date-time/pgsql/default-mapping.hxx:8:10: fatal error: QtCore/QDate: Нет такого файла или каталога
    8 | #include <QtCore/QDate>
      |          ^~~~~~~~~~~~~~
compilation terminated.
```

Что не так? Ладно.. Ковыряемся в гугле и пишут, что надо явно указать путь к заголовочным файлам Qt. Указываем.

```bash
$ odb -I/usr/include/x86_64-linux-gnu/qt5/ --database pgsql --generate-query --generate-schema --profile qt/date-time ../player.hxx
```

И ловим огроооомный вывод с ошибками. Он реально огромный и среди большой кучи текста видим, что проскакивают следующие сообщения.

```
/usr/include/c++/9/bits/c++0x_warning.h:32:2: error: #error This file requires compiler and library support for the ISO C++ 2011 standard. This support must be enabled with the -std=c++11 or -std=gnu++11 compiler options.
```

В usage сообщении odb, помнится мне, я видел, что можно указать стандарт С++. И действительно:

```bash
$ odb --help | grep std
--std <version>               Specify the C++ standard that should be used
```

Теперь запускаем с такими опциями. 

```bash
$ odb -I/usr/include/x86_64-linux-gnu/qt5/ --std c++14 --database pgsql --generate-query --generate-schema --profile qt/date-time ../player.hxx 
```

Ии... Опять ошибка!

```
/usr/include/x86_64-linux-gnu/qt5/QtCore/qglobal.h:1187:4: error: #error "You must build your code with position independent code if Qt was built with -reduce-relocations. " "Compile your code with -fPIC (-fPIE is not enough)."
 1187 | #  error "You must build your code with position independent code if Qt was built with -reduce-relocations. "\

```

Вот на этом моменте я прифигел и потратил несколько часов на гугление, курение документации и танцы с бубном. По итогу выяснилось, что нужно было с помощью опции **-x** передать параметр **-fPIC**. Итоговая команда выгоядела так:

```bash
$ odb -x -fPIC -I/usr/include/x86_64-linux-gnu/qt5/ --std c++14 --database pgsql --generate-query --generate-schema --profile qt/date-time ../player.hxx 
```

И после этого нужные файлы сгенерировались.

```bash
$ ls 
player-odb.cxx  player-odb.hxx  player-odb.ixx  player.sql
```

SQL-файл используем для создания таблицы в базе.

```
$ psql --username=ainr --host=localhost --dbname=huidb < player.sql
DROP TABLE
CREATE TABLE
```

Остальные файлы включаем в Qt/C++ проект. Далее пишем код для вставки данных в таблицу:

```cpp
try {
    odb::transaction transaction(db->begin()); // Создаем транзакцию
    player p(ip, datetime, platform);          // Создаем объект нашего класса
    db->persist(p);                            // Подкидываем объект в базу
    transaction.commit();                      // Фиксируем изменения
}
catch (const odb::exception& e) {
    qDebug() << e.what();
}
```

Компилируем, запускаем и видим, что данные были адекватно добавлены в базу данных.

```sql
huidb=> select * from player;
 id |        timestamp        |        ip        | platform 
----+-------------------------+------------------+----------
  1 | 2020-06-25 22:04:40.589 | ::ffff:127.0.0.1 | linux
  2 | 2020-06-25 22:04:44.053 | ::ffff:127.0.0.1 | linux
(2 rows)
```

Код для получения данных из базы выглядит примерно так:

```cpp
try {
    odb::transaction t (db->begin());
    odb::result<player> r(db->query<player>(
        odb::query<player>::id > 5 /* здесь может быть условие */
    ));
    for (auto i = r.begin(); i != r.end(); i++) {
        qDebug() << (*i).ip().c_str() << (*i).datetime() << (*i).platform().c_str();
    }
    t.commit();
}
catch (const odb::exception& e) {
    qDebug() << e.what();
}
```

---

## Вместо вывода

В будущем здесь может появиться описание и решение другой проблемы.
