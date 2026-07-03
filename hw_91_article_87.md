### 1

```kotlin
// До
fun requestOnboardingContent(suspend request: () -> Out<OnboardingResponse>) {
    ioCoroutineScope.launch(globalErrorHandler) {
        // Либо возвращает результат, либо выбрасывает исключение.
        // При исключении код попадет в обработчик [globalErrorHandler]
        val response = request().getOrThrow()
    }
}

// После

fun requestOnboardingContent(suspend request: () -> Out<OnboardingResponse>) {
    ioCoroutineScope.launch(globalErrorHandler) {
        // вместо размазывания потока выполнения (а-ля goto) явно обрабатываем результат IO операции ниже
        val response = request()
            .onSuccess { response -> 
                // Обработка успешного ответа
            }
            .onError { throwable -> 
                // Явная обработка ошибки
            }
    }
}

```

### 2

```kotlin

// до
@Composable
fun MessagesUI(messages: List<Message>) {
    if (messages.isEmpty()) {
        throw IllegalArgumentException("Список сообщений не должен быть пуст. Для такого случая нужно вызывать UI для пустого экрана")
    }
    
    // UI сообщений...
}

// после
@Composable
fun MessagesUI(messages: NonEmptyList<Message>) {
    // Используем специальный непустой список из Arrow. 
    // Теперь система типов будет подталкивать к правильной доменной логике (показ пустого экрана, передача конкретных сообщений).
    // Здесь логика показа пустого экрана отсутствует в целях следования декларативной парадигмы в Compose 

    // UI сообщений...
}
```

### 3

```kotlin
// До
class EventDtoToDomainMapper {
    
    // маппинг в доменную модель события с валидацией полей
    fun map(dto: EventDto): EventDomain {
        // Валидируем наличие обязательных полей: id
        requireNotNull(dto.id)
        
    }
}

// После
class EventDtoToDomainMapper {

    fun map(dto: EventDto): Either<EventParsingError, EventDomain> = either {
        // внутри either DSL указываем, что id должен быть обязателен. 
        // Если условие не выполняется, то вместо исключения получим доменный тип ошибки
        ensure(dto.id.isNotNullAndNotEmpty())

        // дальнейший маппинг
    }
}
```

### 4

```kotlin
// до
fun onContentScreenShow() {
    // делаем небезопасный cast, подразумевая, что не может отобразиться контент
    val currentState = state as State.Content
    
    analytics.screenOpened(currentState.name)
}

// после
fun onContentScreenShow(stateContent: State.Content) {
    // вместо небезопасного каста (который еще и накладные расходы имеет) явно передаем правильный тип, по которому строится аналитика
    
    analytics.screenOpened(stateContent.name)
}
```

### 5

```kotlin
// до
fun drawGradient(gradientColors: List<Color>) {
    if (gradientColors.size < 2) {
        throw Exception("цветов градиента не должно быть меньше 2")
    }
    
    // отрисываваем градиент
}

// после
fun drawGradient(gradientColors: List<Color>) {
    // Для безопасности добавляем прямо в UI дефолтные цвета: если список пуст - считаем, что цвета нет
    // если элемент один, то считаем, что градиент не нужен - нужен solid цвет
    // в противном случае используем цвета градиента
    val colors = when {
        gradientColors.isEmpty() -> listOf(Color.TRANSPARENT, Color.TRANSPARENT)
        gradientColor.size == 1 -> listOf(gradientColors.first(), gradientColors.first())
        else -> gradientColors
    }

    // отрисываваем градиент
}

```

### Выводы 

В коде рабочего проекта сейчас довольно много выбрасываний исключений, которые происходят из веры, что в это состояние программа не перейдет.
Но даже если это и так, то в любом случае вместо того, чтобы думать о данных/домене, мы думаем о состоянии программы в данный момент.
И в то, что это пути исполнения не поменяются. Избавление от исключений и снижение когнитивной нагрузки - это большой плюс системы типов и типов, в которые уже заложен инвариант.

В некоторых частях рабочего проекта, куда еще не пришла строгая система типов, я все-же предпочитаю использовать контрактный подход: явно указывать инвариант, следующий за предусловием/спецификацией функции.
Как минимум это дисциплинирует разработчиков: когда ты понимаешь, что есть явный риск исключения, ты и тест напишешь, и себя еще проверишь. :)

