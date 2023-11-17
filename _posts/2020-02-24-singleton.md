---
layout: post
title: Как реализовать singleton?
date: 2020-02-24 00:01:30 +0000
categories: [C/C++]
tags: [c/c++, singleton, pattern, solid]
seo:
  date_modified: 2020-02-25 23:06:38 +0500
---

Да.. Мы знаем, что не стоит использовать этот паттерн и сколько бы эту тему не прочесали до меня, я, пожалуй, вставлю свое слово. А именно, слово про реализацию. Хоть я и не скажу ничего нового, но как минимум зафиксирую этот момент для себя. 

Чтож.. В поиске реализаций класса **Singleton** мы обычно обращаемся к статьям, книгам или ищем примеры на гитхабе и, в итоге, натыкаемся на что-то [подобное](https://github.com/JakubVojvoda/design-patterns-cpp/blob/master/singleton/Singleton.cpp):

```cpp
/*
 * C++ Design Patterns: Singleton
 * Author: Jakub Vojvoda [github.com/JakubVojvoda]
 * 2016
 */

class Singleton
{
public:
  Singleton( Singleton const& ) = delete;
  Singleton& operator=( Singleton const& ) = delete;

  static Singleton* get()
  {
    if ( !instance ) {
      instance = new Singleton();
    }    
    return instance;
  }
  
  static void restart()
  {
    if ( instance ) {
      delete instance;
    }
  }
  
  void tell()
  {
    std::cout << "This is Singleton." << std::endl;
  }

private:
  Singleton() {}
  static Singleton *instance;
};

Singleton* Singleton::instance = nullptr;

int main()
{
  Singleton::get()->tell();
  Singleton::restart();
}
```

 И обычно сходу берем его на вооружение. 

 Да. Это типичный пример singleton-а. И на самом деле ничего сверхплохого в данной реализации нет. Но через какое-то время в этом примере можно увидеть некоторый изъян реализации.

 Давайте рассмотрим следующую ситуацию. Вы пишете программу. И в вашей программе должен быть кот. И кот должен быть один. Совсем один. И вы решаете делать его классом одиночкой. Например, по вышеприведенному шаблону. Некоторый дискомфорт можно почувствовать при попытке придумать название класса. Ведь просто *Cat* не подойдет, это ведь кот одиночка. А что тогда подойдет? _SingleCat_? _LonelyKitten_?

![](/assets/img/overly_attached_cat.jpg){: .dark .w-75 .shadow .rounded-10 w='300' }

К тому же, зная про такую штуку как [SOLID](https://en.wikipedia.org/wiki/SOLID) мы увидим, что этот класс будет противоречить первому принципу. А именно [The Single Responsibility Principle (Принцип единственной ответственности)](https://en.wikipedia.org/wiki/Single_responsibility_principle). То есть наш класс будет не только котом, но еще и одиночкой.

А во вторых, у нас может возникнуть потребность создать собаку, гуся, корову и чтобы все они были синглтонами. Или чтобы из трех собак только одна собака была синглтоном, а остальные были обычными.

И что нам нужно делать? Правильно! Делить ответственность! Прибегая к помощи шаблонов создаем класс **Singleton**, *instance* которого будет иметь шаблонный тип. И выглядеть это может примерно следующим образом.


```cpp
#ifndef SINGLETON_HPP_
#define SINGLETON_HPP_

template<class T>
class Singleton {
public:
    static T* Instance();

private:
    Singleton();
    Singleton(Singleton<T> const&);
    Singleton<T>& operator= (const Singleton<T>& t);
    virtual ~Singleton();
    static T* instance;
};

template<class T>
T* Singleton<T>::instance = NULL;

template<class T>
T* Singleton<T>::Instance()
{
    if (instance == NULL) {
        instance = new T;
    }
    return instance;
}

#endif /* SINGLETON_H_ */
```

И в будущем при реализации кота мы будем думать только про кота, не заботясь о том, как делать его синглтоном.

```cpp
#include <Singleton.hpp>

class Cat
{
public:
    Cat() { }
    ~Cat() { }

    void meow()
    {
        std::cout << "Meow" << std::endl;
    }
};
```

А сделать так, чтобы появился кот синглтон (или собака) проще простого.

```cpp
Cat* dog = Singleton<Cat>::Instance();
cat->meow();

Dog* dog = Singleton<Dog>::Instance();
dog->gav();
```

С таким же успехом можно создать одиночный класс std::string или, например, std::vector.

```cpp
std::string* s = Singleton<std::string>::Instance();
```

Зачем? Это просто пример! В реальной жизни это может быть класс для подключения к базе данных.
