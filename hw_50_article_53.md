### 1. 

Было:

```kotlin
fun HomeNode.findDevice(predicate: (Device) -> Boolean): Device? {
    devices.forEach { device ->
        if (predicate(device)) {
            return@forEach device
        }
    }
    
    return null
}

data class HomeNode(
    val devices: List<Device>,
)

data class Device(
    val id: String,
    val name: String,
)
```

Стало:

```kotlin
interface DevicePredicate {
    fun apply(home: HomeNode): Boolean
}

class DeviceIdPredicate(private val targetId: String) : DevicePredicate {

    override fun apply(home: HomeNode): Boolean {
        return home.devices.any { device ->
            device.id == targetId
        }
    }
}

fun HomeNode.findDevice(predicate: DevicePredicate): Device? {
    devices.forEach { device ->
        if (predicate.apply(device)) {
            return@forEach device
        }
    }

    return null
}

data class HomeNode(
    val devices: List<Device>,
)

data class Device(
    val id: String,
    val name: String,
)
```


### 2

Было:

```kotlin
fun executeTask(callback: () -> Unit) {
    println("Executing task...")
    callback()
}

fun main() {
    executeTask { println("Task complete!") }
}
```

Стало:

```kotlin
interface Command {
    fun execute()
}

class PrintCompletionMessage : Command {
    override fun execute() {
        println("Task complete!")
    }
}

fun executeTask(command: Command) {
    println("Executing task...")
    command.execute()
}

fun main() {
    executeTask(PrintCompletionMessage())
}
```

### Выводы

В коде проекта много мест, подобных `findDevice` из 1 примера. Также есть `findRoom`, `findGroup`, `findHome`, которые принимают предикат.
Вызывающему коду приходится каждый раз писать логику поиска. Таким образом появилось множество одинаковых лямбда-функций поиска по параметру.
Делая рабочую задачу уже начал применять предложенный метод рефакторинга, нахожу его довольно полезным, с помощью которого можно уменьшить и количество дублирующего кода, и количество тест-кейсов. 
Достаточно проверять правильно выбранный фильтр.

В число плюсов можно также выделить, что код становится более декларативным, когда передается готовый фильтр. Кроме этого, мы инкапсулируем логику в классах-фильтрах.

Так что в целом дефункционализация приводит к более структурированной архитектуре, выгодной в определенных контекстах применения, когда есть повторное применение таких функций высшего порядка.