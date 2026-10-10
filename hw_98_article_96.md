## Проверка интерфейсов на композит

Для каждого интерфейса я задавал себе один и тот же вопрос: могу ли я написать для него осмысленные `NullX` и `CompositeX`, 
не придумывая данные из воздуха и не делая выбор "наугад, какой результат вернуть".

### 1. CompanionAnalytics - хороший интерфейс

```kotlin
interface CompanionAnalytics {
    fun logEvent(event: AnalyticsEvent, vararg params: AnalyticsParam)
}
```

Метод ничего не возвращает, то есть это по сути Команда. Тут композит пишется сразу и без вопросов:

```kotlin
object NoOpAnalytics : CompanionAnalytics {
    override fun logEvent(event: AnalyticsEvent, vararg params: AnalyticsParam) = Unit
}

class CompositeAnalytics(
    private val targets: List<CompanionAnalytics>,
) : CompanionAnalytics {
    override fun logEvent(event: AnalyticsEvent, vararg params: AnalyticsParam) {
        targets.forEach { it.logEvent(event, *params) }
    }
}
```

`NoOpAnalytics` нужен в тестах и в debug сборках, где события слать не надо. `CompositeAnalytics` позволяет отправлять одно событие сразу в несколько систем, и клиент об этом ничего не знает.
Сюда же ложится декоратор, о котором я писал в прошлом задании, когда `PARAM_PLATFORM` был вписан руками в 12 методов:

```kotlin
class PlatformAnalytics(
    private val inner: CompanionAnalytics,
) : CompanionAnalytics {
    override fun logEvent(event: AnalyticsEvent, vararg params: AnalyticsParam) {
        inner.logEvent(event, PARAM_PLATFORM toStringParam PLATFORM_ANDROID, *params)
    }
}
```

В DI это собирается в одну строку: `PlatformAnalytics(CompositeAnalytics(listOf(firebase, appMetrica)))`.
Платформа это как раз сквозная функциональность, и декоратор тут подходит лучше всего.

Хорошим интерфейс делают две вещи. Первая: метод ничего не возвращает, и спросить у аналитики что-то нельзя.
Вторая: параметры `AnalyticsParam` это простые значения, абстракция через них не протекает.
Если бы в этот интерфейс кто-то добавил, например, `fun isEnabled(): Boolean`, композит сразу бы сломался.
Что должен вернуть `CompositeAnalytics`: `true`, если включена хотя бы одна система, или только если включены все? 
На этот вопрос нет правильного ответа, поэтому такой метод в этом интерфейсе лишний.

### 2. Фильтры дома - хорошая абстракция, но композит так и не сделали

В прошлом задании я ругал `FilterPredicate` как абстракцию про запас. Посмотрел на него еще раз, уже через композит:

```kotlin
internal interface FilterPredicate : Predicate<DeviceNode>

internal object NoneFilter : FilterPredicate { ... }    // пропускает все устройства
internal object LightFilter : FilterPredicate { ... }
internal object ProblemsFilter : FilterPredicate { ... }
internal object ClimateFilter : FilterPredicate { ... }
```

Сигнатура `(DeviceNode) -> Boolean` для композита почти идеальна, потому что `Boolean` замкнут относительно `and` и `or`.
Null Object при этом уже есть: `NoneFilter` возвращает `true` для любого устройства.
Композит пишется в пару строк:

```kotlin
typealias DeviceFilter = (DeviceNode) -> Boolean

val NoneFilter: DeviceFilter = { true }

fun allOf(vararg filters: DeviceFilter): DeviceFilter = { node -> filters.all { it(node) } }
fun anyOf(vararg filters: DeviceFilter): DeviceFilter = { node -> filters.any { it(node) } }
```

Если понадобится выбрать несколько фильтров сразу (например, свет и климат), это будет `anyOf(LightFilter, ClimateFilter)`.
Комбинации вкладываются друг в друга: `allOf(anyOf(LightFilter, ClimateFilter), ProblemsFilter)`. Ни `when`, ни новых классов не нужно.

Самое интересное, что я здесь заметил: `NoneFilter` нейтрален только для `allOf`.
Для `anyOf` нейтральным был бы фильтр, который всегда возвращает `false`. Получается, Null Object это не просто заглушка, а нейтральный элемент конкретного правила объединения. 
Если правило поменять, поменяется и Null Object.

Так что сама абстракция хорошая. Плохим было другое: я завернул ее в свой интерфейс поверх androidx `Predicate`, а композицию, ради которой она вообще нужна, так нигде и не использовал.

### 3. DeviceStateConverter - плохой интерфейс

```kotlin
interface DeviceStateConverter<S : DeviceState> {
    fun convert(type: DeviceType, newState: AttributeState, oldState: S? = null): S
    fun convert(state: S, withReadOnly: Boolean = false): AttributeState
}
```

Попробовал написать `CompositeConverter<S>`, и уже на первом методе застрял.
Каждый конвертер собирает `S` целиком, с нуля. Если внутри композита два конвертера, получается два полных состояния, и что с ними делать, непонятно.
Взять первое? Тогда второй конвертер не нужен. Склеить? Для произвольного `S` склеивать нечем.
С Null Object то же самое: чтобы вернуть `S`, нужно откуда-то взять состояние по умолчанию, а оно у каждого устройства свое (`defaultState` лежит внутри каждого конвертера).
Остается только декоратор, например с логированием, но, декоратор можно сделать почти из чего угодно, и хорошим интерфейс это не делает.

При этом обратное направление композит переносит нормально.
`AttributeState` устроен как коллекция атрибутов и умеет `replace`, поэтому композит сериализаторов просто сливает их результаты, а Null Object возвращает пустой `AttributeState()`.
Выходит, одна половина интерфейса композитная, а другая нет. Это еще одно подтверждение того, что здесь склеены две роли.

Как бы я переделал. Разбор ответа бэкенда сводится к замыканию операций на `S`: на вход приходит состояние, на выходе то же состояние, только обновленное:

```kotlin
fun interface StateReducer<S> {
    fun reduce(state: S, attrs: AttributeState): S
}

class IdentityReducer<S> : StateReducer<S> {
    override fun reduce(state: S, attrs: AttributeState) = state
}

class CompositeReducer<S>(
    private val reducers: List<StateReducer<S>>,
) : StateReducer<S> {
    override fun reduce(state: S, attrs: AttributeState) =
        reducers.fold(state) { acc, reducer -> reducer.reduce(acc, attrs) }
}
```

Датчик дыма тогда собирается из маленьких кусочков, каждый из которых отвечает за свое поле:

```kotlin
val smokeReducer = CompositeReducer(
    listOf(
        StateReducer<SmokeSensorState> { s, a -> s.copy(attributeState = s.attributeState.replace(a)) },
        StateReducer { s, a -> s.copy(isOnline = a.getBool(AttrKeys.ONLINE) ?: s.isOnline) },
        StateReducer { s, a ->
            s.copy(
                isSmoke = a.getBool(AttrKeys.SMOKE_STATE) ?: s.isSmoke,
                lastSync = a.getLastSync(AttrKeys.SMOKE_STATE) ?: s.lastSync,
            )
        },
    )
)

val state = smokeReducer.reduce(oldState ?: defaultState, newState)
```

Состояние по умолчанию переехало туда, где его действительно знают, то есть на сторону вызова. Обновление `isOnline` и `attributeState` одинаково у всех устройств, и такие редьюсеры можно переиспользовать, а не копировать из датчика в датчик.
Сериализация уходит в отдельный `StateSerializer<S>`, и датчикам его просто не нужно реализовывать.

## Выводы

Проверка "можно ли сделать композит" оказалась очень практичной. 
RAP говорит о качестве абстракции только тогда, когда реализаций уже много, а здесь хватает пары минут и трех строк кода в голове, и делать это можно еще до того, как появилась вторая реализация.

По трем примерам видно, что делает интерфейс хорошим:

- метод ничего не возвращает (аналитика), тогда композит получается всегда;
- результат замкнут, как `Boolean` у фильтров или `S -> S` у редьюсера, и для него есть естественное правило объединения;
- результат устроен как коллекция (`AttributeState`), и его можно сливать.

А вот признаки плохого интерфейса:

- метод возвращает целый объект, собранный с нуля, и два таких результата нечем объединить;
- для Null Object приходится выдумывать данные (`defaultState`);
- в командном интерфейсе появляется метод-вопрос, и непонятно, что должен ответить композит;
- композит получается только для части методов. Это почти наверняка значит, что в интерфейсе несколько ролей.

Больше всего меня удивил последний пункт. В прошлом задании я пришел к тому, что `DeviceStateConverter` надо разделить, рассуждая через ISP и пустые реализации у датчиков.
Сейчас к той же границе я пришел совсем с другой стороны: одна половина композитная, другая нет. Два разных подхода дали один и тот же ответ, и это добавляет уверенности, что граница правильная.

Еще пришлось пересмотреть свою оценку `FilterPredicate`. Раньше я записал его в лишние абстракции и, в общем, был прав, но причину назвал не до конца. Сам предикат как абстракция один из лучших в проекте. Лишними были обертка над androidx и то, что ни одной композиции никто не написал. Бесполезная абстракция и плохая абстракция это разные вещи. Здесь абстракция хорошая, просто ее возможности никто не использовал.

И про Null Object. Раньше я думал о нем как о заглушке для тестов. Теперь вижу, что это нейтральный элемент для конкретного способа объединения, как ноль для сложения или единица для умножения. Если нейтральный элемент не находится, композит, скорее всего, тоже не получится, и это повод еще раз посмотреть на сигнатуру.

Для себя беру простое правило: прежде чем заводить интерфейс, мысленно пишу для него `NoOp` и `Composite`. Если не получается, сначала разбираюсь, что не так с сигнатурой или сколько в нем ролей, и только потом пишу реализации.
