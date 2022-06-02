# kotlin-pattern-matching 
## 加强kt模式匹配, 支持守卫与解构
## 现在match是表达式了
## 加入else分支模式匹配
Support pattern matching with complex patterns

[YouTrack KT-186启发](https://youtrack.jetbrains.com/issue/KT-186) ->  https://youtrack.jetbrains.com/issue/KT-186

[Kotlin KEEP 213启发](https://github.com/Kotlin/KEEP/pull/213) -> https://github.com/Kotlin/KEEP/pull/213

```kotlin
data class Relative(val name: String = "", val relationship: String = "", val age: Int = 0)
data class Staff(val name: String = "Good colleague", val id: Int = 0)


fun main() {
    val example = Staff("jack", 1)

    val num = match(example) {
        //类的KClass对象是单例的, 不会被重复创建消耗内存
        `is`(Relative::class) `if` {
            this.age > 18
        } then { (name, _, age) ->
            println("I'm $name")
            println("My age is $age")
            "表达式1"
        }
        `is`(Staff::class) `if` {
            this.id > 10
        } then { (name, id) ->
            println("fellow $name, id is $id")
            "表达式2"
        }
        `is`(Relative::class) { (name, _, age) ->
            println("I'm $name")
            println("My age is $age")
            "表达式3"
        }
        `is`(Staff::class) `if` {
            true
        } then { (name, id) ->
            println("$name, id is $id")
            "表达式4"
        }
        `else` {
            println("I'm not a relative")
            "表达式5"
        }
    }
}
```
### [源文件](./MatchPatternX.kt "source code")

```kotlin
package com.example.myapplication.tool.extension

import java.util.concurrent.locks.ReentrantLock

@DslMarker
@Target(AnnotationTarget.FUNCTION)
annotation class DSL

@DSL
inline fun <reified T : Any> match(matchObject: T, block: MatchPatternScope<T>.() -> Unit) {
    MatchPatternScope(matchObject).block()
}

class MatchPatternScope<T>(val matchObject: T) {
    val lock = ReentrantLock()
    var doNextStep = true
    var isEnd = false
    fun matchEnd() {
        isEnd = true
    }
}

@DSL
inline fun <reified T : Any, reified V : Any> MatchPatternScope<T>.`is`(matchObjectExpect: V): V {
    lock.lock()
    if (isEnd) {
        //println("match is end")
        return matchObjectExpect
    }
    //println("is 0 doNextStep - $doNextStep")
    if (T::class == V::class) {
        doNextStep = true
        return matchObject as V
    } else {
        doNextStep = false
    }
    //println("is 1 doNextStep - ${doNextStep}")
    return matchObjectExpect
}

@DSL
inline fun <reified T : Any, reified V : Any> MatchPatternScope<T>.`is`(
    matchObjectExpect: V,
    crossinline block: V.(V) -> Unit
) {
    with(this@MatchPatternScope) {
        `is`(matchObjectExpect) `if` { true } then {
            matchObjectExpect.block(matchObjectExpect)
        }
    }
}

context(MatchPatternScope<T>) @DSL
infix fun <T, V> V.`if`(block: V.() -> Boolean): V {
    if (isEnd) {
//        println("match is end")
        return this
    }

    if (!block()) {
        doNextStep = false
    }
//    println("if 2 doNextStep - ${doNextStep}")
    return this
}

context(MatchPatternScope<T>) @DSL
infix fun <T, V> V.then(block: (V) -> Unit) {
    if (isEnd) {
//        println("match is end")
        return
    }
    if (doNextStep) {
        block(this@then)
    }
//    println("then 3 doNextStep - ${doNextStep}")
//    println("final before (end match  ${this@MatchPatternScope.isEnd})")
    if (doNextStep) {
        this@MatchPatternScope.matchEnd()
    }
//    println("final after  (end match  ${this@MatchPatternScope.isEnd})")
    lock.unlock()
}

data class Relative(val name: String = "", val relationship: String = "", val age: Int = 0)
data class Staff(val name: String = "Good colleague", val id: Int = 0)

fun main() {
    val example = Staff("jack", 1)
    match(example) {
        `is`(Relative()) `if` {
            this.age > 18
        } then { (name, _, age) ->
            println("I'm $name")
            println("My age is $age")
        }
        `is`(Staff()) `if` {
            this.id > 0
        } then { (name, id) ->
            println("fellow $name, id is $id")
        }
        `is`(Staff()) { (name, id) ->
            println("$name, id is $id")
        }
    }
}
```
