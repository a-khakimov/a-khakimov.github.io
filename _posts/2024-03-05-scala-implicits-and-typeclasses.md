---
layout: post
title: Имплиситы и тайпклассы в Scala
date: 2024-03-05 00:23:30 +0000
categories: [Scala]
tags: [Implicits, Typeclasses]
math: true
mermaid: true
---

![](assets/img/scala-implicits-and-typeclasses/mem0.png){: .w-50 .shadow }

> Статья, в большей степени, будет интересна для начинающих скалистов и по сути является переработанным конспектом лекции с добавлением парочки мемасов и практических примеров для лучшей перевариваемости (надеюсь, это действительно так). А еще стоит отметить, что все примеры кода написаны на Scala 2.

## **План у нас такой**

- [ ] [Implicit conversions](#implicit-conversions)
- [ ] [Implicit parameters](#implicit-parameters)
- [ ] [Implicit classes](#implicit-classes)
- [ ] [Type classes](#type-classes)
- [ ] [Simple type classes](#simple-type-classes)

## **Implicit conversions**

И так, давайте же сразу начнем с примера! Допустим, у нас есть такой код. Вопрос - скомпилируется ли он?

```scala
val x: String = 123
```

Конечно же нет! Мы получим ошибку, поскольку пытаемся присвоить строковому типу численный тип и поэтому компилятор бьет нас по рукам - говорит, что так делать нельзя:

```
[error] ...: type mismatch;
[error]  found   : Int(123)
[error]  required: String
[error]   val x: String = 123
[error]                   ^
[error] one error found
[error] (Compile / compileIncremental) Compilation failed
```

Но в Scala есть способ сделать так, чтобы такой код скомпилировался. Нам нужно прибегнуть к использованию механизма неявных преобразований. Давайте посмотрим на следующий пример:

```scala
implicit def intToString(x: Int): String = x.toString

val x: String = 123 // будет вызван intToString
```

 Такой код скомпилируется? Да! Что же тут происходит? У нас есть функция, которая помечена ключевым словом `implicit`, из-за чего она может вызываться неявно.

```scala
implicit def func(param: A): B = ???
```

Неявные преобразования могут носить произвольные названия. Но вы спросите - она же вызывается неявно, зачем ей название? Название неявной функции играет роль только в двух ситуациях:
- если вы хотите вызывать его явно
```scala
implicit def intToString(x: Int): String = x.toString
val x: String = intToString(123)
```
- если нужно определить, какие неявные преобразования доступны в том или ином месте программы когда делаем импорт
```scala
object my_implicits {
    implicit def intToString(x: Int): String = x.toString
}
import my_implicits.intToString
val x: String = 123
```

Таким образом, для определения неявных функций
- Нужно использовать ключевое слово **`implicit`**
- Это функция и должна быть объявлена внутри трейта/класса/объекта/метода (главное, что она не может быть на верхнем уровне)
- В списке аргументов должен быть только **один параметр**
```scala
// Если неявная функция будет принимать два или более аргументов,
// то она не будет вызываться неявно
implicit def func(argA: A, argB: B): C = ???
```


### Implicit scopes and priorities

Компилятор будет использовать только те неявные преобразования, которые находятся в области видимости. Поэтому, чтобы обеспечить доступность имплиситных функций, нужно каким то образом поместить их в область видимости. 

Рассмотрим ключевые моменты, на которые стоит обратить внимание

#### Local scope

Неявные функции можно определить в текущей области видимости, например, внутри метода или объекта. Такие функции будут иметь приоритет над неявными функциями из других областей видимости.

```scala
object Example {  
  
  implicit def intToString(x: Int): String = x.toString  
  
  val x: String = 123  
}
```

Стоит отметить, что если объявить две неявных функции с одинаковыми сигнатурами, то в этом случае (ожидаемо) получим ошибку компиляции, поскольку компилятор не знает какую из них использовать для преобразования.

```scala
object Example {  
  
  implicit def intToString1(x: Int): String = x.toString  
  implicit def intToString2(x: Int): String = x.toString  
  
  val x: String = 123  
}

// Получим ошибку
// [error] Note that implicit conversions are not applicable because they are ambiguous:
// [error]  both method intToString1 in object Example of type (x: Int): String
// [error]  and method intToString2 in object Example of type (x: Int): String
```

#### Imports

Неявные функции, импортированные в текущую область видимости, также доступны для использования. Это позволяет управлять доступностью неявных преобразований и параметров на уровне отдельных файлов или блоков кода.

```scala
object ExternalImplicits {  
    implicit def intToString(x: Int): String = x.toString  
}  

object Example {  
  
  import ExternalImplicits.intToString  
  // ИЛИ импортируем все
  import ExternalImplicits._  
  
  val x: String = 123  
}
```

#### Объекты-компаньоны

Так же компилятор будет искать неявные функции в объекте-компаньоне типа, для которого происходит преобразование, или для типа параметра функции. Это означает, что если вы определите неявную функцию в объекте-компаньоне класса `Currency`, то неявная функция будет доступна везде, где доступен `Currency`.

```scala
trait Currency  
case class Dollar(amount: Double) extends Currency  
case class Euro(amount: Double) extends Currency  
  
object Currency {  
    implicit def euroToDollar(euro: Euro): Dollar = Dollar(euro.amount * 1.13)  
}

object Example extends App {  
  
    val dollar: Dollar = Euro(100) // euroToDollar  
}
```

Если определены две неявные функции с одинаковой сигнатурой - одна в объекте компаньоне, а другая в текущей области, то из них будет использована функция из текущей области видимости.

```scala
trait Currency  
case class Dollar(amount: Double) extends Currency  
case class Euro(amount: Double) extends Currency  
  
object Currency {  
    implicit def euroToDollar(euro: Euro): Dollar = Dollar(euro.amount * 1.13)
}

object Example extends App {  

    implicit def euroToDollar(euro: Euro): Dollar = Dollar(euro.amount)

    val dollar: Dollar = Euro(100) // result: 100
}
```

### Предостережение!

![](assets/img/scala-implicits-and-typeclasses/mem1.png){: .w-50 .shadow }

С большой силой приходит большая ответственность! Неосознанное использование неявных преобразований может привести к написанию трудного для понимания кода (тем более когда кодовой базы становится много). Применять их нужно осознанно только там, где это действительно улучшает код и не создает дополнительной путаницы!

![](assets/img/scala-implicits-and-typeclasses/mem4.png){: .w-50 .shadow }

У вас может возникнуть вопрос - зачем тогда мы изучали неявные преобразования если их использование является антипаттерном?

Ответ такой - это механизм языка о котором стоит знать и понимать как оно работает. Поскольку сами имплиситы используются не только лишь для преобразований, но еще участвуют в других механизмах языка.

## **Implicit parameters**

```scala
def func(implicit x: Int): Unit = ???
```

Неявные параметры - это мощная особенность, позволяющая функциям автоматически получать значения для своих параметров из текущей области видимости без явной передачи аргументов при вызове функции.

Давайте рассмотрим простой пример:

```scala
def multiply(x: Int)(implicit y: Int) = x * y

implicit val z: Int = 10 // должна быть неявной

multiply(3) // result: 30 
multiply(4) // result: 40
```

В данном случае у метода `multiply` аргумент `y` передается неявно. Стоит отметить такой момент, что передаваемая переменная тоже должна быть отмечена как `implicit`.

Если в области видимости будут две неявно определенные переменные с одним и тем же типом, то (опять же ожидаемо) на выходе получим ошибку компиляции, поскольку компилятору непонято какую из переменных использовать:

```scala
implicit val z: Int = 10  
implicit val y: Int = 42  
  
multiply(3)

// [error]  ....scala:119:11: ambiguous implicit values:
// [error]  both value z in object ExampleImplicitParameters of type Int
// [error]  and value y in object ExampleImplicitParameters of type Int
// [error]  match expected type Int
// [error]   multiply(3)
```

Корректные и некорректные примеры объявления функций с неявными параметрами:

- `def func(implicit x: Int)` -  аргумент **`x`** неявный
- `def func(implicit x: Int, y: Int)` - аргументы **`x`** и **`y`** неявные
- `def func(x: Int, implicit y: Int)` - **ошибка компиляции!**
- `def func(x: Int)(implicit y: Int)` - аргумент **`y`** неявный
- `def func(implicit x: Int)(y: Int)`  - **ошибка компиляции!**
- `def func(implicit x: Int)(implicit y: Int)` - **ошибка компиляции!**

То есть, группа неявных параметров всегда должна быть последней.

Давайте рассмотрим пример использования неявных параметров больше приближенный к реальной жизни. Допустим у нас есть некое приложение в котором есть логгер. Приложение имеет некоторый контекст, например, он содержит некоторый id запроса, и нам нужно логировать информацию из этого контекста. При передаче в логгер этого параметра явно в коде приложения при логировании вынуждены писать `logger.log(...)(requestContext)`

```scala
case class RequestContext(requestId: String)  

class Logger {
  def log(message: String)(ctx: RequestContext): Unit = {  
      println(s"[${ctx.requestId}] $message")  
  }
}

object SomeApplication extends App {
  val logger = new Logger()

  def handle(requestContext: RequestContext) = {
    logger.log("Starting process")(requestContext)
    // some action ...
    logger.log("Continue process...")(requestContext)
    // some action ...
    logger.log("End process")(requestContext)
  }
}
```

Тогда как сделав параметр запроса неявным, мы можем избавиться от явной передачи этого параметра, тем самым упростив и уменьшив количество кода в бизнес логике нашего приложения.

```scala
case class RequestContext(requestId: String)  

class Logger {
  def log(message: String)(implicit ctx: RequestContext): Unit = {  
      println(s"[${ctx.requestId}] $message")  
  }
}

object SomeApplication extends App {
  val logger = new Logger()

  def handle(implicit requestContext: RequestContext) = {
    logger.log("Starting process")
    // some action ...
    logger.log("Continue process...")
    // some action ...
    logger.log("End process")
  }
}
```

## **Implicit classes**

В Scala есть возможность сделать классы неявными, выглядеть это будет следующим образом. 

```scala
implicit class ImplicitClass(val field: Int) extends AnyVal {
  def method: Unit = ???
}
```

### Зачем они нужны?

Давайте разберемся! Начнем с вопроса - какая разница между нашим кодом и библиотеками других разработчиков? Принципиальная разница в том, что свой код при желании мы можем изменить или расширить, но библиотеки, зачастую, приходится принимать такими, какие они есть.

![](assets/img/scala-implicits-and-typeclasses/draw0.png){: width="500" }

Чтобы облегчить решение этой проблемы, в языках программирования есть ряд подходов. В ООП языках, например, можно воспользоваться структурным паттерном [адаптер](https://refactoring.guru/ru/design-patterns/adapter). К примеру, мы хотим расширить тип (или класс) `Int` методами для проверки четности и нечетности. Для этого создаем класс обертку `IntAdapter`, в котором реализуем необходимые нам методы.

```scala
class IntAdapter(val i: Int) {
  def isEven: Boolean = i % 2 == 0
  def isOdd: Boolean = !isEven
}

// Создание экземпляра адаптера и использование его методов
new IntAdapter(42).isEven  // true
new IntAdapter(42).isOdd   // false
```

В Scala для этой цели мы можем использовать имплиситные классы. Для этого перепишем наш пример следующим образом.

```scala
implicit class RichInt(val i: Int) extends AnyVal {
  def isEven: Boolean = i % 2 == 0
  def isOdd: Boolean = !isEven
}

10.isEven  // true
10.isOdd   // false
```

Отличие будет в том, что методы для проверки на четность и нечетность теперь сможем вызывать так, будто они принадлежат типу `Int`. Это позволяет писать более лаконичный и выразительный код.

> Примечание: наследование от `AnyVal` в Scala используется для создания [value классов](https://docs.scala-lang.org/overviews/core/value-classes.html), которые представляют собой механизм оптимизации, позволяющий избежать выделения памяти для объектов-оберток.
{: .prompt-info }

Вы наверняка заметили, что пример с имплиситным классом подозрительно сильно похож на пример с адаптером, написанный выше. Дело в том, что если мы избавимся от синтаксического сахара (в IntelliJ IDEA можно сделать `Desugar Scala Code`), то увидим, что в обессахаренном коде производится явное оборачивание в класс обертку и вызов его методов.

```scala
// Обессахаренный код
org.example.app.RichInt(10).isEven  // true
org.example.app.RichInt(10).isOdd   // false
```

![](assets/img/scala-implicits-and-typeclasses/mem5.png){: .w-50 .shadow }

То есть фактически под капотом применяется тот же самый паттерн адаптер приправленный механизмом имплиситов.

## **Type classes**

Тайпкласс - это паттерн, используемый в функциональном программировании для обеспечения [Ad-hoc полиморфизма](https://en.wikipedia.org/wiki/Polymorphism_(computer_science)#Ad_hoc_polymorphism), известного как перегрузка методов. Этот паттерн позволяет писать код, в котором мы оперируем интерфейсами и абстракциями и при этом использовать правильную реализацию этих абстракций на основе типов.

### Полиморфизм через наследование

Начнем с рассмотрения абстрактного примера, в котором есть классы `Circle` и `Rectangle`. Нам нужно обогатить их методом для вычисления площади.

```scala
trait Area {
  def area: Double
}

class Circle(radius: Double) extends Area {
  override def area: Double = math.Pi * math.pow(radius, 2)
}

class Rectangle(width: Double, length: Double) extends Area {
  override def area: Double = width * length
}

// Обобщенная функция
def areaOf(area: Area): Double = area.area

areaOf(new Circle(10))
areaOf(new Rectangle(5, 5))
```

При использовании полиморфизма через наследование мы создаем интерфейс `Area` с методом `area` и наследуем от него классы `Circle` и `Rectangle`, в которых делаем реализацию этого метода. Это позволяет нам создать общую функцию `areaOf`, способную работать с любым типом, который наследуется от `Area`.

Данный подход, в большей степени, присущ ООП, когда поля и методы лежат в определении класса. То есть сущности, представляющие данные, **сосредоточены рядом** с сущностями, отвечающих за поведение.

### Полиморфизм через тайпклассы

Тайпклассы предлагают подход, когда сущности, представляющие данные, **отделены** от сущностей, отвечающих за поведение.

![](assets/img/scala-implicits-and-typeclasses/draw2.png){: width="500" .shadow }

В следующем примере интерфейс `Area` является тайпклассом. Он параметризован и метод его принимает на вход аргумент - те самые данные, которыми нужно будет оперировать в реализациях интерфейса.

```scala
// сущности, представляющие данные
case class Circle(radius: Double)
case class Rectangle(width: Double, length: Double)

// тайпкласс
trait Area[A] {
  def area(a: A): Double
}

// сущности, отвечающие за реализацию
class CircleArea extends Area[Circle] {
  override def area(circle: Circle) : Double = math.Pi * math.pow(circle.radius, 2)
}

class RectangleArea extends Area[Rectangle] {
  override def area(rectangle: Rectangle): Double = rectangle.width * rectangle.length
}

// Обобщенная функция
def areaOf[A](shape: A, area: Area[A]): Double = area.area(shape)

areaOf(Circle(42), new CircleArea)
areaOf(Rectangle(12, 15), new RectangleArea)
```

Мы можем уменьшить количество кода, если создадим неявные инстансы тайпкласса `Area` для типов `Circle` и `Rectangle`, а так же если будем пробрасывать эти инстансы в функцию `areaOf` неявно.

```scala
implicit val circleArea = new Area[Circle] {
  override def area(circle: Circle) : Double = math.Pi * math.pow(circle.radius, 2)
}

implicit val rectangleArea = new Area[Rectangle] {
  override def area(rectangle: Rectangle): Double = rectangle.width * rectangle.length
}

// Обобщенная функция
def areaOf[A](figure: A)(implicit area: Area[A]): Double = area.area(figure)

areaOf(Circle(42))
areaOf(Rectangle(12, 15))
```

Можно пойти еще дальше. Путем замены функции `areaOf` на имплиситный класс, мы можем добавить **синтаксис** для тайпкласса, что позволит вызывать метод `area` так, будто он принадлежит типам `Circle` и `Rectangle`.

```scala
// Синтаксис
implicit class AreaSyntax[A](val figure: A) extends AnyVal {
  def area(implicit area: Area[A]): Double = area.area(figure)
}

Circle(42).area
Rectangle(12, 15).area
```

По этим примерам видно, что тайпклассы, на самом деле, можно реализовать и в ООП языках программирования, но в Scala они, за счет имплиситов, выглядят более изящно и выразительно.

### Анатомия тайпклассов

Так из чего, в итоге, строятся тайпклассы? Они состоят их трех обязательных компонентов:
- trait (сам тайпкласс)
- методы тайпклассов
- инстансы трейта для определенных типов
- синтаксис, на базе implicit class (опционально)

Давайте рассмотрим эти компоненты поподробнее. Вот пример трейта:

```scala
trait TypeClass[A] {
  def method(value: A): Unit
}
```

Это собственно сам тайпкласс у которого есть некоторый метод. Важно обратить внимание, что этот трейт параметризован некоторым типом `A`.

Далее мы создаем инстансы этого тайпкласса, например для типа `Int`. По сути мы тут пишем реализацию класса и создаем его экземпляр, причем инстанс его создается в виде неявной переменной. Обычно инстансы размещают внутри объекта, который именуется названием тайпкласса и приставкой `Instances`.

```scala
object TypeClassInstances {
  implicit val intInstance = new TypeClass[Int] {
    def method(value: Int): Unit = ???
  }
}
```

Ну и необязательный компонент тайпклассов - это имплиситный класс, который позволяет создать некоторый синтаксис для нашего тайпкласса.

```scala
object TypeClassSyntax {
  implicit class TypeClassOps[A](private val value: A) extends AnyVal {
    def method(implicit ev: TypeClass[A]): Unit = ev.method(value)
  }
}
```

### Способы доставки инстансов тайпклассов

Инстанс тайпкласса мы можем прокинуть через аргументы функций (явно или неявно).

```scala
object SomeApp {
  def someMethod[A](arg: A)(implicit t: TypeClass[A]): Unit = {
    t.method(arg)
  }
}
```

Так же тайпклассы можно прокидывать через тайп-параметры.

```scala
object SomeApp {
  def someMethod[A: TypeClass]: Unit = {
    TypeClass[A].method
  }
}
```

Но для этого, нужно предварительно создать объект компаньон тайпкласса с методом `apply`, который умеет доставать неявный инстанс тайпкласса.

```scala
trait TypeClass[A] {
  def method(value: A): Unit
}

object TypeClass {
  def apply[A](implicit ev: TypeClass[A]): TypeClass[A] = ev
}
```

Для использования синтаксиса тайпкласса необходимо этот синтаксис импортировать.

```scala
object SomeApp {
  import TypeClassSyntax._
  
  def someMethod[A: TypeClass](arg: A): Unit = {
    arg.method
  }  
}
```

## **Simple type classes**

Давайте рассмотрим пару тайпклассов из реальной жизни. 

### Show

`Show` - это альтернатива для [джавового метода `toString`](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#toString--). Он определяется единственной функцией `show`.

Вам может быть любопытно для чего нужен этот тайпкласс, учитывая, что `toString` уже служит той же цели. Причем кейс-классы имеют неплохие реализации метода `toString`. Проблема в том, что `toString` определен на уровне `Any` (или джавовый `Object`) и, следовательно, может быть вызван для чего угодно, что не всегда корректно:

```scala
(new {}).toString
// result: "example.ExampleApp$$anon$5@5b464ce8"
```

То есть, тайпкласс `Show` позволит нам определять преобразования в строки только для нужных нам типов. Рассмотрим пример реализации тайпкласса:

```scala
// Тайпкласс
trait Show[A] {  
  def show(value: A): String  
}  
  
// Объект-компаньон
object Show {  
  def apply[A](implicit ev: Show[A]): Show[A] = ev  
}  
  
// Инстансы тайпкласса для Int и String
object ShowInstances {  
  implicit val showInt = new Show[Int] {  
    def show(value: Int): String = value.toString  
  }  

  implicit val showString = new Show[String] {  
    def show(value: String): String = value  
  }  
}  
  
// Синтаксис
object ShowSyntax {  
  implicit class ShowOps[A](private val value: A) extends AnyVal {  
    def show(implicit ev: Show[A]): Unit = ev.show(value)  
  }  
}  
```

Пример использования `Show` c примитивными типами.

```scala
import ShowInstances._
import ShowSyntax._  
  
val meaningOfLife = 42  
  
Show[Int].show(meaningOfLife)   // result: "42"  
// or
meaningOfLife.show   // result: "42"  
```

Пример использования `Show` c кастомными типами.

```scala
import ShowInstances._
import ShowSyntax._  

case class User(name: String, age: Int)

object User {
  
  implicit val showUser = new Show[User] {
    def show(user: User): String = s"User(name = ${user.name}, age = ${user.age})"
  }
}
  
val user = User("Mark", 25)
user.show
// result: "User(name = Mark, age = 25)"
```

### Eq

`Eq` является альтернативой стандартному методу [джавового `equals`](https://docs.oracle.com/javase/8/docs/api/java/lang/Object.html#equals-java.lang.Object-). Проблема с Java `equals` заключается в том, что мы можем сравнить два совершенно не связанных между собой типа и не получим ошибку от компилятора (максимум получим предупреждение), что может привести к веселым багам.

```scala
"Hello" == 42
// res1: Boolean = false
```

Введением этого тайпкласса, мы отсечем возможность сравнивать значения с разными типами на уровне компиляции.

Рассмотрим пример реализации тайпкласса:

```scala
// Тайпкласс
trait Eq[A] {
  def eqv(x: A, y: A): Boolean
}

// Объект-компаньон
object Eq {
  def apply[A](implicit ev: Eq[A]): Eq[A] = ev
}
  
// Инстансы тайпкласса для Int и String
object EqInstances {
  implicit val eqInt = new Eq[Int] {
    def eqv(x: Int, y: Int): Boolean = x == y
  }

  implicit val eqString = new Eq[String] {
    def eqv(x: String, y: String): Boolean = x == y
  }
}

// Синтаксис
object EqSyntax {
  implicit class EqOps[A](private val x: A) extends AnyVal {
    def eqv(y: A)(implicit ev: Eq[A]): Unit = ev.eqv(x, y)
    def ===(y: A)(implicit ev: Eq[A]): Unit = ev.eqv(x, y)
    def =!=(y: A)(implicit ev: Eq[A]): Unit = !ev.eqv(x, y)
  }
}
```

Пример использования `Eq` c примитивными типами.

```scala
import EqInstances._  
import EqSyntax._  

Eq[Int].eqv(2 + 2, 4)   // result: true  
  
"Hello" === "world"     // result: false  
"Hello" =!= "world"     // result: true  
  
"Hello" === 42  
/*  
  [error]  fff.scala:202:15: type mismatch;  
  [error]  found   : Int(42)  
  [error]  required: String  
  [error]   "Hello" === 42  
  [error]               ^  
  [error] one error found  
  [error] (Compile / compileIncremental) Compilation failed */
```

Пример использования `Eq` c кастомными типами.

```scala
case class User(name: String, age: Int)  
  
object User {
  implicit val eqUser = new Eq[User] {  
    def eqv(x: User, y: User): Boolean = x.name === y.name && x.age === y.age
  }
}

val mark = User("Mark", 25)
val joe = User("Joe", 33)

mark === joe  // result: false
```

## Заключение

Освоение имплиситов и тайпклассов является важным шагом на пути становления Scala разработчика. Однако, важно использовать их с умом. Правильно применяемые имплиситы и тайпклассы способствуют написанию лаконичного, выразительного и легко расширяемого кода, подчеркивая при этом мощь Scala.

## Некоторые ссылки

- [https://typelevel.org/cats/typeclasses.html](https://typelevel.org/cats/typeclasses.html)
- [https://medium.com/@olxc/type-classes-explained-a9767f64ed2c](https://medium.com/@olxc/type-classes-explained-a9767f64ed2c)
- [https://typelevel.org/cats/typeclasses/eq.html](https://typelevel.org/cats/typeclasses/eq.html)
- [https://typelevel.org/cats/typeclasses/show.html](https://typelevel.org/cats/typeclasses/show.html)
