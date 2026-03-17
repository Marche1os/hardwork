### 1 

```kotlin
fun openBrowser(url: String) {
    if (url.startWith("http://") || url.startWith("https://")) {
        // Открытие браузера
    }
}
```

```kotlin
@JvmInline
value class Url(val value: String) {
    
    init {
        // Ассерт, гарантирующий невозможность хранения некорректной для нас схемы
        require(value.startWith("http://") || value.startWith("https://"))
    }
}

fun openBrowser(url: Url) {
    // Открытие точно корректной ссылки в браузере
}
```

### 2

```kotlin
/**
 * Применение градиента в Android Compose. 
 * Если размер [colors] меньше 2, то фреймворк Compose UI упадет в рантайме 
 */
fun applyGradient(colors: List<Color>) {
    // применение градиента
}
```

```kotlin
@JvmInline
value class GradientColors(
    val colors: List<Color>,
) {
    
    init {
        require(colors.size >= 2)
    }
}

/**
 * Применение градиента в Android Compose.
 */
fun applyGradient(gradient: GradientColors) {
    // применение градиента
}
```

### 3

Рекурсивная схема умного дома, в котором могут находиться устройства и группы, не относящиеся к комнатам.
И комнаты, в которых могут находиться устройства и группы. В группе - список устройств.
Работать с этим тяжело и требуется постоянно проверять [UnionType]. 
Кроме этого, можно забыть сделать проверку и работать, например, с группой устройств. А ожидалась комната...

```kotlin
data class Union(
    val node: UnionNode,
    val deviceIds: List<String>,
    val children: List<Union>,
)

data class UnionNode(
    val name: String,
    val id: String,
)

enum class UnionType {
    ROOM,
    HOME,
    GROUP,
}
```

```kotlin
sealed class Union(
    val deviceIds: List<String>,
) {

    data class Room(
        val data: RoomData,
        val deviceIds: List<String>,
        val groupIds: List<String>,
    ) : Union(deviceIds)

    data class Home(
        val data: HomeData,
        val deviceIds: List<String>,
        val groupIds: List<String>,
    ) : Union(deviceIds)

    data class GroupDevices(
        val data: GroupData,
        val deviceIds: List<String>,
    ) : Union(deviceIds)
}

data class GroupData(
    val groupId: String,
    val groupName: String,
)

data class HomeData(
    val homeName: String,
    val homeId: String,
)

data class RoomData(
    val roomName: String,
    val parentId: String,
    val roomId: String,
)
```

### Выводы

Благодаря вашей Школе открыл для себя увлекательное направление типов данных, проектирования по контракту, инварианты и т.д.
Поэтому и до выполнения текущего задания на рабочем проекте продвигал идеи, помогающие создавать качественные программные системы.
Сюда относится и использования ассертов, как часть парадигмы Design by contract. 

При разработке новых фичей следую подходу: доменный тип + smart constructor и инвариантом в теле конструктора. 
А скоро еще интегрируем библиотеку Arrow (требуется только поднятие Котлина до 2.0+).
Что позволит при обращении к smart конструктору либо получить создаваемый тип, либо получить тип-ошибку инициализации, которая не вызовет возникновения исключительной ситуации и которую нужно будет явно обработать.

В иных местах используются ассерты (в котлин это предусловия из Preconditions). 
Поначалу были опасения в их использовании, так как использование их чревато падениями в рантайме.
Но на самом деле я думаю следовать такой парадигме этого хорошо. 
Разработчики более сконцентрированно и дисциплинрованно пишут код, думая о потоке данных. 
И бонусом, Preconditions и Contracts помогают тайпчекеру в выводе типов.
Поэтому, если мы сделали, например, `check(state is Error)`, то далее мы работаем именно с типом `Error`.

