---
title: "Kotlin's Syntax Sugar / Kotlinのシンタックスシュガー —— Embracing Kotlin"
emoji: "🫠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [kotlin]
published: true
---

# Kotlinを愛でる - Syntax Sugar
## What is "syntax sugar"?

> Syntax Sugar 糖衣構文
> プログラミング言語において、読み書きのしやすさのために設計されている。より明確に、より簡潔に、あるいは一部の人が好むような別のスタイルで表現が可能になる.
> ref: https://en.wikipedia.org/wiki/Syntactic_sugar

## Kotlin's syntax sugar
### 呼び出し/演算子
#### property access (getter/setter)
getterやsetterがproperyのようにアクセス可能
```kotlin
data class User(
    val id: String,
    val name: String,
)

user.id = "1234567890"
user.id // "1234567890"
// desugar
user.setId("1234567890")
user.getId() // "1234567890"
```

#### operator invoke
```kotlin
data class User private constructor(
    val id: String,
    val name: String,
){
    init {
        require(name.length < 20)
    }

    companion object {
        fun of(name: String): User {
            return User(id = "id_$name", name = name)
        }

        operator fun invoke(name: String): User = of(name)
    }
}

val sugaredUser = User("hondaya")
// desugar
val user = User.of("hondaya")

```
ref: [Case Study: Why Kakao Pay Chose Kotlin and Spring for Backend Development](https://blog.jetbrains.com/kotlin/2025/07/case-study-why-kakao-pay-chose-kotlin-for-backend-development/)


### ラムダ・関数呼び出し

#### Trailing lambda
個人的にkotlinのもっとも愛でポイントである気持ちのいいsyntaxを実現しているsyntax sugarであると思う
関数の引数の末尾が関数であったり、関数の引数が関数の場合

stdlibの実装で多く使う一つ`filter`
```kotlin
val numberList = listOf(-1, 0, 1, 2)
numberList.filter { it > 0} // [1, 2]
```
stdlibの内部実装、関数定義的には以下のようになっている.
```kotlin
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T> {
    return filterTo(ArrayList<T>(), predicate)
}
```
定義している関数の引数が関数のみの場合、このような記法で記述できる.

よく使うcoroutineやKtorのroutineも全てこのsyntax sugarによる実装である.
```kotlin
// coroutine
withContext(Dispatchers.IO) { 
    // access storage process 
}

// Ktor routing
get("hello") { call.respond("world") }

// Kotest test
class DomainTest : FunSpec({
    test("domain is valid") {
        // do test assertion
    }
})
```
ref: [Passing trailing lambdas](https://kotlinlang.org/docs/lambdas.html#passing-trailing-lambdas)


#### Destructuring in lambdas
kotlinでは、lambdaのパラメータを分割できる. `Pair`, `data class`, `Map.Entry`, ... などさまざまなデータ型に対してdestructuringが可能.
```kotlin
fun printMap(inputMap: Map<String, Int>) {
    inputMap.map { (key, value) -> print("$key: $value")}
}

fun printUser(users: List<User>) {
    users.map { (id, name) -> print("$id: $name")}
}
```

kotlinの言語内部でのこれの実現は、componentN()の実装がされているためで、Mapでは, component1()ではkey, component2()ではvalueが返却されるように実装されている.
data classなどもメソッド生成によりpropertyに対して、componentN()が自動生成されるため、destructureingが可能になっている.
https://github.com/JetBrains/kotlin/blob/617c5702dc0b28c2f53663c09f6de301aead8d94/libraries/stdlib/src/kotlin/collections/Maps.kt#L301-L325

#### Return@label
labelを使ってのreturnが利用でき、lamdba記法のまま、scopeの切り替えが柔軟になる
```kotlin
list.forEach {
    if (it < 0) return@forEach
    use(it)
}

// desugar
for (e in list) {
    if (e < 0) continue
    use(e)
}
```

### 宣言

#### companion object
staticな変数や関数を定義する.
```kotlin
data class User private constructor(
    val id: String,
    val name: String,
){
    companion object {
        fun of(name: String): User {
            return User(id = "id_$name", name = name)
        }
    }
}

val user = User.of("hondaya")
// desugar
val user = User.Companion.of("hondaya")

```

#### by (delegation)
汎用loggerのdelegationによる実装例
```kotlin
class LoggerDelegation<in T: Any>: ReadOnlyProperty<T, Logger> {
    override fun getValue(thisRef: T, property: KProperty<*>): Logger {
        return getLogger(clazz = getClass(thisRef.javaClass))
    }

    private fun <R: Any> getClass(javaClass: Class<R>): Class<*> {
        return javaClass.enclosingClass?.takeIf {
            it.kotlin.companionObject?.java == javaClass
        } ?: javaClass
    }
}

class HappyUsecase(){
    companion object {
        val LOGGER by LoggerDelegation
    }
}

```

#### Extension functions
SwiftのextensionやRustのtraitのように既存のclassに対して、拡張するようにfunctionを定義することができます。
```kotlin
fun String.capitalize(): String {
    // "hoge" to "Hoge" 
    return capitarizedStr
}
```
ref: https://kotlinlang.org/docs/extensions.html

#### type alias
```kotlin
typealias UserId = String
```

### Functions
#### Named arguments
```kotlin
fun createUser(id: String, name: String, age: Int?, hobby: String?, ): User = ...

// Fxxk: Which field/property is being set to null????
val user = createUser("id", "name", null, null)

// GOAT
val user = createUser(id = "id", name = "name", age = null, hobby = null)
```

#### vararg / spread
vararg ... 可変長引数
`*` spread ... 既存の配列を、可変長の個々の要素として展開して渡す
```kotlin
// stdlib collections #listOf
public fun <T> listOf(vararg elements: T): List<T> = if (elements.size > 0) elements.asList() else emptyList()
```
List<T>を生成するstdlibの`listOf`実装を見てみるとvarargでelementを渡せるようになっている.
`listOf`は任意の数の値を渡せたり、spreadで配列を展開して渡せたりする.

```kotlin
val dockerNetworkLsCmd = listOf("docker", "network", "ls")
val dockerComposeUpCmd = listOf("docker", "compose", "-f", "$filePath", "up", "-d")

fun main(args: Array<String>) {
    val inputArguments = listOf(*args)
}
```

spreadはArray<T>


#### infix
```kotlin
// create Pair
val pair = "miyagawa" to "rebuild.fm"
```
この to はinfix methodで定義されている.
stdlibの実装を見ると,
```kotlin
public infix fun <A, B> A.to(that: B): Pair<A, B> = Pair(this, that)
```
となっていて、Aに対してinfix関数で拡張関数を定義すると、このような呼び出しが可能になる. 

kotestのassertion apiもこの方式で実装されているものが多い
```kotlin
actual shouldBe expected
```

--- 
この記事における標準ライブラリなどは、[JetBrains/kotlin](https://github.com/JetBrains/kotlin) (6ee413875) の実装を参考に執筆しています.
