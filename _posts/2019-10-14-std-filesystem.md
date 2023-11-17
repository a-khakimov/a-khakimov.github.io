---
layout: post
title: Играемся с std:filesystem [C++17]
date: 2019-02-02 00:01:30 +0000
categories: [C/C++]
tags: [c, c++, filesystem, experimental, c++17]
seo:
  date_modified: 2020-01-28 23:54:15 +0500
---

В C++17 добавлено новое пространство имен [std::filesystem](https://en.cppreference.com/w/cpp/experimental/fs). На мой взгляд он предоставляет удобный интерфейс для работы с файловой системой. Тогда как раньше приходилось складывать строки и вызывать си функции для манипуляций атрибутами файлов.

### Как компилить?

* Установить С++17
* Указать линковщику флаг *-lstdc++fs*

```bash
 g++-7 main.cpp -o main -lstdc++fs
```

### Начинаем эксперементировать

Для удобства я переопределил namespace. (все так делают)
```cpp
namespace fs = std::experimental::filesystem;
```

### Первый пример

Сущность *fs::path* представляет путь к объектам файловой системы. И некоторые его методы.

```cpp
fs::path p {"./path/file.txt"};
std::cout << "parent_path: " << p.parent_path() << std::endl;
std::cout << "extension: " << p.extension() << std::endl;
std::cout << "filename: " << p.filename() << std::endl;
std::cout << "root_name: " << p.root_name() << std::endl;
std::cout << "has_relative_path: " << p.has_relative_path() << std::endl;
std::cout << "has_root_directory: " << p.has_root_directory() << std::endl;
```

На выходе получим информацию о заданном пути и файле.

```
parent_path: "./path"
extension: ".txt"
filename: "file.txt"
root_name: ""
has_relative_path: 1
has_root_directory: 0
```

### Кроссплатформенный разделитель

Раньше для определения разделителя в зависимости от операционной системы приходилось выкручиватся применением условных директив. Сейчас для этой цели можно применить следующий метод.

```cpp
std::cout << "preferred_separator: " << fs::path::preferred_separator << std::endl;
```

Output:

```
preferred_separator: /
```

### Применение перегруженного знака деления для получения полного пути.

Как по мне такой способ удобен тем, что при чтении, код явно говорит нам, что мы работаем с файловой системой уже в привычном нам виде.

```cpp
fs::path root {"/"};
fs::path httpd { "etc/httpd/" };
fs::path sites {"sites-enabled"};
fs::path conf {"site.conf"};

fs::path path_to_site_conf = root / httpd / sites / conf;
std::cout << "path_to_site_conf: " << path_to_site_conf << std::endl;
```

Output:

```
path_to_site_conf: "/etc/httpd/sites-enabled/site.conf"
```

### Рекурсивное создание и удаление каталогов

```cpp
fs::create_directories("banana/cocosa/mango");
fs::create_directories("banana/apple/kiwi");
system("ls -a banana/*");
fs::remove_all("banana");
system("ls -a banana/*");
```

Output:

```
banana/apple:
.  ..  kiwi

banana/cocosa:
.  ..  mango
ls: cannot access 'banana/*': No such file or directory
```
