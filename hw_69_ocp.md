### 1

```kotlin
// Базовая функция обработки — не меняем, а стратегии подставляем извне
fun processTrafficData(
    data: TrafficData,
    predictionStrategy: (TrafficData) -> Prediction
): Prediction = predictionStrategy(data)

// Пример новой стратегии — просто передаём новую функцию
val simpleStrategy: (TrafficData) -> Prediction = { data ->
    Prediction("Прогноз загруженности: ${data.speed}")
}

val aiBasedStrategy: (TrafficData) -> Prediction = { data ->
    // более сложный анализ
    Prediction("AI прогноз: ${data.speed * 0.8}")
}
```

### 2 

```kotlin
typealias MapLayer = String
typealias MapTransformer = (List<MapLayer>) -> List<MapLayer>

fun renderMap(layers: List<MapLayer>, transformers: List<MapTransformer>) =
    transformers.fold(layers) { acc, transformer -> transformer(acc) }

val addRoadStyles: MapTransformer = { layers -> layers + "Road Styles" }
val highlightTraffic: MapTransformer = { layers -> layers + "Traffic Highlights" }

val rendered = renderMap(listOf("Base Layer"), listOf(addRoadStyles, highlightTraffic))
```

### 3

```kotlin
typealias NotificationRule = (HiveData) -> Notification?

fun processNotifications(data: HiveData, rules: List<NotificationRule>): List<Notification> =
    rules.mapNotNull { rule -> rule(data) }

val tempThresholdRule: NotificationRule = { hive ->
    if (hive.temperature > 35) Notification("Слишком жарко!") else null
}

val lowActivityRule: NotificationRule = { hive ->
    if (hive.activityLevel < 20) Notification("Снижение активности!") else null
}

data class HiveData(
    val temperature: Int,
    val activityLevel: Int,
)
```

### 4

```kotlin
typealias WorkoutFilter = (List<Workout>) -> List<Workout>

fun generatePlan(baseWorkouts: List<Workout>, filters: List<WorkoutFilter>): List<Workout> =
    filters.fold(baseWorkouts) { acc, filter -> filter(acc) }

val cardioOnly: WorkoutFilter = { list -> list.filter { it.type == "Cardio" } }
val noInjuries: WorkoutFilter = { list -> list.filter { !it.isInjuryRisk } }

val plan = generatePlan(allWorkouts, listOf(cardioOnly, noInjuries))
```

### Выводы

В ходе выполнения задания увидел, как OCP принцип соблюдается в функциональном стиле и ООП. 
 
Раннее не задумывался, что алгебраические типы данных (в котлине это sealed классы/интерфейсы) тоже поддерживают OCP. 
Ведь действительно, во-первых, весь sealed описан в одном модуле и за рамками этого модуля запрещено расширять АТД.
Во-вторых, каждый новый вариант sealed класса не затрагивает существующие типы.

На рабочем проекте по наитию используются функции высшего порядка для задания поведения. 
Например, когда некая фильтрующая функция принимает функцию-предикат как параметр. 
Но видно, что такое решение было сделано инстинктивно для конкретного сценария, но общего понимания использования не было: поэтому в похожих других местах в проекте используется другой, более громоздкий и сложный подход недекларативный подход. 

Про абстракции. В какой-то момент, по крайней мере в мире Android разработки, сообщество разработчиков просто стало одержимым идеями из книги "Чистая архитектура" и в проектах стали лепить столько абстракций, что это просто стало мешать.
И разработчики Android выпускали свои гайды по архитектуре приложений с большим количеством абстракций, при этом на простых примерах. На операцию "загрузить для экрана данные из одной ручки, сохранить в БД и показать" создавалось по 10-20 классов!
Благо, в какой-то момент они одумались и навешали предупреждающие блоки, что одержимость абстракциями не всегда правильный путь и нужно думать, нужны ли они в конкретном случае.