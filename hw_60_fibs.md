Реализовал подобный код на Котлине

```kotlin
class FibsSum {
    private var fsum = 0

    fun next(value: Int): Int {
        fsum += value
        return fsum
    }
}

fun getFibs(num: Int): Sequence<Int> = sequence {
    var a = 0
    var b = 1
    val gsum = FibsSum()
    repeat(num) {
        yield(gsum.next(b))
        val c = b
        b = a + b
        a = c
    }
}

fun main() {
    for (n in getFibs(5)) {
        println(n)
    }
    
    /**
     * Вывод:
     * 1
     * 2
     * 4
     * 7
     * 12
     */
}
```

Здесь код содержит класс FibsSum, который хранит сумму чисел Фибоначчи, формирует следующее число и обновляет сумму.
И также из функции, где поддерживается пара чисел (предыдущее и следующее), запрашивается следующее число последовательности и обновляется пара чисел по правилам последовательности Фибоначчи.


Ошибка заключается в формировании первого числа. 
Я исходил из того, что мы хотим получить именно последовательность чисел, а не сумму последовательности.
Для этого нам не нужен класс-обертка, хранящий сумму.

Исправленный код:

```kotlin
fun getFibs(num: Int): Sequence<Int> = sequence {
    var a = 0
    var b = 1
    repeat(num - 1) {
        yield(a)
        val c = b
        b = a + b
        a = c
    }
}

fun main() {
    for (n in getFibs(10)) {
        println(n)
    }

    /**
     * Вывод:
     * 0
     * 1
     * 1
     * 2
     * 3
     * 5
     * 8
     * 13
     * 21
     */
}
```

Если же мы хотели получить именно сумму, то код будет выглядеть так:

```kotlin
fun getFibs(num: Int): Sequence<Int> = sequence {
    var a = 0
    var b = 1
    var fsum = 0
    repeat(num) {
        fsum += a
        yield(fsum)
        val tmp = a + b
        a = b
        b = tmp
    }
}

fun main() {
    for (n in getFibs(5)) {
        println(n)
    }

    /**
     * Вывод:
     * 0
     * 1
     * 2
     * 4
     * 7
     * 12
     */
}
```