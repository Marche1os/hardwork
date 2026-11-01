## Обобщение замыкания

Строгое замыкание `A -> A` получается не всегда.
Часто методу кроме основного значения нужен еще какой-то контекст.
Обобщение в том, что на выходе тип совпадает с одним из входных, а остальные входы только помогают его вычислить.
Null Object тогда тоже очевиден: вернуть основной вход без изменений, контекст просто проигнорировать.

### 1. Обогащение DeviceNode

Сейчас название комнаты, признак избранного и прочее подмешиваются в `DeviceNode` в разных местах, где-то в мапперах, где-то прямо во `ViewModel`.
Можно свести это к одной сигнатуре:

```kotlin
fun interface DeviceNodeEnricher {
    fun enrich(node: DeviceNode, context: HomeContext): DeviceNode

    companion object {
        val NONE = DeviceNodeEnricher { node, _ -> node }
    }
}

class CompositeEnricher(
    private val enrichers: List<DeviceNodeEnricher>,
) : DeviceNodeEnricher {
    override fun enrich(node: DeviceNode, context: HomeContext) =
        enrichers.fold(node) { acc, enricher -> enricher.enrich(acc, context) }
}
```

На вход `DeviceNode` и `HomeContext`, на выходе снова `DeviceNode`.
Каждый обогатитель отвечает за одно поле:

```kotlin
val roomEnricher = DeviceNodeEnricher { node, ctx -> node.copy(roomName = ctx.rooms[node.id]) }
val favoriteEnricher = DeviceNodeEnricher { node, ctx -> node.copy(isFavorite = node.id in ctx.favorites) }

val enricher = CompositeEnricher(listOf(roomEnricher, favoriteEnricher))
```

Если завтра понадобится, например, подмешать статус прошивки, это будет еще один элемент в списке, а не правка в трех мапперах.
`HomeContext` при этом только читается, и поэтому порядок обогатителей почти не важен, пока они пишут в разные поля.

### 2. Политики команд

Есть команды устройствам, и есть ограничения: гость не может открыть замок, ребенку нельзя выставить кондиционер на 30 градусов.
Обычно такие проверки расползаются по `if` перед отправкой команды.
Если сделать политику обобщенным замыканием на команде, ограничения начинают собираться в цепочку:

```kotlin
sealed interface DeviceCommand {
    data class SetTemperature(val deviceId: String, val celsius: Int) : DeviceCommand
    data class Unlock(val deviceId: String) : DeviceCommand
    data class Rejected(val reason: String) : DeviceCommand
}

fun interface CommandPolicy {
    fun check(command: DeviceCommand, role: UserRole): DeviceCommand

    companion object {
        val ALLOW_ALL = CommandPolicy { command, _ -> command }
    }
}

class CompositePolicy(
    private val policies: List<CommandPolicy>,
) : CommandPolicy {
    override fun check(command: DeviceCommand, role: UserRole) =
        policies.fold(command) { acc, policy ->
            if (acc is DeviceCommand.Rejected) acc else policy.check(acc, role)
        }
}
```

Главное решение здесь: отказ это тоже команда, `Rejected`.
Если бы политика бросала исключение или возвращала `null`, замыкание бы сломалось, и цепочку пришлось бы собирать вручную.
А так политика может пропустить команду, исправить ее или отклонить, и результат в любом случае остается `DeviceCommand`.

```kotlin
val guestCannotUnlock = CommandPolicy { cmd, role ->
    if (cmd is DeviceCommand.Unlock && role == UserRole.GUEST) {
        DeviceCommand.Rejected("Гость не может открыть замок")
    } else {
        cmd
    }
}

val childTemperatureLimit = CommandPolicy { cmd, role ->
    if (cmd is DeviceCommand.SetTemperature && role == UserRole.CHILD) {
        cmd.copy(celsius = cmd.celsius.coerceIn(18, 24))
    } else {
        cmd
    }
}
```

Я проверил на трех случаях. Гостю замок не открывается, команда ребенка на 30 градусов превращается в 24, а владелец проходит без изменений.

### 3. Защита навигации

Перед переходом на экран нужно проверить, залогинен ли пользователь и есть ли у него дом.
Сейчас такие проверки обычно лежат в стартовом экране и в обработчике диплинков, причем в двух местах немного по-разному.

```kotlin
fun interface RouteGuard {
    fun guard(route: Route, session: Session): Route
}

class CompositeGuard(
    private val guards: List<RouteGuard>,
) : RouteGuard {
    override fun guard(route: Route, session: Session) =
        guards.fold(route) { acc, guard -> guard.guard(acc, session) }
}

val authGuard = RouteGuard { route, s -> if (!s.isLoggedIn) Route.Login else route }
val homeGuard = RouteGuard { route, s ->
    if (route != Route.Login && !s.hasHome) Route.CreateHome else route
}
```

Защита получает маршрут и возвращает маршрут, тот же самый или перенаправленный.
Диплинк на устройство без логина ведет на `Login`, с логином, но без дома, на `CreateHome`, а если все есть, открывается само устройство.
Стартовый экран и диплинки используют один и тот же `CompositeGuard`, и расходиться им больше негде.

Здесь порядок важен, в отличие от обогатителей. `homeGuard` не должен перебивать `Login`, поэтому он смотрит на результат предыдущей защиты.
Это работает только потому, что каждая защита получает на вход результат предыдущей, то есть благодаря замыканию по `Route`.

## Составные типы

Чаще всего метод все-таки переводит один тип в другой, и замкнуть его не получается.
Тогда можно сделать компонуемым сам результат: обернуть его в тип, который умеет объединяться и у которого есть пустое значение.
Лучше всего для этого подходят перечислимые типы.

### 1. Проблемы устройства как список

Для фильтра "Проблемы" нужно понять, что не так с устройством. Первое, что приходит в голову:

```kotlin
fun interface ProblemDetector {
    fun detect(node: DeviceNode): DeviceProblem?
}
```

Композит для такой сигнатуры можно собрать только по правилу "первый ненулевой".
Но у датчика может одновременно сесть батарейка и пропасть связь, и тогда вторая проблема просто потеряется.
Если вернуть список, все встает на место:

```kotlin
fun interface ProblemDetector {
    fun detect(node: DeviceNode): List<DeviceProblem>

    companion object {
        val NONE = ProblemDetector { emptyList() }
    }
}

class CompositeDetector(
    private val detectors: List<ProblemDetector>,
) : ProblemDetector {
    override fun detect(node: DeviceNode) = detectors.flatMap { it.detect(node) }
}
```

Null Object это пустой список, композит это `flatMap`.
Детекторы пишутся независимо друг от друга:

```kotlin
val lowBattery = ProblemDetector { node ->
    if (node.battery < 15) listOf(DeviceProblem.LowBattery(node.battery)) else emptyList()
}
val offline = ProblemDetector { node ->
    if (!node.isOnline) listOf(DeviceProblem.Offline) else emptyList()
}
```

Датчик с батарейкой 7% и без связи теперь дает `[LowBattery(7), Offline]`, а не только первую проблему.
Фильтр "Проблемы" при этом выражается через детектор одной строкой: `{ detector.detect(it).isNotEmpty() }`.
Отдельно описывать, что считается проблемой, больше не нужно, источник правды один.
А сортировать по важности или показывать только первую проблему в карточке решает клиент, детекторам об этом знать не надо.

### 2. Пачка команд как свой перечислимый тип

Сценарий "Ухожу из дома" должен выключить свет и закрыть замок, а "Выключить весь свет" только выключить свет.
Каждое действие пользователя превращается в набор команд разным устройствам.
Сначала я хотел возвращать `List<AttrCommand>`, но у простого списка есть проблема: два энкодера могут прислать команду для одного и того же атрибута, и в списке окажутся дубли или даже противоречащие друг другу команды.
Поэтому сделал свой составной тип с правилом объединения:

```kotlin
data class AttrCommand(val deviceId: String, val key: AttrKey, val value: AttributeValue)

class CommandBatch private constructor(
    private val commands: Map<Pair<String, AttrKey>, AttrCommand>,
) : Iterable<AttrCommand> {

    operator fun plus(other: CommandBatch) = CommandBatch(commands + other.commands)

    override fun iterator() = commands.values.iterator()

    companion object {
        val EMPTY = CommandBatch(emptyMap())

        fun of(vararg commands: AttrCommand) =
            CommandBatch(commands.associateBy { it.deviceId to it.key })
    }
}
```

Инвариант "одна команда на атрибут устройства" живет внутри типа, а при конфликте побеждает последняя команда.
`CommandBatch` реализует `Iterable`, поэтому снаружи с ним можно работать как с обычной коллекцией: `for`, `map`, `filter`, `groupBy`.

Энкодер действий возвращает этот тип:

```kotlin
fun interface ActionEncoder {
    fun encode(action: UiAction): CommandBatch

    companion object {
        val NONE = ActionEncoder { CommandBatch.EMPTY }
    }
}

class CompositeEncoder(
    private val encoders: List<ActionEncoder>,
) : ActionEncoder {
    override fun encode(action: UiAction) =
        encoders.fold(CommandBatch.EMPTY) { acc, encoder -> acc + encoder.encode(action) }
}
```

Отправка на бэкенд группирует команды по устройству через `batch.groupBy { it.deviceId }`, и на каждое устройство уходит один запрос.

Разница между двумя примерами в том, что в первом хватило стандартного `List`, потому что проблемы просто складываются.
Во втором объединение должно соблюдать правило, и для этого понадобился свой тип.
Поддержка `Iterable` при этом сохранила все удобства коллекции.

## Выводы

Все три примера обобщенного замыкания устроены одинаково: `(A, Context) -> A`.
Обогатитель, политика и защита навигации отличаются только тем, что за `A` и что за контекст.
Поэтому композит у них один и тот же, `fold` по списку, а Null Object возвращает `A` как есть.
Получается, что обобщенное замыкание это просто замыкание по `A` с параметром, который не участвует в композиции.

Самым важным решением во всех трех случаях было не выпустить результат за пределы типа.
Для политики это `Rejected` вместо исключения, для навигации редирект вместо `null`.
Как только метод начинает возвращать что-то другое (бросать, возвращать `null`, `Boolean`), цепочка перестает собираться сама, и ее приходится менеджить снаружи.

С составными типами главный вопрос: как объединять результаты.
Если они просто складываются, хватает `List` и `flatMap`.
Если при объединении есть правило (уникальность, приоритет, последняя команда побеждает), нужен свой тип, и это правило должно жить внутри него, а не в каждом композите.
Реализация `Iterable` стоит одной строки и сохраняет весь API коллекций, так что свой тип ничего не отнимает у клиента.

Еще заметил, что `DeviceProblem?` и `List<DeviceProblem>` различаются не только удобством композиции.
Nullable вариант неявно утверждает, что проблема может быть только одна, а это неправда.
То есть составной тип точнее описывает предметную область, и композируемость здесь скорее следствие.

## Заключение

Что можно применять сразу:

- возвращать списки вместо nullable там, где результатов в принципе может быть несколько. Это детекторы, валидаторы, подсказки. Это дешево, понятно коллегам и сразу дает Null Object и композит;
- цепочки вида `(A, Context) -> A` для политик, защит навигации и обогащения моделей. Таких мест в проекте немало, и сейчас они решены через `if` в разных местах;
- делать отказ или редирект значением того же типа, а не исключением или `null`.

Что будет сложнее:

- переделывать существующие интерфейсы. У того же `DeviceStateConverter` 25 реализаций, и его разделение это отдельная задача, а не правка по ходу. Реалистично делать это постепенно, когда все равно трогаешь модуль;
- часть кода живет на колбэках Android и сторонних SDK, где сигнатуры навязаны снаружи. Там максимум, что можно сделать, это обернуть их в свой глубокий интерфейс, как с сокетом;
- читаемость для команды. `fold` по списку лямбд для многих менее очевиден, чем `when`. Поэтому элементам цепочки важно давать нормальные имена (`guestCannotUnlock`, а не `policy2`), иначе выигрыш в архитектуре съест потеря в понятности.

И главное ограничение, которое я вынес из прошлых заданий: не обобщать заранее.
Можно было бы завести общий `Refinement<A, C>` для всех трех примеров, но пока это три разных места, и общий интерфейс был бы ровно той абстракцией про запас, на которую я уже наступал с фильтрами.
Если таких цепочек станет пять-шесть и они начнут дублировать инфраструктуру, тогда будет повод.

Практически я собираюсь использовать все рекомендации как чеклист на ревью, своего кода и чужого. Для нового интерфейса задаю себе несколько вопросов:

- сколько в нем ролей;
- сколько работы он забирает у клиента;
- можно ли написать для него `NONE` и `Composite`;
- в какую категорию он попадает: команда, замыкание, обобщенное замыкание, составной результат.

Если ни в одну, это повод еще раз подумать над сигнатурой, прежде чем писать реализации.
По моим ощущениям, в Android коде чаще всего будут встречаться команды и составные результаты (`List`, `Flow`).
Обобщенное замыкание пригодится в доменной логике вроде политик и навигации.
А чистое замыкание останется для небольшого числа value-объектов вроде `AttributeState` и `LightSettings`, зато там оно дает больше всего.
