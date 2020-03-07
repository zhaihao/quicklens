![Quicklens](https://github.com/softwaremill/quicklens/raw/master/banner.png)

[![Maven Central](https://maven-badges.herokuapp.com/maven-central/com.softwaremill.quicklens/quicklens_2.11/badge.svg)](https://maven-badges.herokuapp.com/maven-central/com.softwaremill.quicklens/quicklens_2.11)

**Modify deeply nested fields in case classes:**

````scala
import com.softwaremill.quicklens._

case class Street(name: String)
case class Address(street: Street)
case class Person(address: Address, age: Int)

val person = Person(Address(Street("1 Functional Rd.")), 35)

val p2 = person.modify(_.address.street.name).using(_.toUpperCase)
val p3 = person.modify(_.address.street.name).setTo("3 OO Ln.")

// or
 
val p4 = modify(person)(_.address.street.name).using(_.toUpperCase)
val p5 = modify(person)(_.address.street.name).setTo("3 OO Ln.")
````

**Chain modifications:**

````scala
person
  .modify(_.address.street.name).using(_.toUpperCase)
  .modify(_.age).using(_ - 1)
````

**Modify conditionally:**

````scala
person.modify(_.address.street.name).setToIfDefined(Some("3 00 Ln."))
person.modify(_.address.street.name).setToIf(shouldChangeAddress)("3 00 Ln.")
````

**Modify several fields in one go:**

````scala
import com.softwaremill.quicklens._

case class Person(firstName: String, middleName: Option[String], lastName: String)

val person = Person("john", Some("steve"), "smith")

person.modifyAll(_.firstName, _.middleName.each, _.lastName).using(_.capitalize)
````

**Traverse options/lists/maps using .each:**

````scala
import com.softwaremill.quicklens._

case class Street(name: String)
case class Address(street: Option[Street])
case class Person(addresses: Seq[Address])

val person = Person(List(
  Address(Some(Street("1 Functional Rd."))),
  Address(Some(Street("2 Imperative Dr.")))
))

val p2 = person.modify(_.addresses.each.street.each.name).using(_.toUpperCase)
````

`.each` can only be used inside a `modify` and "unwraps" the container (currently supports `Seq`s, `Option`s and
`Maps`s - only values are unwrapped for maps).
You can add support for your own containers by providing an implicit `QuicklensFunctor[C]` with the appropriate
`C` type parameter.

**Traverse selected elements using .eachWhere:**

Similarly to `.each`, you can use `.eachWhere(p)` where `p` is a predicate to modify only the elements which satisfy
the condition. All other elements remain unchanged.

````scala
def filterAddress: Address => Boolean = ???
person
  .modify(_.addresses.eachWhere(filterAddress)
           .street.eachWhere(_.name.startsWith("1")).name)
  .using(_.toUpperCase)
````

**Modify specific elements in an option/sequence/map using .at:**

````scala
person.modify(_.addresses.at(2).street.at.name).using(_.toUpperCase)
````

Similarly to `.each`, `.at` modifies only the element at the given index/key. If there's no element at that index,
an `IndexOutOfBoundsException` is thrown. In the above example, `.at(2)` selects an element in `addresses: Seq[Address]`
 and `.at` selects the lone possible element in `street: Option[Street]`. If `street` is `None`, a
 `NoSuchElementException` is thrown.
 
`.at` works for map keys as well:

````scala
case class Property(value: String)

case class PersonWithProps(name: String, props: Map[String, Property])

val personWithProps = PersonWithProps(
  "Joe",
  Map("Role" -> Property("Programmmer"), "Age" -> Property("45"))
)

personWithProps.modify(_.props.at("Age").value).setTo("45")
````

Similarly to `.each`, `.at` modifies only the element with the given key. If there's no such element,
an `NoSuchElementException` is thrown.

**Modify specific elements in an option or map with a fallback using .atOrElse:**

````scala
personWithProps.modify(_.props.atOrElse("NumReports", Property("0")).value).setTo("5")
````

If `props` contains an entry for `"NumReports"`, then `.atOrElse` behaves the same as `.at` and the second
parameter is never evaluated. If there is no entry, then `.atOrElse` will make one using the second parameter
 and perform subsequent modifications on the newly instantiated default.
 
 For Options, `.atOrElse` takes no arguments and acts similarly. 
 
 ````scala
 person.modify(_.addresses.at(2).street.atOrElse(Street("main street")).name).using(_.toUpperCase)
 ````
 
 `.atOrElse` is currently not available for sequences because quicklens might need to insert many
 elements in the list in order to ensure that one is available at a particular position, and it's not
 clear that providing one default for all keys is the right behavior. 

**Modify Either fields using .eachLeft and eachRight:**

````scala
case class AuthContext(token: String)
case class AuthRequest(url: String)
case class Resource(auth: Either[AuthContext, AuthRequest])

val devResource = Resource(auth = Left(AuthContext("fake"))

val prodResource = devResource.modify(_.auth.eachLeft.token).setTo("real")

````

**Modify fields when they are of a certain subtype:**

```scala
trait Animal
case class Dog(age: Int) extends Animal
case class Cat(ages: Seq[Int]) extends Animal

case class Zoo(animals: Seq[Animal])

val zoo = Zoo(List(Dog(4), Cat(List(3, 12, 13))))

val olderZoo = zoo.modifyAll(
  _.animals.each.when[Dog].age,
  _.animals.each.when[Cat].ages.at(0)
).using(_ + 1)
```

This is also known as a *prism*, see e.g. [here](http://julien-truffaut.github.io/Monocle/optics/prism.html).

**Re-usable modifications (lenses):**

````scala
import com.softwaremill.quicklens._

val modifyStreetName = modify(_: Person)(_.address.street.name)

val p3 = modifyStreetName(person).using(_.toUpperCase)
val p4 = modifyStreetName(anotherPerson).using(_.toLowerCase)

//

val upperCaseStreetName = modify(_: Person)(_.address.street.name).using(_.toUpperCase)

val p5 = upperCaseStreetName(person)
````
Alternate syntax:
````scala
import com.softwaremill.quicklens._

val modifyStreetName = modify[Person](_.address.street.name)

val p3 = modifyStreetName.using(_.toUpperCase)(person)
val p4 = modifyStreetName.using(_.toLowerCase)(anotherPerson)

//

val upperCaseStreetName = modify[Person](_.address.street.name).using(_.toUpperCase)

val p5 = upperCaseStreetName(person)
````

**Composing lenses:**

````scala
import com.softwaremill.quicklens._

val modifyAddress = modify(_: Person)(_.address)
val modifyStreetName = modify(_: Address)(_.street.name)

val p6 = (modifyAddress andThenModify modifyStreetName)(person).using(_.toUpperCase)
````
or, with alternate syntax:
````scala
import com.softwaremill.quicklens._

val modifyAddress = modify[Person](_.address)
val modifyStreetName = modify[Address](_.street.name)

val p6 = (modifyAddress andThenModify modifyStreetName).using(_.toUpperCase)(person)
````


**Modify nested sealed hierarchies:**

> *Note: this feature is experimental and might not work due to compilation order issues.
> See https://issues.scala-lang.org/browse/SI-7046 for more details.*

````scala
import com.softwaremill.quicklens._

sealed trait Pet { def name: String }
case class Fish(name: String) extends Pet
sealed trait LeggedPet extends Pet
case class Cat(name: String) extends LeggedPet
case class Dog(name: String) extends LeggedPet

val pets = List[Pet](
  Fish("Finn"), Cat("Catia"), Dog("Douglas")
)

val juniorPets = pets.modify(_.each.name).using(_ + ", Jr.")
````

---

Similar to lenses ([1](http://eed3si9n.com/learning-scalaz/Lens.html),
[2](https://github.com/julien-truffaut/Monocle)), but without the actual lens creation.

Read [the blog](http://www.warski.org/blog/2015/02/quicklens-modify-deeply-nested-case-class-fields/) for more info.

Available in Maven Central:

````scala
val quicklens = "com.softwaremill.quicklens" %% "quicklens" % "1.4.13"
````

Also available for [Scala.js](http://www.scala-js.org) and [Scala Native](http://www.scala-native.org)!

## Commercial Support

We offer commercial support for Quicklens and related technologies, as well as development services. [Contact us](https://softwaremill.com) to learn more about our offer!

## Copyright

Copyright (C) 2015-2019 SoftwareMill [https://softwaremill.com](https://softwaremill.com).
