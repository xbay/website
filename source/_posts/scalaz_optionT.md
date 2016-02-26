date: 2016-02-26 23:19:50
author: [uni.x.bell]
title: 用scalaz的optionT让业务代码变得更简洁
tags: [scala, scalaz, optionT, Option, Future, Monad]
---

scala里使用Option类型来封装一个可能为null的引用，和Java引用相比，它避免了直接操作null引发的异常。一般来说可以用switch/case来处理Option，比如要把一个Int对象转成一个String对象：
```scala
def f(p: Option[Int]): Option[String] = {
  p match {
    case Some(i) => i.toString
    case None => None
  }
}
```
和Java里判断引用是否为空的代码其实是类似的，不过Option的封装保证了程序员不会忘记处理空的情况－－除非程序员使用get取值，非常不推荐这样做：
```scala
p.get.toString
```
跟Java一样，这样写代码很容易收获一个空引用异常。
不过通过Option的map函数能把代码写得跟用get求值一样简洁，还不会引发空引用异常。
```scala
def f(p: Option[Int]): Option[String] = p.map(_.toString)
```
map函数只在Option为Some()的时候调用传入的函数，也就是说，None被直接“短路”到输出了，这对于大部分情况都是适用的。

---

scala里用Future来封装IO操作，代表”未来可能会获得某个值“。比如发起一个HTTP请求或者访问数据库，函数会返回一个Future，和Option有点类似，Future也有俩状态，一个是成功一个是失败。所以常用的方法也是map。
```scala
def f(p: Future[Int]): Future[String] = p.map(_.toString)
```
当IO成功的时候，获得的值会被用做参数，调用map里传入的函数，失败则直接“短路”到输出。
我们还可以用flatMap简单的把IO“串联”起来：
```scala
def readById(id: Int): Future[String] = ...
def readByName(name: String): Future[Result] = ...
def getResult(p: Future[Int]): Future[Result] = p.flatMap(id => {
  readById(id).flatMap(name => {
    readByName(name)
  })
})
```
这么写在形式上有个弱点，类似回调函数，每次串联都会带来一层缩进，影响可读性。scala提供了for来处理这个问题，下面这段代码和上面的多层flatMap是等价的：
```scala
def readById(id: Int): Future[String] = ...
def readByName(name: String): Future[Result] = ...
def getResult(p: Future[Int]): Future[Result] = for {
  id <- p
  name <- readById(id)
  result <- readByName(name)
} yield(result)
```

---

不过真实世界里的代码会稍微复杂一些，我们经常会混用Future和Option，最常见的是IO返回的对象里，某些属性可能为空，我们用Option来表示这些属性的话，意味着代码逻辑里会出现Future[Option[T]]这样的类型。我们当然可以在Future的map/flatMap里用match/case来写逻辑，其实也比较清晰，但是缩进的层数会比较多，影响可读性。来看一段实际的例子，这是从真实世界的业务代码里拷贝/粘贴出来的：
```scala
def cityCodeFriendly(term: Option[String]): Future[Option[JsValue]] = {
  term match {
    case Some(cityCode) => {
      AreaService.findOneByCode(cityCode) flatMap { cityOption =>
        cityOption match {
          case Some(city) => {
            AreaService.findOneByCode(city.parentCode.getOrElse("")) map { provinceOption =>
              provinceOption match {
                case Some(province) => {
                  Some(JsObject(Seq("provinceName" -> JsString(province.name.getOrElse("")),
                    "cityName" -> JsString(city.name.getOrElse("")),
                    "cityCode" -> JsString(cityCode))))
                }
                case None => None
              }
            }
          }
          case None => Future(None)
        }
      }
    }
    case None => Future(None)
  }
}
```
代码逻辑还是比较清晰的：传入一个城市id（可能为空），然后从数据库里依次读取城市的名称和所属省份的名称，最后把结果组装成一个json对象输出，只要中间有一步读到空值，就输出一个空的json对象。代码风格看起来有两个问题，一是缩进太多，第二是case None => None看起来很多余。
试试用Option的flatMap来改造一下，得到如下的代码：
```scala
def cityCodeFriendly(term: Option[String]): Future[Option[JsValue]] = {
  term.map(cityCode => {
    AreaService
    .findOneByCode(cityCode)
    .flatMap(cityOption => {
      cityOption.map(city => {
        city.parentCode.map(parentCode => {
          AreaService.findOneByCode(parentCode).map(provinceOption => {
            provinceOption.map(province =>
              JsObject(Seq("provinceName" -> JsString(province.name.getOrElse("")),
                "cityName" -> JsString(city.name.getOrElse("")),
                "cityCode" -> JsString(cityCode))))
          })
        }).getOrElse(Future(None))
      }).getOrElse(Future(None))
    })
  }).getOrElse(Future(None))
}
```
看起来更糟糕了，Future的flatMap只能和Future级联，Option的flatMap只能和Option的级联，所以两者都用flatMap其实并无好处，反而需要用.getOrElse(Future(None))来处理分支的情形，比switch/case更不清晰，缩进只是略少，总体来说得不偿失。
因为Future和Option的flatMap互相不能级联，所以我们没发用for把它们串联起来，看看flatMap写法里的getOrElse就知道了。
但是从逻辑上来说，应该存在一种方法，能把Future和Option串起来，中间只要有Future.failure就短路到Future.failure，只要有Future(None)就短路到Future(None)，这是一个很机械的逻辑，编译器应该能处理。
scalaz提供的optionT函数，就能做到这一点。改造后实现看起来清爽多了：
```scala
def cityCodeFriendly(term: Option[String]): Future[Option[JsValue]] = {
  def wrapCityCode: Future[Option[String]] = Future(term)
  def wrapJsObject(province: Area, city:Area, cityCode: String): Future[Option[JsValue]] = 
    Future(Some(JsObject(Seq(
      "provinceName" -> JsString(province.name.getOrElse("")),
      "cityName" -> JsString(city.name.getOrElse("")),
      "cityCode" -> JsString(cityCode)))))
      
  (for {
    cityCode <- optionT(wrapCityCode)
    city <- optionT(AreaService.findOneByCode(cityCode))
    parentCode <- optionT(Future(city.parentCode))
    province <- optionT(AreaService.findOneByCode(parentCode))
    res <- optionT(wrapJsObject(province, city, cityCode))
  } yield res).run
}
```
optionT函数接收一个Future[Option[T]]类型，返回一个封装好的OptionT类型，OptionT类型相当于“混合”了Future和Option的特性，它提供了map/flatMap函数，只针对成功并有值的情况调用传入的函数，否则直接“短路”到输出。这样的写法不但让代码变得简洁（平板的结构，很少的缩进），而且在编写过程中不需要再去关注错误处理（不需要再加入case None => None 或者 getOrElse之类的玩意了）。