### 1

```kotlin
fun List<Scenario>.sort(): List<Scenario> {
    return sortedWith(compareBy<Scenario> { it.nextEventAt }
        .thenBy { it.name }
        .thenByDescending { it.isActive })
} 

data class Scenario(
    val name: String,
    val nextEventAt: Long,
    val isActive: Boolean,
)
```

### 2

```kotlin
fun List<UnionNode>.flatDevices(predicate: (DeviceNode) -> Boolean = { true }) {
    // сбор устройств с родительской и дочерних union node
}

data class UnionNode(
    val id: String,
    val name: String,
    val devices: List<DeviceNode>,
    val children: List<UnionNode>,
)

data class DeviceNode(
    val id: String,
    val name: String,
    val type: String,
)
```

### 3

```kotlin
fun parseRecipe(input: String): Recipe {
    // парсинг текста в объект Recipe.
    return input.split("\n").let {
        Recipe(name = it[0], ingredients = it.drop(1))
    }
}
```

### 4

```kotlin
fun calculateTotalCost(ingredients: List<Ingredient>): Double {
    // вычисление общей стоимости ингредиентов.
    return ingredients.sumOf { it.cost }
}
```

### Выводы

Работа над заданием сильнее убедила, что использование стиля ФП, в особенности над коллекциями, позволяет легко решить целый use кейсов и выделить эти чистые функции в общий код.

Также выполнение этого задания помогло мне оценить композицию в ФП, когда отдельные небольшие функции работают вместе для выполнения более сложных задач. 
Подход SRP+ФП сделал код более читаемым и модульным, что помогает не только в разработке, но и в тестировании и обслуживании.

Соблюдать SRP в ООП стиле сложнее при наличии классов и интерфейсов, приходится либо тратить время на группировку функций в классах по их назначению. 
Либо следовать крайности, что в классе содержится лишь один публичный метод, что делает ООП бессмысленным, нивелирует его фишки.
Стиль ФП же естественным образом поддерживает SRP благодаря фокусу на чистых функциях и компоновке. 
