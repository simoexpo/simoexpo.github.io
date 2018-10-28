---
layout: post
title:  "Phantom subtyping"
categories: [scala]
tags: [scala, implicits, phantom types]
---

# Phantom Subtyping

## Introduction

When modeling a system one of the basic step is to define the representation of the various entities inside it. Sometimes could be easy to find a good representation for an entity that fit most of the application use cases, but then we discover a new specific behaviors for which the choosen models don't fit very well and maybe it's neessary to add some new properties to the former model in order to fix the situation. 

Adding some properties to an already defined model could mean two things:

- that we found out we miss something that we atually need
- that our model could have different state in different situation

In the first case we can simply add new fields or define new methods, but the second one it's a bit more tricky. First of all the new property could be undefined for some instances, other than that if we are required to add new methods or new operation they also could be legit only in some specific state of the model and not for all of them.

Sometimes when encountering this situation we realize that the right thing to do is splitting our model because, from a logical point of view, we need to represent different entities. Other times instead it dosen't make sense to define different model because the entity is well defined in a single class.

In this post we'll try to show a possible solution to this specific problem using a powerful mechanism of Scala, the *phantom types*.

## A simple use case

Let's take the following case class definition:

```scala
case class User(firstName: String, lastName: String)
```

This could be a good representation for a user entity, but then we decide that we want to persist our user in a database and once it's saved we also want the possibility to activate it. In order to do that we decide to add the new details in our `User` case class.

```scala
case class User(id: Option[Long] = None, firstName: String, lastName: String, activetedAt: Option[LocalDate] = None)
```

After these additions we have a new class composed by some strictly user related properties like the first name and the last name and also some system related properties like the user id and the activation date. You should have noticed by now that we decide to make these two new fields optional, this is obviously because we can assume that all users will be activated and no user can have an id before the save.

Now let's try our `User` class during some common operations:

```scala
class UserDao() {
    def insert(user: User): User = ???
    def activate(userId: Long): User = ???
}

val userDao: UserDao = new UserDao()
val user = User(firstName = "Jon", lastName = "Snow")
val registeredUser = userDao.insert(user)
userDao.activate(registeredUser.id)   // ??? 
```

Ok there is something wrong here! We need to access the `id` of our user to activate it and we know our user has an `id` because we previously inserted it in the `UserDao` but... we can't directly access the `id` field because it's wrapped in an `Option`.

A quick and dirty solution could be invoking the `get()` method on the optional id to retrieve the actual value.

```scala
userDao.activate(registeredUser.id.get)   // now it works :)
```

Ok now it works! But... Well usually it's not a very good idea to call `get()` on an `Option` value, expecially considering that we could encounter this kind of problem multiple times in our code and so we need to replicate this trick each time. In fact, even if we are sure that the optional value exists, this approach could be error prone and make the code less robust (it could bring to errors during refactoring for example). All of this is due to the fact that it's not possible to be sure at compile time that a user instance actually has the id property setted.

In order to avoid this situation, there are many solution like mapping on the optional value or using the subtyping to define multiple `User` for every possible status (i.e. draft, registered and activated in our example). While all these solution are perfectly fine from a program safety point of view they can lead to code duplication or reduce readability in some cases.

Here we are going to present another possible solution using Scala *phantom types*.

## Introducing phantom types

```
 .-.
(o o) boo!
| O \
 \   \
  `~~~'
```

*Phantom types* are a particular Scala feature that allow to define some additional property to check at compile time. The name is due to the fact that these types are never instantiated, these types are used by the compiler but, actually, don't have any effect on the code at runtime.

Enough talk, let's see these *"scary" phantom types* at work!

```scala
class Light {
    def turnOn(): Light = ???
    def turnOff(): Light = ???
}

val light = new Light()
light.turnOn.turnOff
light.turnOn.turnOn         // This is perfectly legit but it seems a bit strange :|
```

This is a simple class that define a light with two simple operation: turn the light on or turn it off. This is actually quite simple but what if we want to allow the `turnOn` method only when the light is off and the `turnOff` method only when it's on? Well this is a perfect use case to exploit the *phantom types* and achieve the result in a very elegant way.

```scala
sealed trait LightStatus
trait LightOn extends LightStatus
trait LightOff extends LightStatus

class Light[T <: LightStatus] {    // T is a phantom type
    def turnOn(implicit ev: T =:= LightOff): Light[LightOn] = ???
    def turnOff(implicit ev: T =:= LightOn): Light[LightOff] = ???
}

val light = new Light[LightOff]()
light.turnOn.turnOff
light.turnOn.turnOn             // error: Cannot prove that LightOn =:= LightOff (at compile time!)
```

And finally it works as we expected!

Before moving on let's try to understand what's going on here and why we were able to achieve this behavior. First of all we have defined some traits to model the two possible state of the a light (`LightOn` and `LightOff`) then we used these traits to define a *phantom type* on the `Light` class. In addition the `turnOn` and `turnOff` methods have been modified to reuturn respectively an instance of `Light` with status `LightOn` (`Light[LightOn]`) and an instance with status `LightOff` (`Light[LightOff]`) and now took an implicit parameter of this strange type `T =:= LightOff` (`T =:= LightOn` for the `turnOff` method).

The trick is that we can call the two `Light`'s methods only if the required implicits are in scope but `A =:= B` define a special implicit that is always in scope for any `A == B`. This means that in our `Light[T]` class we will always have an implicit of type `T =:= LightOff` if `T` is `LightOff` and an implicit `T =:= LightOn` if `T` is `LightOn` and, as a consequence, we are able to call only one of the two methods depending on the `T` *phantom type*.

In addition to `A =:= B` it's also possible to define another special implicit `A <:< B` that share more or less the same behavior but this time is always defined only if `A <: B`.

## Phantom types in action

After all this introduction we can finally look at how the *phantom types* feature can help use to solve our little optional property problem on the `User` class.

```scala
case class User[A <: UserStatus] private(private val _id: Option[Long], name: String, age: Int, private val _activatedAt: Option[LocalDate]) {

    def id(implicit ev: A <:< RegisteredUser): Long = _id.get

    def activatedAt(implicit ev: A =:= ActivatedUser): LocalDate = _activatedAt.get
}

object User {
    sealed trait UserStatus
    trait DraftUser extends UserStatus
    trait RegisteredUser extends UserStatus
    trait ActivatedUser extends RegisteredUser

    def apply(name: String, age: Int): User[DraftUser] = {
        new User(None, name, age, None)
    }

    def apply(id: Long, name: String, age: Int): User[RegisteredUser] = {
        new User(Some(id), name, age, None)
    }

    def apply(id: Long, name: String, age: Int, activatedAt: LocalDate): User[ActivatedUser] = {
        new User(Some(id), name, age, Some(activatedAt))
    }
}

class UserDao() {
    def insert(user: User[DraftUser]): User[RegisteredUser] = ???
    def activate(userId: Long): User[ActivatedUser] = ???
}

val userDao: UserDao = new UserDao()
val user = User(firstName = "Jon", lastName = "Snow")
val registeredUser = userDao.insert(user)
userDao.activate(user.id)             // this won't compile since user is of type User[DraftUser]
userDao.activate(registeredUser.id)   // since registeredUser is of type User[ActivatedUser] now the id is immediatly available
```

With the above changes to the `User` class we are now forced to create a user from the utility `apply` methods in the companion object (since the constructor is now `private`) and we are sure to have always the right user status based on the arguments we pass during the creation. In addition the `id` and `activatedAt` properties are now private and exposed with two methods available, thank to the *phantom type*, only to the right instances of `User` and unwrapped from the `Option` container.

Obviusly we are still using the "infamous" `get()` method to achieve this result but we are doing it in a more safer way since this time we isolate these calls in the class definition where we know for sure, thanks to the *phantom type*, that the instance has that property setted.

This mechanism can help writing a more elegant and readable code and to keep the complexity of `User` in the class definiton. Moreover we are now able to guarantee at compile time, given certain conditions, the existance of optional value and that we are using the right `User` instances for the right operations.

## Conclusion

The above example demonstrate how we can exploit *phantom types* to enanche compile time property check. In this specific case we manage to "split" our `User` class in different types based on some properties without actually extends it and without define any sub classes. In this way we were able to define different operation available only for some specific type of `User` and to access some optional fields directly only in case they were defined. As a result we got a cleaner code and a static guarantee that we use the right `User` in the right situation.

### Pros
- compile time check on optional properties existance
- ability to define different behaviour for instance with different properties
- clean and readable code
- enforce organized creation of instances with well defined factory methods to avoid errors
- use of unsafe `get()` methods only inside the class definition

### Cons
- in order to make it works we need to write a bit of boilerplate when defining phantom types and classes
- phantom types syntax could be a bit unclear at the beginning
- use `copy()` on a case class instance with phantom type could break all the mechanism (you can redefine it properly to avoid this)
- need to specify the phantom type on every occurrency of the class type
- phantom types will be removed in dotty, however the same behavior could be achieved using [erased terms](http://dotty.epfl.ch/docs/reference/erased-terms.html)
