---
layout: post
title: Как написать свою библиотеку на Си
date: 2020-08-14 00:23:30 +0000
categories: [C/C++]
tags: [C/C++, Linux]
---

![img-description](https://source.unsplash.com/b1jZUrEUM3g){: w="400" .shadow }
_Photo by [ainr](https://unsplash.com/@ainr) on [Unsplash](https://unsplash.com)_

Если ты встал на путь С/С++ разработчика, то скорее всего (помимо использования стандартной библиотеки - **libc**) рано или поздно вам потребуется занятся разработкой собственных библиотек. Зачем??.. Причин может быть несколько. Например вы написали свою структуру данных или свой алгоритм, и хотите использовать его повторно или распространять. Так же возможно вы написали несколько утилит и все они используют один и тот же кусок кода (например, как часто это бывает, логгер), и будет логично вынести этот кусок кода в отдельный модуль. Поскольку сопровождать такой код будет проще.

# Реализация

И так, давайте начнем с примера. Создадим заголовочный файл **somecode.h**, который будет содержать объявление некоторой функции. Пусть будет простая функция которая разбивает предложение на слова и печатает каждое слово в новой строке. Простой синтетический пример.

```c
#ifndef __SOMECODE__
#define __SOMECODE__

void print_split(char* str);

#endif
```

И создадим файл **somecode.c**, в которой напишем реализацию нашей функции.

```c
#include "somecode.h"
#include <string.h>
#include <stdio.h>

void print_split(char* str)
{
	const char *word = strtok(str, " ");
	while (word != NULL) {
		printf("> %s\n", word);
		word = strtok(NULL, " ");
	}
}
```

Далее создадим файл **main.c**, где будем вызывать нашу функцию.

```c
#include <stdlib.h>
#include "somecode.h"


int main(int argc, char** argv)
{
	if (argc > 1) {
		print_split(argv[1]);
	}
}
```

Давайте для начала скомпилируем это все самым обычным способом для проверки работоспособности.
* из исходных файлов получаем объектные файлы
* из объектных файлов получаем исполняемый файл

```bash
$ gcc -c -Wall -g -o somecode.o somecode.c
$ gcc -c -Wall -g -o main.o main.c
$ gcc -Wall -g -o a.out.1 main.o somecode.o
$ ls -l
$ ls -l
total 40
-rwxr-xr-x 1 ainr ainr 19816 Aug  9 21:22 a.out.1
-rw-r--r-- 1 ainr ainr   123 Aug  9 21:20 main.c
-rw-r--r-- 1 ainr ainr  3576 Aug  9 21:22 main.o
-rw-r--r-- 1 ainr ainr   213 Aug  9 21:21 somecode.c
-rw-r--r-- 1 ainr ainr    72 Aug  9 21:20 somecode.h
-rw-r--r-- 1 ainr ainr  6224 Aug  9 21:21 somecode.o
```

Запускаем исполняемый файл и видим, что все работает.

```bash
$ ./a.out.1 "hello my little pony"
> hello
> my
> little
> pony
```

А теперь рассмотрим пример получения библиотеки и линковки его к исполняемому файлу.
Для компиляции используем вызов **gcc** со следующими опциями.

```bash
$ gcc -Wall -g -shared -fpic -o libsomecode.so somecode.c
```

Из файла с расширением *.c* мы получаем файл с расширением .*so*. И так же обратите внимание, что библиотека имеет префикс **lib**.
Еще мы видим, что появились два дополнительных аргумента **-shared** и **-fpic**. С помощью опции **-shared** мы говорим компилятору, что хотим получить а выходе библиотеку. А опция **-fpic** говорит компилятору, что объектные файлы должны содержать позиционно-независимый код (position independent code), который рекомендуется использовать для динамических библиотек.

Теперь скомпилируем наш исполняемый файл подключив к нему нашу библиотеку. Для этого нужно указать название библиотеки через опцию **-l**.

```bash
$ gcc -Wall -g -o a.out.2 main.c -lsomecode
/usr/bin/ld: невозможно найти -lsomecode
collect2: error: ld returned 1 exit status
```

И опс.. мы получили ошибку... Линкер говорит нам, что он не знает где лежит наша библиотека. С помощью опции **-L** указываем текущую директорию, где лежит наша библиотека и компиляция проходит успешно. 

```bash
$ gcc -Wall -g -o a.out.2 main.c -lsomecode -L.
```

Пытаемся запустить нашу и программу и ловим еще одну ошибку в котором говорится, что в процессе загрузки динамических библиотек отсуствует наша библиотека. 

```bash
$ ./a.out.2 "hello my little pony"
./a.out.2: error while loading shared libraries: libsomecode.so: cannot open shared object file: No such file or directory
```

По умолчанию в операционной системе есть некоторое количество стандартных директорий, где должны располагатся библиотеки. 
Посмотреть этот список можно так. 

```bash
$ ld --verbose | grep SEARCH_DIR | sed "s/\;\ /\n/g"
SEARCH_DIR("=/usr/local/lib/x86_64-linux-gnu")
SEARCH_DIR("=/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu")
SEARCH_DIR("=/usr/lib/x86_64-linux-gnu64")
SEARCH_DIR("=/usr/local/lib64")
SEARCH_DIR("=/lib64")
SEARCH_DIR("=/usr/lib64")
SEARCH_DIR("=/usr/local/lib")
SEARCH_DIR("=/lib")
SEARCH_DIR("=/usr/lib")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib64")
SEARCH_DIR("=/usr/x86_64-linux-gnu/lib");
```

Так же есть возможность задавать дополнительные директории с библиотеками с помощью переменной окружения **LD_LIBRARY_PATH**.
С помощью утилиты ldd посмотрим от каких библиотек зависит наша программа.

```bash
$ ldd a.out.2
        linux-vdso.so.1 (0x00007ffffe4e7000)
        libsomecode.so => not found
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fcf3b680000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fcf3b897000)
```

Добавим в **LD_LIBRARY_PATH** текущую директорию.

```bash
$ export LD_LIBRARY_PATH="$LD_LIBRARY_PATH:$PWD"
```

Видим, что наша библиотека подгрузилась.

```bash
ldd a.out.2
        linux-vdso.so.1 (0x00007fffe2115000)
        libsomecode.so (0x00007f4848a6c000)
        libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f4848860000)
        /lib64/ld-linux-x86-64.so.2 (0x00007f4848a76000)
```

И теперь программа запускается и работает.

```bash
$ ./a.out.2 "hello my little pony"
> hello
> my
> little
> pony
```

Если сравнить два исполняемых файла, то видим, что программа, которая использует динамическую библиотеку имеет меньший размер.

```bash
$ ls -l
total 92
-rwxr-xr-x 1 ainr ainr 19816 Aug  9 21:22 a.out.1
-rwxr-xr-x 1 ainr ainr 17864 Aug  9 21:25 a.out.2
-rwxr-xr-x 1 ainr ainr 18808 Aug  9 21:24 libsomecode.so
-rw-r--r-- 1 ainr ainr   123 Aug  9 21:20 main.c
-rw-r--r-- 1 ainr ainr  3576 Aug  9 21:23 main.o
-rw-r--r-- 1 ainr ainr   213 Aug  9 21:21 somecode.c
-rw-r--r-- 1 ainr ainr    72 Aug  9 21:20 somecode.h
-rw-r--r-- 1 ainr ainr  6224 Aug  9 21:23 somecode.o
```

Это произошло как раз за счет того, что реализация нашей функции теперь лежит за пределами нашего исполняемого файла.
С помощью утилиты **objdump** можем глянуть внутрь бинарных файлов. И увидим, что во втором бинарном файле реализация функции отсутствует, но присутствует в библиотеке.

```bash
$ objdump -t a.out.1 | grep print_split
000000000000119c g     F .text  0000000000000061              print_split
$ objdump -t a.out.2 | grep print_split
0000000000000000       F *UND*  0000000000000000              print_split
$ objdump -t libsomecode.so | grep print_split
0000000000001139 g     F .text  0000000000000061              print_split
```

Разница в нашем случае может и маленькая, но в масштабах десятков и сотен файлов разница будет значительной. 

# Так что же мы сделали?

У нас есть исходные файлы, мы скомпилировали их, получив из них динамическую библиотеку. Далее эту библиотеку можем использовать повторно в других проектах.

```
 --------------------------------   ---------------------------------
|       Заголовочный файл        |  |          Реализация             |
|         (somecode.h)           |  |         (somecode.c)            |
 --------------------------------    ---------------------------------
|                                |  |                                 |
| void print_split(char* string);|  |  void print_split(char* string) |
|                                |  |  {                              |
|                                |  |      // ...                     |
|                                |  |  }                              |
|                                |  |                                 |
 -------------------------------    ---------------------------------
                                                     |
                                                     |   Компилируем
                                                    \ /  (gcc -shared -fpic -o liblog.so ....)
                                                     |
                                    ---------------------------------
                                   |     Динамическая библиотека     |
                                   |           (liblog.so)           |
                                    ---------------------------------
                                                     |
                                                     | (gcc ... -llog)
              ---------------------------------------------  
             |                        |                    |
 -----------------------    -----------------------    -----------------------
|        Утилита1       |  |       Утилита2        |  |      Утилита3         |
 -----------------------    -----------------------    -----------------------
|                       |  |                       |  |                       |
| #include "somecode.h" |  | #include "somecode.h" |  | #include "somecode.h" |
|                       |  |                       |  |                       |
| print_split("...");   |  | print_split("...");   |  | print_split("...");   |
|                       |  |                       |  |                       |
 -----------------------    -----------------------    ----------------------- 
```

Примерно те же действия вы будете делать и под windows и под мак, будут немного другие компиляторы, но идея одна.

{% include embed/youtube.html id='Jcr9jhAysIc' %}
