### 1 **Было**. Модель, представляющая сущность UI-слоя. 

```kotlin
data class UiItem(val items: List<Scenario>)

data class Scenario(
    val scenarioId: String,
    val title: String,
    val subtitle: String,
    val launchTimeTimestamp: Long,
)

class FragmentScreen : Fragment {
    private val columns = 4

    fun showUI(uiItem: UiItem) {
        uiItem.items
            .chunked(columns)
            .forEach { row ->
                row.forEach { scenario ->
                    // Отображаем данные scenario на UI
                }
            }
    }
}
```

**Стало.** Создана структура данных, имеющая необходимый для задачи набор операций для добавления элементов. 
И метод для получения гридового представления нужных нам данных в нужном виде. 
Теперь при необходимости отображения элементов сценариев в виде сетки мы можем использовать единую структуру с нужными настройками по количеству отображаемых элементов и по количеству столбцов. 

```kotlin
class GridStructure(
    private val columns: Int,
    private val maxElements: Int,
) {
    
    fun add(scenario: Scenario) {
        // Добавление нового элемента в грид с учетом параметров columns и maxElements
    }
    
    fun grid(): ScenarioGrid {
        TODO("предоставление данных в гридовом виде")
    }}

@JvmInline
value class ScenarioGrid(val value: List<List<Scenario>>)

data class UiItem(val scenarioGrid: ScenarioGrid)

data class Scenario(
    val scenarioId: String,
    val title: String,
    val subtitle: String,
    val launchTimeTimestamp: Long,
)

class FragmentScreen : Fragment {
    fun showUI(uiItem: UiItem) {
        uiItem.scenarioGrid
            .value
            .forEach { row ->
                row.forEach { scenario ->
                    // Отображаем данные scenario на UI
                }
            }
    }
}
```

### 2. 

```kotlin
class WidgetsToBannersMapper {
    fun map(widgets: List<Width>): List<Banner> {
        // Перевод
        
        return emptyList()
    }
}

data class Widget(
    val widgetName: String,
    val imageUrl: String,
    val width: Int,
    val height: Int,
)

data class Banner(
    val width: Int,
    val height: Int,
    val imageUrl: String,
)
```

**Стало** Создан единый конвертер типов. Теперь использующий класс может не знать о конкретном конвертере, а только указать нужный входной и выходной тип.
Появляется возможность динамически выбирать нужный конвертер, нужно только дополнительно сделать проверку, чтобы на этапе компиляции выдавалась ошибка, если для указанных типов нет нужного конвертера.

```kotlin
interface Converter<Input, Output> {
    fun convert(input: Input): Output
}

class WidgetsToBannersConverter: Converter<List<Widget>, List<Banner>> {
    
    override fun convert(input: List<Widget>): List<Banner> {
        TODO("Маппинг сущностей") 
    }
}

data class Widget(
    val widgetName: String,
    val imageUrl: String,
    val width: Int,
    val height: Int,
)

data class Banner(
    val size: Pair<Width, Height>,
    val width: Int,
    val height: Int,
    val imageUrl: String,
)

@JvmInline
value class Width(val value: Int)

@JvmInline
value class Height(val value: Int)
```

### 3. 

**Было**
```kotlin
class AnimationUtil {
    fun fadeIn(view: View) {
        view.animate().alpha(1.0f).setDuration(500).start()
    }

    fun slideUp(view: View) {
        view.animate().translationY(0f).setDuration(500).start()
    }
}
```

**Стало**

```kotlin
interface ViewAnimation {
    fun apply(view: View)
}

class FadeInAnimation : ViewAnimation {
    override fun apply(view: View) {
        view.animate().alpha(1.0f).setDuration(500).start()
    }
}

class SlideUpAnimation : ViewAnimation {
    override fun apply(view: View) {
        view.animate().translationY(0f).setDuration(500).start()
    }
}

fun applyAnimation(view: View, animation: ViewAnimation) {
    animation.apply(view)
}
```

Создаем иерархию анимаций, что позволит "поставлять" вьюхам нужные типы анимации, при этом сама view не будет знать об анимации.
Настройка анимаций поднялась наверх.

### 4. 

**Было**

```kotlin
private fun List<Widget>.sort(): List<Widget> = sortedWith(
    compareBy(
        { it.nextEventAt < platformClock.now() || it.nextEventAt == 0L },
        { it.nextEventAt },
        { it.name },
    ),
)

data class Widget(
    val nextEventAt: Long,
    val name: String,
)
```

**Стало**

```kotlin
interface Sortable {
    val isImmediate: Boolean
    val nextEventAt: Long
    val name: String
}

private fun <T : Sortable> List<T>.sort(platformClock: PlatformClock): List<T> = sortedWith(
    compareBy<T>(
        { it.nextEventAt < platformClock.now() || it.nextEventAt == 0L },
        { it.nextEventAt },
        { it.name },
    )
)
```

Теперь метод sort является полиморфным и может работать с любым типом данных, который реализует интерфейс Sortable. 
Это позволяет использовать этот метод для сортировки списков любых объектов, которые соответствуют подходу, определенному в интерфейсе.

Также для работы функции необходимо теперь передать экземпляр platformClock, чтобы метод мог сравнивать текущее время системы, что делает функцию более чистой и не зависимой от глобальных состояний.


### 5.

**Было**

```kotlin
fun generateHomeHeader(items: List<AdapterItem>) {
    val header = items.filterIsInstance<HomeHeader>().firstOrNull() ?: return
    // Дальнейшая работа с header
}

data class HomeHeader(
    val title: String,
    val backgroundUrl: String,
): AdapterItem

interface AdapterItem
```

**Стало**

```kotlin
inline fun <reified R> Iterable<*>.firstInstanceOfOrNull(): R? {
    forEach { item -> 
        if (item is R) return item
    }
    
    return null
}

data class HomeHeader(
    val title: String,
    val backgroundUrl: String,
): AdapterItem

interface AdapterItem

fun generateHomeHeader(items: List<AdapterItem>) {
    val header = items.firstInstanceOfOrNull<HomeHeader>() ?: return
    // Дальнейшая работа с header
}
```

Создана чистая функция, которая возвращает первый экземпляр указанного типа. Сама функция ничего не знает о типе и может использовать для любого содержимого списка.

**Выводы**

На рабочих проектах порой сталкиваюсь с недостаточно полиморфным кодом. Вижу объем кода с логикой, которая нужна и мне, но ввиду того, что тип задан конкретный, не получается сходу переиспользовать этот кусок кода.
Хотя его логика применима и к другим типам. Потому планирую на работе в команде и рассказывать о лучших практиках написания такого полиморфного кода, и на пулл-реквестах обращать внимание на куски кода, которые можно сделать иначе, более полиморфными.
Сам по себе полиморфизм это не просто техника, но и образ мышления, поэтому важно уметь видеть общую структуру кода и в целом мыслить структурами, а не только находиться на уровне мышления о коде.
Я сторонник идеи о том, что с развитием проекта время на реализацию новых функций в целом должно становиться меньше, а не оставаться на уровне или даже расти.
Потому что мы должны писать полиморфный код, который будет переиспользоваться и снижать время на разработку сценариев, которые уже были разработаны.
Хорошим примером является разработка UI интерфейсов и методов манипуляции с ним. Для этого создается дизайн-система и весь экран строится из по-сути реализованных UI-компонентов, которые можно легко настраивать в случае необходимости.
В какой-то мере это тоже часть полиморфизма на уровне UI интерфейса.

В целом, правильное использование полиморфизма часто ведет к более чистому и понятному коду. 
Код становится менее зашумленным и более ориентированным на поведение, а не на конкретные реализации и данные, что улучшает читаемость и облегчает его поддержку.

В рамках осмысления правильного полиморфного кода не следует забывать и о потенциальных подводных камнях. 
Неуместное или избыточное использование полиморфизма может увести к усложнению архитектуры и к снижению производительности, если из-за абстракций вводится большое количество сущностей. 
К тому же, необходимость поддерживать общий интерфейс может ограничить возможности конкретных реализаций.

