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

import java.util.concurrent.atomic.AtomicInteger
import java.util.concurrent.locks.ReentrantLock
import kotlin.reflect.KClass

@DslMarker
@Target(AnnotationTarget.FUNCTION)
annotation class DSL

@DSL
inline fun <reified T : Any, V> match(
    matchObject: T,
    block: MatchPatternScope<T, V>.() -> Unit
): V {
    val scope = MatchPatternScope<T, V>(matchObject)
    scope.block()
    return scope.result!!
}

class MatchPatternScope<T, V>(val matchObject: T) {
    //无法从语法层面实现每次只对一个case进行匹配，因为case可能会有多个，所以需要一个锁
    val lock = ReentrantLock()

    //是否进行下一步操作
    var doNextStep = true

    //是否匹配成功
    var isEnd = false

    //设置匹配成功
    fun matchEnd() {
        isEnd = true
    }

    //返回值
    var result: V? = null

    //用于解决并发问题, 防止else先被执行
    val intoTimes = AtomicInteger(0)
}

@DSL
inline fun <reified T : Any, reified V : Any, reified K : Any> MatchPatternScope<T, K>.`is`(
    //这个参数仅用于匹配类型, 必须要有, 不然写不成 dsl 形式
    matchObjectExpect: KClass<V>
): V? {
    intoTimes.incrementAndGet()
    lock.lock()
    if (isEnd) {
        //println("match is end")
        return null
    }
    //println("is 0 doNextStep - $doNextStep")
    if (T::class == V::class) {
        doNextStep = true
        return matchObject as V
    } else {
        doNextStep = false
    }
    //println("is 1 doNextStep - ${doNextStep}")
    return null
}

@DSL
inline fun <reified T : Any, reified K : Any> MatchPatternScope<T, K>.`else`(block: T.() -> K) {
    intoTimes.incrementAndGet()
    while (true) {

        if (intoTimes.get() > 1) {
            "here1".println()
            continue
        }
        if (intoTimes.get() == 1) {
            if (!isEnd) {
                this.result = this.matchObject.block()
            }
            return
        }
    }
}

@DSL
inline fun <reified T : Any, reified V : Any, reified K : Any> MatchPatternScope<T, K>.`is`(
    matchObjectExpect: KClass<V>,
    crossinline block: V.(V) -> K
) {
    with(this@MatchPatternScope) {
        `is`(matchObjectExpect) `if` { true } then {
            it.block(it)
        }
    }
}

context(MatchPatternScope<T, K>) @DSL
infix fun <T, V, K> (V?).`if`(block: V.() -> Boolean): V? {
    if (this == null) {
        //this 为 null时，说明上一步没有匹配到，直接返回null
        return null
    }

    if (!this!!.block()) {
        //this 不为 null，但是 block 不为 true，说明上一步匹配到了，但是 block 不满足，下一步就不执行了
        //设置 doNextStep 为 false
        doNextStep = false
    }
//    println("if 2 doNextStep - ${doNextStep}")
    return this
}

context(MatchPatternScope<T, K>) @DSL
infix fun <T, V, K> (V?).then(block: (V) -> K?) {
    if (isEnd) {
//        println("match is end")
        return
    }
    if (doNextStep) {
        this@MatchPatternScope.result = block(this@then!!)
    }
//    println("then 3 doNextStep - ${doNextStep}")
//    println("final before (end match  ${this@MatchPatternScope.isEnd})")
    if (doNextStep) {
        this@MatchPatternScope.matchEnd()
    }
//    println("final after  (end match  ${this@MatchPatternScope.isEnd})")

    lock.unlock()
    intoTimes.decrementAndGet()
}

data class Relative(val name: String, val relationship: String, val age: Int)
data class Staff(val name: String, val id: Int)

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
