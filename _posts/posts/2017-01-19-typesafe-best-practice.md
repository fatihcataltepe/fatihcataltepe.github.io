---
layout: post
title: Proper Usage of Typesafe in Scala Project
date:   2017-01-19 23:14:24 +0300
categories: general typesafe configuration scala
---

`Typesafe` library allows developers to configure `scala` applications very easily. In this post, I will try to explain the proper usage of this library.
Let's start:

* Add the following lines to your `build.sbt` the following lines and rebuild your project.

```scala
//typesafe config
libraryDependencies += "com.typesafe" % "config" % "1.3.0"
```

* Create an `application.conf` file under the `src/main/resources` directory. Add the following snippet inside your `application.conf` file.

```json
person = {
  name = steve
  age = 26
  height = 190
  weight = 88
}
```

* Create a corresponding class for `Person` :

```scala
case class Person(props: PersonProps)
```

* Create a corresponding class for `PersonProps` and its companion object :

```scala
case class PersonProps(name:String, age: Int, height: Int, weight: Int)

object PersonProps{
    def apply(config: Config = ConfigFactory.load()): PersonProps = {
        val name: String = config.getString(person.name)
        val age: Int= config.getInt(person.age)
        val height: Int= config.getInt(person.height)
        val weight: Int= config.getInt(person.weight)

        PersonProps(name, age, height, weight)
    }
}
```


* Let's run the example:

```scala
object Main extends App{
    //since no configuration is assigned, application.conf will be used by default
    val props: PersonProps: PersonProps()

    //initialize the person
    val person: Person = Person(props)

    //print the properties of person
    println(s"age: ${person.age}")
}

```





