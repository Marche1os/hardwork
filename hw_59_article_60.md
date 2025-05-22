### Пример 1

```kotlin

// до
fun sendAnalytics(items: List<ScenarioGridItem>) {
    items.forEach { item ->
        if (item is ScenarioGridItem.Scenario) {
            analytics.sendEvent("")
        }
    }
}

// после
fun sendAnalytics(items: List<ScenarioGridItem>) {
    items
        .filterIsInstance<ScenarioGridItem.Scenario>()
        .forEach { scenario ->
            analytics.sendEvent("")
        }
}

sealed interface ScenarioGridItem {

    data class Scenario(val name: String) : ScenarioGridItem

    /**
     * Тип для заполнения пустого простанства ячейки сетки
     */
    object Dummy : ScenarioGridItem
}
```

### Пример 2

```kotlin

class RoomDtoConverter @Inject constructor() : Mapper<RoomDto, Room> {
    
    // до
    override fun map(dtos: List<RoomDto>): List<Room> {
        val rooms = mutableListOf<Room>()
        
        dtos.forEach { dto ->
            if (dto.header != null && dto.deviceIds != null) {
                rooms.add(
                    Room(
                        header = dto.header,
                        deviceIds = dto.deviceIds
                    )
                )
            }
        }
        
        return rooms
    }
    
    // после
    override fun map(dtos: List<RoomDto>): List<Room> {
        val rooms = mutableListOf<Room>()

        dtos
            .filter { dto -> dto.header != null && dto.deviceIds != null }
            .map { dto ->
                Room(
                    header = dto.header,
                    deviceIds = dto.deviceIds
                )
            }

        return rooms
    }
}

@Serializable
data class RoomDto(
    
    @SerialName("header")
    val header: String?,
    
    @SerialName("device_ids")
    val deviceIds: List<String>?
)

data class Room(
    val header: String,
    val deviceIds: List<String>
)
```

### Пример 3

```kotlin
// до
fun mapVirtualRoomItems(node: UnionNode): List<RoomItem> {
    val rooms = mutableListOf<RoomItem>()
    
    val sensors = mapSensors(node)
    val templates = mapTemplates(node)
    
    rooms += sensors
    rooms += templates
    
    return rooms
}

// после
fun mapVirtualRoomItems(node: UnionNode): List<RoomItem> {
    return listOf(
        mapSensors(node),
        mapTemplates(node)
    ).fold(emptyList()) { acc, list -> acc + list }
}
```

### Пример 4

```kotlin
// до
val seen = mutableSetOf<String>()
val uniqueUsers = mutableListOf<User>()
for (user in users) {
    if (user.email !in seen) {
        uniqueUsers.add(user)
        seen.add(user.email)
    }
}

// после
val (uniqueUsers, _) = users.fold(Pair(mutableListOf<User>(), mutableSetOf<String>())) { (list, seen), user ->
    if (user.email !in seen) {
        list.add(user)
        seen.add(user.email)
    }
    Pair(list, seen)
}
```

### Пример 5

```kotlin
// до
var completed = 0
for (task in tasks) {
    if (task.isDone) completed++
}

// после
val completed = tasks.fold(0) { acc, task -> acc + if (task.isDone) 1 else 0 }
```

### Выводы

Выполняя это задание, я научился переводить обычные итеративные циклы в более абстрактные и лаконичные конструкции, такие как map, filter, fold и reduce. 
Это задание помогло лучше понять стандартные функции работы с коллекциями и их возможности.  

Практические навыки, которые я получил:

- Научился определять, когда привычный for-цикл можно заменить на map, filter, fold или reduce для повышения читаемости и компактности кода. 
- Укрепил навыки работы с лямбда-выражениями и функциями высшего порядка. 
- Стал увереннее использовать стандартную библиотеку Kotlin для обработки коллекций. 
- Понял, как такие абстракции позволяют писать более декларативный и безопасный код. 
- Рассмотрел плюсы и минусы такого подхода, например, читаемость, производительность и удобство отладки.

В целом, задание способствовало формированию правильного подхода к написанию современного кода. 
Теперь я чаще задумываюсь, можно ли вместо обычного цикла применить функции стандартной библиотеки, чтобы сделать решение короче, понятнее и выразительнее.
При этом когда я провожу ревью чужого кода, то задумываюсь над конструкциями с обычным циклом и какой-то фильтрацией и как можно такие конструкции заменить готовой абстракцией из функциональной парадигмы.