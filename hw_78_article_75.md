### 1

До:
```kotlin
suspend fun filterMessages(messages: List<Message>, threshold: Long): Pair<List<Message>, List<Message>> {
    val fresh = messages.filter { it.timestamp > threshold }
    val stale = messages.filter { it.timestamp <= threshold }
    
    return fresh to stale
}
```

После:

```kotlin
data class MessageGroups(
    val freshMessages: List<Message>,
    val staleMessages: List<Message>
)

suspend fun splitMessages(messages: List<Message>, threshold: Long): MessageGroups {
    val freshMessages = mutableListOf<Message>()
    val staleMessages = mutableListOf<Message>()
    for (msg in messages) {
        if (msg.timestamp > threshold) {
            freshMessages.add(msg)
        } else {
            staleMessages.add(msg)
        }
    }
    return MessageGroups(freshMessages, staleMessages)
}
```

### 2

```kotlin
fun splitMessages(messages: List<Message>, threshold: Long): Pair<List<Message>, List<Message>> {
    val fresh = messages.filter { it.timestamp > threshold }
    val stale = messages.filter { it.timestamp <= threshold }
    return Pair(fresh, stale)  // Неявно: first = fresh, second = stale
}

val (oldMessages, newMessages) = splitMessages(messages, threshold)  
```

Надежное решение с использованием именованного класса (data class)

```kotlin
data class MessageGroups(
    val freshMessages: List<Message>,
    val staleMessages: List<Message>
)

fun splitMessages(messages: List<Message>, threshold: Long): MessageGroups {
    val freshMessages = messages.filter { it.timestamp > threshold }
    val staleMessages = messages.filter { it.timestamp <= threshold }
    return MessageGroups(freshMessages, staleMessages)
}

val groups = splitMessages(messages, threshold)
val fresh = groups.freshMessages
val stale = groups.staleMessages
```

### 3

До
```kotlin
fun selectActiveUsers(users: List<User>): List<User> {
    val active = users.filter { it.status == UserStatus.ACTIVE }
    val inactive = users.filter { it.status != UserStatus.ACTIVE } // логическая связь скрыта
    cacheInactive(inactive)
    return active
}
```

После

```kotlin
data class UserSplit(val active: List<User>, val inactive: List<User>)

fun splitUsers(users: List<User>): UserSplit {
    val active = mutableListOf<User>()
    val inactive = mutableListOf<User>()

    for (u in users) {
        if (u.status == UserStatus.ACTIVE) active += u else inactive += u
    }
    return UserSplit(active, inactive)
}

fun selectActiveUsers(users: List<User>): List<User> {
    val split = splitUsers(users)
    cacheInactive(split.inactive)
    return split.active
}
```

### 4

До
```kotlin
// Pair<List<Message>, List<Message>> - легко забыть, что где
fun splitByRead(messages: List<Message>): Pair<List<Message>, List<Message>> {
    val read = messages.filter { it.isRead }
    val unread = messages.filter { !it.isRead }
    return read to unread
}

val (firstList, secondList) = splitByRead(msgs) // неясно, какой из них "прочитанные"
```

После

```kotlin
data class ReadSplit(val read: List<Message>, val unread: List<Message>)

fun splitByRead(messages: List<Message>): ReadSplit {
    val read = mutableListOf<Message>()
    val unread = mutableListOf<Message>()

    for (msg in messages) {
        if (msg.isRead) read += msg else unread += msg
    }
    return ReadSplit(read, unread)
}

val lists = splitByRead(msgs)
showUnread(lists.unread)
```

### Выводы

В процессе выполнения задания стало ясно, что логическая неоднозначность в коде чаще всего возникает не из‑за сложности самих условий, 
а из‑за того, что код скрывает или размывает смысл. Два выделенных типа проблем проявляются особенно часто.

Работая с кортежами легко создать ловушку: порядок значений никак не документирован. 
Рефакторинг примеров подтвердил, что введение именованных data‑классов почти полностью устраняет двусмысленность. 
Это важно в местах, где результаты часто передаются между слоями (ViewModel, репозиторий, доменные слои), и ошибка в интерпретации порядка может проявиться далеко от места возникновения.

Создание примеров помогло еще раз убедиться, что большинство логических ошибок это не следствие сложной логики, 
а последствие неявных соглашений и невыраженных связей. Устранение неоднозначности это не только улучшение стиля, но и вклад в надежность Android проектах.