На проекте заменен генерации исключений 

### 1

```kotlin
// до
fun logAnalyticEvent(event: String) {
    if (analyticEngine == null) {
        throw IllegalStateException("Аналитика не проинициализирована")
    }
}

// после
fun initAnalyticEngine() {
    if (engine == null) {
        localLogger.d { "Аналитика не проинициализирована" }
        return
    }
}

```

### 2

```kotlin
// до

fun map(dto: HomeDto): Home {
    if (dto.id.isNullOrEmpty()) {
        throw IllegalArgumentException("id не может быть null")
    }

    if (dto.name.isNullOrEmpty()) {
        throw IllegalArgumentException("name не может быть null")
    }
    
    // маппинг полей..
}

// после

fun map(dto: HomeDto): Either<IllegalContract, Home> = either {
    ensureOrError(dto.id.isNotNullOrEmpty())
    ensureOrError(dto.name.isNotNullOrEmpty())
    
    // маппинг
}
```

### 3

```kotlin
fun onUserClick() {
    // до
    val contentState = state as? State.Content ?: throw IllegalStateException("Невалидный стейт: не может быть клика без контента")
    
    // после
    if (contentState !is State.Content) {
        logger.wtf("Внезапно оказались в невалидном стейте")
    }
}
```

### 4

```kotlin
fun getRefreshToken(): String {
    val refreshToken = storage.readRefreshToken()
    
    // до
    if (refreshToken.isNullOrEmpty()) {
        throw GetTokenException()
    }
    
    // после
    
    if (refreshToken.isNullOrEmpty()) {
        // отправляем в аналитику: неожиданно оказались в невалидном состоянии и возвращаем безопасный тип Result
        logger.wtf("Токен пуст")
        
        return Result.failure(GetTokenException())
    }
}
```

### 5

```kotlin
class UserProfile : Fragment() {
    
    fun onViewCreated(
        inflater: LayoutInflater,
    ): View {
        // до
        val argumentUserId = arguments.getString(USER_ID_KEY) ?: throw IllegalArgumentException("Экран требует передачи id пользователя")
        
        // после
        val argumentUserId = arguments.getString(USER_ID_KEY)
        if (argumentUserId.isNullOrEmpty()) {
            setState { State.Error(
                description = "Не получилось загрузить данные пользователя"
            ) }
        }
    }
}

```


### Выводы

В целом согласен с мнением, что генерация исключений сильно усложняет понимание программы через ввод дополнительного потока управления. 
Поди еще найди первый обработчик, в который попадет эксепшен.
Для клиентских приложений бросать исключение допустимо лишь в случаях безопасности. 
Например, когда в Android системе пытается обратиться по IPC приложение с невалидной подписью. Или при ошибках в работе защищенного хранилища.

Единственное, иногда, на мой взгляд, все-таки полезно используя `Preconditions` обозначить явно невалидные состояния, чтобы наоборот сузить область поведения программы.

На проекте после выполнения задания решили уйти от генерации исключений в невалидных стейтах в пользу игнора ошибки и отправки специального события в аналитику.

И не стоит забывать, что большое количество исключений могут быть заменены добавлением строгой системы типов.