---
layout: post
title:  Preč s implicitnými konverziami
date:   2021-01-29 08:45:00
categories: [Design kódu]
tags: scala
---

Konverzia je jednou z foriem aplikácie [Liskovej princípu][liskov]; inou aplikáciou je [dedenie v OOP][liskov-oop] alebo [variancia][liskov-variance].
Konverzia je metóda, ktorá dostane na vstup jeden parameter typu `A` a vráti výsledok typu `B`. Implicitná znamená, že sa aplikuje
automaticky, implicitne, všade tam, kde sa očakáva `B` a v kontexte je k dispozícii len objekt typu `A`. Aj keď sa
bez implicitnej konverzie niekedy nedá zaobísť (napríklad pri [Magnet patterne][magnet-pattern]), dnes sa implicitné konverzie
považujú za anti-pattern (rovnako aj Magnet pattern). Bohužiaľ si tento anti-pattern často berú do huby ignorantskí
odporcovia Scaly ako dôvod, prečo je Scala zlá. Ako keby sa v iných jazykoch nedalo napísať nič smradľavé.




V posledných rokoch, hlavne vďaka knižniciam typu [scalaz][scalaz], [cats][cats], apod. sa Scala stala viac funkcionálnym jazykom. To znamená,
že sa zmenilo aj mnoho best practices, ako napríklad aj užívanie implicitných konverzií. Príklad takej implicitnej konverzie
môžme vidieť tu:

```scala
import scala.language.implicitConversions

implicit def convertToInt(s: String): Int = s.toInt

def sumUp(values: Int*): Int = values.sum

sumUp("1", "2", "4.0", "35")
```

Keď kompilátor zbadá, že metódu `sumUp` voláme s argumentami typu `String`, zistí, že typy nesedia a tak hľadá implicitnú
hodnotu alebo konverziu, ktorou by mohol žiadaný typ `String` vyrobiť. Nájde ju (`convertToInt`) a použije. Výsledkom
je očakávaná 42.

[Liskovej substitučný princíp][liskov] hovorí o tom, že typ `A` vieme nahradiť typom `B` vtedy, ak platí, že `B` je podtypom `A`. V prípade našej implicitnej konverzie to znamená, že vieme nahradiť `Int` typom `String` (ak by `String` bol podtyp `Int`). Ale tu podvádzame, pretože `String` nie je skutočný podtyp `Int`. Miesto neho však máme predpis, podľa ktorého "podtyp" len simulujeme. Lebo ak `B` je podtypom `A`, vieme vždy skonvertovať `B` na `A`. 

## Prípad 1

Hlavný dôvod, prečo je implicitná konverzia považovaná za anti-pattern je ten, že *nevieme ako sa program bude chovať
v _runtime_ ak konverzia zlyhá*. Zlyhať môže vtedy:

- ak funkcia konverzie nie je matematicky "úplná" (po anglicky _total_).
  Čiže vtedy, ak existuje hodnota vstupného argumentu, ktorú funkcia nevie spracovať. Príklad:
  
```scala
implicit def stringToBoolean(s: String): Boolean = {
  s.toUpper match {
    case "TRUE" => true
    case "FALSE" => false
  }
}
```

- ak funkcia konverzie nie je referenčne transparentná, inak povedané, ktorá nemá "side effect". Príklad:

```scala
implicit def ipFromHost(host: String): InetAddress = {
  // side effect je napríklad čítanie súboru /etc/hosts z disku! Alebo aj UnknownHostException
  InetAddress.getByName(host) 
}
```

## Prípad 2

Okrem týchto relatívne viditeľných problémov existujú ďalšie (nie tak úplne) runtime problémy, ktoré majú spoločné nežiadúce skrývanie chovania. Predstavme si napríklad, že v konfigurácii máme uloženú nejakú URL ako String:

```scala
implicit def toURL(s: String): URL = ...

trait Configuration {
  def serviceUrl: String
}
val config: Configuration = ...


val url: URL = config.serviceUrl
```

Tento kód aktívne skrýva možné zlyhanie. Všetko vyzerá tak krásne, až kým `serviceUrl` nevráti napr. "abcdefgh" a program
spadne. Ako odchytíme chybu? Kde? Vo funkcii konverzie? V jej použití? Keďže sa implicitná konverzia volá automaticky,
pri jej používaní na viacerých miestach si už prestaneme všímať a uvedomovať si možné zlyhanie.

## Prípad 3

Predstavme si tento príklad:

```scala
implicit def toURL(s: String): URL = ...

def find(name: String): Option[String] = ...
def find(url: URL): Option[String] = ...  // viem, že tieto metódy sú hlúpy príklad...

find("file://...")
```

Ak viete trochu Scalu, tak sa posledného volania funkcie `find` nebojíte. Viete, že v tomto prípade sa žiadna konverzia
realizovať nebude, pretože netreba. Horšie to však bude, keď zapojíme `Trait`:

```scala
trait Configuration {
  def serviceUrl: String
}

trait NameKeyStore {
  def find(name: String): Option[String] 
}

trait UrlKeyStore extends NameKeyStore {
  def find(url: URL): Option[String]
}

class UrlKeyStoreImpl extends UrlKeyStore {
  ...
}


// BadApplication.scala
object BadApplication {

  val config: Configuration = ...
  val store = new UrlKeyStore() 

  implicit def toURL(s: String): URL = ...
  store.find(config.serviceUrl) // no, ktorá 'find' sa zavolá?
}
```

Ktorá z dvoch implementácií metódy `find` za zavolá? Predpokladajme, že funkcia `find(String)` vráti niečo iné ako
`find(URL)` (nie je to vôbec pekné, ale aj také kódy bývajú).

Ak sú všetky tieto traity v iných súboroch a my
vidíme len súbor s objektom `BadApplication`, jednoducho to nemôžeme vedieť (bez podpory nášho inteligentného IDE).
Nepriamo implicitná konverzia skrýva to, čo by nemalo byť skryté. 

## Čo použiť miesto implicitnej konverzie

Implicitné konverzie sa často píšu vtedy, keď vieme jednoducho vytvoriť potrebný typ `B` z typu `A`; a keď to robíme príliš
často, ako napr.:

```scala
implicit def stringToUrl(s: String): URL = ...

val config: Configuration = ...

service1.find(config.serviceUrl1)  // def find(url: URL) 
service2.find(config.serviceUrl2)  // def find(url: URL)
service3.find(config.serviceUrl3)  // def find(url: URL)
...
```

Toto použitie má však symptómy vyššie uvedených prípadov. Miesto toho, aby sme teraz zahodili celú Scalu, skúsme nájsť
riešenie, ktoré by nás nebolelo.

### Riešenie 1: Extension metóda

```scala
object syntax {
  object string {
    implicit class StringExt(str: String) {
      def toURL: URL = new URL(str)
      def toTryURL: Try[URL] = Try(toURL)
      def toMaybeURL: Option[URL] = toTryURL.toOption
    }
  }
}

// Použitie:
import syntax.string._

service1.find(config.serviceUrl1.toURL) 
service2.find(config.serviceUrl2.toURL)
service3.find(config.serviceUrl3.toURL)

```

Rozdielom oproti implicitnej konverzii je okrem explicitného volania `.toURL` fakt, že:

- výsledný typ môže byť zaobalený do niečoho, čo môže zachytávať chybový stav. My sami sa tu rozhodujeme, ako ten chybový stav zachytíme.
- môžme vytvoriť niekoľko variantov konverzie, z ktorých si pri použití vyberieme.
- konverzia nie je viditeľná v celom scope, ale len tam, kde ju importujeme

### Riešenie 2: Typová trieda (type class)

Predstavme si, že potrebujeme napríklad previesť ľubovoľný objekt do JSON-u.
Túto operáciu (vlastne _konverziu_) vieme popísať aj typovou triedou, pre ľubovoľný typ `A`:

```scala
trait JsonPrintable[A] {
  def toJson(value: A): Try[String]
}
```

Jednou z možností ako napísať metódu, ktorá využíva túto operáciu je takáto:

```scala
def sendJson[A](value: A)(implicit printable: JsonPrintable[A]): Unit = {
  val json = printable.toJson(value)
  ...
}
```

Už teraz vidno, že sa jedná o úplne iný prístup ku konverzii. Nielenže nepoužívame skutočnú implicitnú konverziu, ktorá skrýva veci "pod pokličku", ale zároveň nám ostala krásna syntax použitia. Pre každý typ zvlášť vytvoríme implicitnú
inštanciu typovej triedy a použitie je priam skvostné:

```scala
case class Person(name: String, age: Int)

implicit val personJsonPrintable = new JsonPrintable[Person] {
  def toJson(value: Person): Try[String] = Try(s"""{"name":"${value.name}","age":${value.age}}""")
}

sendJson(Person("Peter", 36))
```

Vidíte tú krásu? Dosiahli sme syntakticky ideálne riešenie, ktoré nič neskrýva. Pokiaľ by nastal problém v konverzii
objektu na JSON, budeme ju mať zachytenú v monáde `Try`. 

Nie vždy sa však dá použiť typová trieda. Problém nastáva, keď potrebujeme skutočný typ `B`, nie len operácie nad `B`.








[liskov]: https://en.wikipedia.org/wiki/Liskov_substitution_principle
[liskov-oop]: https://reflectoring.io/lsp-explained/
[liskov-variance]: https://apiumhub.com/tech-blog-barcelona/scala-generics-covariance-contravariance/
[magnet-pattern]: http://blog.madhukaraphatak.com/scala-magnet-pattern/
[scalaz]: https://github.com/scalaz/scalaz
[cats]: https://github.com/typelevel/cats