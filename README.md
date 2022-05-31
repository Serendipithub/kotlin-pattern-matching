# kotlin-pattern-matching
Support pattern matching with complex patterns

```kotlin
data class Relative(val name: String = "", val relationship: String = "", val age: Int = 0)
data class Staff(val name: String = "Good colleague", val id: Int = 0)

fun main() {
    /**
     * With powerful dsl capabilities, you are able to write more powerful pattern matching in kotlin, 
     * including destructuring and conditional guards.
     */
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
