Добавил базовый код по теме полугруппы и моноида. Адаптировал под Kotlin

# Статья 1

```kotlin
interface SemiGroup<T> {

    fun plus(
        left: T,
        right: T,
    ): T
}


interface Monoid<T> : SemiGroup<T> {

    /**
     * Изучив определение моноидов, решил использовать каноничное название нейтрального элемента
     */
    fun neutral(): T
}


fun <T> Iterable<T>.sum(
    monoid: Monoid<T>
): T = fold(monoid.zero()) { left, right ->
    monoid.plus(left, right)
}
```

### Пример 1 с моноидами Min/Max

```kotlin

class Max<T: Comparable<T>>(
    private val zero: T
): Monoid<T> {

    override fun zero() = zero

    override fun plus(left: T, right: T): T{
        return if (left > right) {
            left
        } else {
            right
        }
    }
}

class Min<T: Comparable<T>>(
    private val zero: T
): Monoid<T> {

    override fun zero() = zero

    override fun plus(left: T, right: T): T{
        return if (left < right) {
            left
        } else {
            right
        }
    }
}

fun main() {
    val numbers = listOf(
        1, 2, -24, 95,
    )

    val minValue = numbers.sum(Min(Int.MAX_VALUE))

    println(minValue)
}
```

### Пример 2

Добавим моноиды предикатов для удобного фильтра

```kotlin
class AnyPredicate<T>() : Monoid<Predicate<T>> {
    override fun zero() = Predicate<T> { false }

    override fun plus(
        left: Predicate<T>,
        right: Predicate<T>
    ): Predicate<T> = left.or(right)
}

class StrongPredicate<T> : Monoid<Predicate<T>> {

    override fun zero() = Predicate<T> { false }

    override fun plus(
        left: Predicate<T>,
        right: Predicate<T>
    ) = left.and(right)
}

fun main() {
    val alarmPredicates = buildList<Predicate<Sensor>> {
        add(Predicate { sensor ->
            sensor is Sensor.Valve && sensor.hasAlarm
        })

        add(Predicate { sensor ->
            sensor is Sensor.Humidity && sensor.isDirty
        })
    }

    val houseSensors = listOf(
        Sensor.Valve(hasAlarm = false),
        Sensor.Humidity(isDirty = false), // Этот датчик требует внимания
        Sensor.Valve(hasAlarm = false)
    )

    val houseHealthCheck: Predicate<Sensor> = alarmPredicates.sum(AnyPredicate())

    val hasAlarmsInHouse = houseSensors.any { sensor -> houseHealthCheck.test(sensor) }

    println("Случилось ли что-то в доме: $hasAlarmsInHouse")
}

sealed interface Sensor {

    data class Valve(val hasAlarm: Boolean) : Sensor
    data class Humidity(val isDirty: Boolean) : Sensor
}
```

# Статья 2

```kotlin
interface Group<T> : Monoid<T> {

    fun inverse(item: T): T
}

interface SemiRing<T> {

    val zero: T // Нейтральный элемент для сложения
    val one: T  // Нейтральный элемент для умножения

    fun T.plus(other: T): T  
    fun T.times(other: T): T 
}
```


### Пример 1

```kotlin
data class LightState(val brightness: Int)

// Абелева группа для дельты изменений
class LightDeltaGroup : Group<Int> {
    val identity: Int = 0

    fun combine(a: Int, b: Int): Int = (a + b).coerceIn(-100, 100)

    override fun inverse(item: Int): Int {
        return -item
    }

    override fun zero() = 0

    override fun plus(left: Int, right: Int): Int {
        return left + right
    }
}

class LightManager {
    private val group = LightDeltaGroup()
    var currentState = LightState(50)
    private val pendingChanges = mutableListOf<Int>()

    // Локальное изменение (пользователь крутит слайдер)
    fun applyDelta(delta: Int) {
        pendingChanges.add(delta)
        currentState = LightState(group.combine(currentState.brightness, delta))
    }

    // Реверт операции (например, не получили от сокета подтверждение выполнения команды)
    fun rollbackDelta(delta: Int) {
        pendingChanges.remove(delta)
        val inverseDelta = group.inverse(delta)
        currentState = LightState(group.combine(currentState.brightness, inverseDelta))
    }
}
```

### Пример 2

```kotlin
// Тропическое полукольцо
object ShortestPathSemiring : Semiring<Double> {
    override val zero = Double.POSITIVE_INFINITY
    override val one = 0.0

    override fun Double.plus(other: Double): Double = minOf(this, other)
    override fun Double.times(other: Double): Double = this + other
}

// расстояние между узлами сети умного дома
fun <T> distance(matrix: Array<Array<T>>, semiring: Semiring<T>): Array<Array<T>> {
    val n = matrix.size
    val result = matrix.map { it.clone() }.toTypedArray()
    with (semiring) {
        for (k in 0 until n) {
            for (i in 0 until n) {
                for (j in 0 until n) {
                    result[i][j] = result[i][j] + (result[i][k].times(result[k][j]))
                }
            }
        }
    }
    return result
}
```

### Выводы

Как сказал Степан во второй статье "Математики, как и программисты, стремятся к обобщению".
Согласен с этим утверждением, вот только по-настоящему обобщить можно только используя единый инструмент - математический аппарат.
В мейнстриме обобщения часто не имеют математической строгости и заключается в более линейном подходе.
С математикой же можно написать алгоритм, который считает и самый дешевый тариф, и высчитать самые релевантные сценарии автоматизации умного дома для пользователя.

Используя свойства групп и моноида нашел реальные способы улучшить качество кода: во-первых, сделать его более декларативным и обобщенным.
Особенно в части предикатов: я стараюсь воспринимать составные условия на множестве элементов как список предикатов и рад получить инсайт, позволяющий элегантно составить список предикатов.

Уже начал глубже погружаться в алгебраические структуры, составляю доклад на команду, как на примере нашей кодовой базы можно улучшить качество кода, добавив ему математической строгости и лаконичности.