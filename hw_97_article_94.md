## Когда универсальности было явно маловато

### 1. Конвертеры состояний датчиков

Датчики движения, открытия, дыма и газа сделаны по одной схеме. Вот дым:

```kotlin
internal class SmokeSensorStateConverter @Inject constructor() : DeviceStateConverter<SmokeSensorState> {

    private val defaultState = SmokeSensorState(
        attributeState = AttributeState(),
        isOnline = false,
        isSmoke = false,
        lastSync = Date(0),
    )
    
    override fun convert(type: DeviceType, newState: AttributeState, oldState: SmokeSensorState?): SmokeSensorState {
        val prevState = oldState ?: defaultState

        return SmokeSensorState(
            attributeState = prevState.attributeState.replace(newState),
            isOnline = newState.getBool(AttrKeys.ONLINE) ?: prevState.isOnline,
            isSmoke = newState.getBool(AttrKeys.SMOKE_STATE) ?: prevState.isSmoke,
            lastSync = newState.getLastSync(AttrKeys.SMOKE_STATE) ?: prevState.lastSync,
        )
    }
    // ...
}
```

`GasSensorStateConverter` точно такой же, только вместо `SMOKE_STATE` и `isSmoke` там `GAS_LEAK_STATE` и `isGasLeak`. Классы по 40 с лишним строк, а отличаются ключом атрибута и именем поля.
У `OpeningSensorStateConverter` та же логика, но при обратной конвертации он почему-то не отдает `lastSync`, а дым и газ отдают. 
Намеренно так сделано или просто по-разному скопировали, по коду не понять. 
С копиями так всегда: со временем они начинают тихо расходиться.

Когда наследование оправдано: у нескольких похожих классов есть общий код, и его выносят наверх.
Можно было бы сделать базовый конвертер.
Но у датчиков отличается не поведение, а только данные: ключ и то, как собрать конкретное состояние. 
Поэтому я бы обошелся без иерархии и передал эти отличия параметрами:

```kotlin
internal class BinarySensorStateConverter<S : DeviceState>(
    private val key: AttrKey,
    private val default: S,
    private val create: (attrs: AttributeState, isOnline: Boolean, value: Boolean, lastSync: Date) -> S,
    private val read: (S) -> Pair<Boolean, Date>,
)
```

Тогда новый датчик (протечка, вибрация) описывается парой строк в DI модуле, а не очередным файлом.

### 2. Платформа в каждом событии аналитики звонков

В `SmarthomeCallAnalyticsImpl` строка

```kotlin
PARAM_PLATFORM toStringParam "android"
```

повторяется 12 раз, по разу в каждом методе.
В старой копии этого модуля, которая еще лежит в `src/`, этого параметра нет вообще, то есть кто-то прошел по всем методам и вписал его в каждый руками. 
В технических событиях (`logTechEvent`) платформы нет. Может, так и задумано, а может, их просто пропустили. Опять же по коду этого не видно.

Параметр общий для всего модуля, и ему место в одном приватном методе:

```kotlin
private fun logCallEvent(event: SmarthomeCallEvent, vararg params: AnalyticsParam) {
    analytics.logEvent(event, PARAM_PLATFORM toStringParam PLATFORM_ANDROID, *params)
}
```

Если пойти дальше, можно сделать декоратор над `CompanionAnalytics`, который добавляет платформу ко всем событиям. 
Это маленький компонент с одной обязанностью, и его можно подключить в любом модуле, тогда никому больше не придется об этом помнить.

### 3. TreeChanges и три одинаковые ветки 

`TreeChanges` я написал в 2024 году: sealed класс с тремя наследниками, у каждого есть `unionNode`. А в 2026, когда делал фильтры дома, получил вот это:

```kotlin
when (treeChanges) {
    is TreeChanges.DeviceChange -> treeChanges.copy(unionNode = update)
    is TreeChanges.HomeChanged -> treeChanges.copy(unionNode = update)
    is TreeChanges.UnionChange -> treeChanges.copy(unionNode = update)
}
```

Ветки одинаковые. Базовый класс знает, что дом есть у всех наследников, но заменить его не умеет. 
Не хватает одного абстрактного метода:

```kotlin
sealed class TreeChanges(open val unionNode: UnionNode) {
    abstract fun withUnionNode(node: UnionNode): TreeChanges
}
```

Каждый наследник реализует его через свой `copy`, и у клиента остается `map { it.withUnionNode(update) }`. 
Честно говоря, когда я писал `when`, я понимал, что метода не хватает.
Но `TreeChanges` лежит в соседнем api модуле, и лезть туда в задаче про фильтры не захотелось.

## Когда универсальности было явно многовато

### 1. AttributeStateDslBuilder

Большой файл почти на 1к строк, в нем под 2 сотни функциий, и почти все выглядят так:

```kotlin
fun co2(value: Int) {
    attributeValues.add(ReadOnlyAttributes.co2(value))
}
```

То есть DSL повторяет каждую фабрику из `ReadOnlyAttributes` и `ReadWriteAttributes`. 
При этом в том же классе есть `attribute(value: AttributeValue)`, которой достаточно для любого атрибута, а у `Device` есть `applyState(vararg attributeValue)`. 
Я поискал вызовы: в продовом коде никто не зовет ни `applyState { }`, ни `attributeState { }`, 
DSL используется только в тестах мапперов датчиков.

Это почти дословно сценарий из статьи: огромный API, который задумывали для всех случаев, и которым в итоге никто не пользуется. 
Зато цена у него вполне реальная. Чтобы добавить новый атрибут, нужно дописать ключ в `AttrKeys`, фабрику в `ReadOnlyAttributes` и функцию в DSL, то есть три места связаны между собой, хотя могли бы не знать друг о друге. 
Порядок в самом файле уже поплыл: в секции `// Read only` лежит `antiFrostConfig`, который берется из `ReadWriteAttributes`.

Я бы оставил `attribute()` и vararg вариант, а 180 функций обертки удалил. 
Тестам хватит `attributeState(ReadOnlyAttributes.co2(800), ReadOnlyAttributes.online(true))`.

### 2. Контракт DeviceStateConverter

```kotlin
interface DeviceStateConverter<S : DeviceState> {
    fun convert(type: DeviceType, newState: AttributeState, oldState: S? = null): S
    fun convert(state: S, withReadOnly: Boolean = false): AttributeState
}
```

Интерфейс сделали максимально общим: любой конвертер обязан работать в обе стороны и принимать тип устройства. 
Реализаций 25, и вот что с ними получается:

* `type` используют только `BoomHubStateConverter` и `StarosGeneralStateConverter`, и оба через приведение `type as StarosDeviceType`. Общая сигнатура потеряла информацию о типе, и ее приходится возвращать небезопасным кастом.
* Датчики ничего не отправляют на бэкенд, но обязаны реализовать обратную конвертацию. У них она при `withReadOnly = false` возвращает пустой `AttributeState`. Этот код существует только потому, что его требует интерфейс.

Получается, что один интерфейс отвечает сразу за две разные задачи, разбор ответа бэкенда и подготовку команды. Лучше было бы разделить его на `StateParser<S>` и `StateSerializer<S>`: датчики реализуют только первый, управляемые устройства оба.
А тип для Staros конвертеров можно передать через конструктор, фабрика и так знает `DeviceBean`.

### 3. FilterPredicate

```kotlin
internal interface FilterPredicate : Predicate<DeviceNode>

internal object NoneFilter : FilterPredicate { ... }
internal object LightFilter : FilterPredicate { ... }
internal object ProblemsFilter : FilterPredicate { ... }
internal object ClimateFilter : FilterPredicate { ... }
```

Четыре фильтра, все internal, выбираются одним `when` по enum `Filter`. 
Свой интерфейс поверх `Predicate` из android.core я сделал просто на всякий случай. 
Методы `and`, `or`, `negate`, которые дает `Predicate`, нигде не используются, а в доменный слой приехала зависимость от androidx. 
То есть лишняя общность еще и ухудшила переносимость: этот код уже не вынести в чистый Kotlin модуль без переделки. Хватило бы обычного `(DeviceNode) -> Boolean`. 
Ущерб небольшой, но это та самая абстракция про запас, которой ни один клиент не воспользовался.

## Выводы

Неприятно было обнаружить в списке собственный код, причем из обеих категорий сразу. В одном месте я не довел обобщение до конца (`TreeChanges`), в другом добавил лишнее (`FilterPredicate`). Причины тоже разные. В первом случае мне было лень выходить за рамки задачи и трогать соседний модуль.
Во втором сработала привычка сначала заводить интерфейс, раз уж так принято.


При этом чужие примеры с недостатком универсальности появились явно не от лени.
Каждый новый конвертер датчика копировали с ближайшего соседа, и в момент написания это выглядело самым быстрым и безопасным вариантом.
Каждая копия по отдельности нормальная, проблема видна, только когда смотришь на все вместе.
Поэтому по ходу работы такое не ловится, нужно отдельно останавливаться и смотреть на модуль целиком.

Если универсальности не хватает, это видно по таким признакам:

- несколько классов отличаются константой или именем поля;
- одна и та же строка повторяется в каждом методе;
-`when` с одинаковыми ветками по наследникам sealed класса.

А если ее слишком много, признаки другие:

-параметр в общей сигнатуре нужен двум реализациям из 25;
-реализации возвращают пустой результат или делают приведение типа, чтобы соответствовать контракту;
-у публичного API ноль вызовов в продовом коде;
-компонент тянет за собой зависимости, которые нужны не ему, а его "универсальности".

По сути обе проблемы про одно и то же: точка вариативности стоит не там. 
У датчиков меняется только ключ атрибута, но этот ключ так и не вынесли в параметр, и каждый датчик получил отдельный класс.
У `DeviceStateConverter` все наоборот: интерфейс заранее предусматривает изменчивость во всем, хотя реально почти ничего не меняется.

В статье есть мысль, что интерфейс сам по себе еще не абстракция. На своих примерах я это хорошо увидел. 
`FilterPredicate` формально интерфейс, но никакой роли он не описывает, это просто обертка. 
А `DeviceStateConverter` описывает сразу две роли, и поэтому реализациям в нем тесно. 
Хорошая абстракция совпадает с тем, что реально различается между случаями, и ее нужно выводить из уже существующих конкретных классов, а не из предположений о том, что может понадобиться потом.
