### 1

**Было**

```kotlin
object GlobalFormatters {
    var dateFormatter = SimpleDateFormat("dd/MM/yyyy")
}

fun renderUser(user: User): String {
    return GlobalFormatters.dateFormatter.format(user.birthDate)
}
```

**Стало**

```kotlin
class UserRenderer(private val formatter: DateFormat) {
    fun render(user: User): String =
        formatter.format(user.birthDate)
}
```

Один глобальный форматтер влияет на всю систему, что подразумевает хрупкость. 
Я сделал зависимость явной, удалил скрытую изменяемость.


### 2

**Было**
```kotlin
fun User.isPremium(): Boolean {
    return purchaseHistory.sumOf { it.amount } > 5000
}
```

**Стало**

```kotlin
class PremiumPolicy(private val threshold: Int) {
    fun isPremium(user: User): Boolean =
        user.purchaseHistory.sumOf { it.amount } > threshold
}
```

Extension выглядит как утилита, но содержит бизнес‑правило. Вынес в отдельную политику, бизнес‑слой стал явным

### 3

**Было**
```kotlin
fun canShowGreeting(): Boolean {
    return LocalTime.now().hour < 12
}
```

**Стало**

```kotlin
class Clock {
    fun now() = LocalTime.now()
}

class Greeter(private val clock: Clock) {
    fun canShowGreeting(): Boolean =
        clock.now().hour < 12
}
```

Когда бизнес-логика зависит от времени, прямой вызов now() делает код нетестируемым и хрупким. 
Ввел Clock как abstraction over time.

### 4

**Было**
```kotlin
fun calculateScore(user: User): Int {
    user.lastScoreCheck = System.currentTimeMillis()
    return user.posts.size * 2
}
```

**Стало**

```kotlin
fun calculateScore(posts: List<Post>): Int =
    posts.size * 2

fun updateCheckTime(user: User) {
    user.lastScoreCheck = System.currentTimeMillis()
}
```

Название `calculateScore` обещает чистоту, но мутирует объект. Разорвал побочный эффект и вычисление.

### 5

**Было**
```kotlin
class ProfileViewModel : ViewModel() {
    fun calculateReputation(posts: List<Post>): Int {
        return posts.sumOf { it.likes * 3 }
    }
}
```

**Стало**

```kotlin
class ReputationUseCase {
    fun calculate(posts: List<Post>): Int =
        posts.sumOf { it.likes * 3 }
}

class ProfileViewModel(private val reputation: ReputationUseCase) : ViewModel() {
    fun load() { /* ... */ }
}
```

ViewModel занималась вычислениями бизнес‑логики. Перенёс в use case, это сделало архитектуру чище.

### 6

**Было**
```kotlin
fun parseAge(text: String): Int {
    try {
        return text.toInt()
    } catch (e: Exception) {
        return 0
    }
}
```

**Стало**

```kotlin
fun parseAge(text: String): Int? =
    text.toIntOrNull()
```

Ловить исключения для обычных ситуаций это плохая практика и вносит запутанность. 
Использую встроенную безопасную функцию. Для более сложных кейсов использую `Either`

### Выводы

Выполняя задание выделил несколько выводов:

• Насколько часто разработчики маскируют бизнес‑логику в местах, где ее не ожидаешь - в extension‑функциях, глобальных объектах, ViewModel. Например, у меня в команде некоторые разработчики используют маппинг в domain/UI сущности в extension-функциях.

• Как много проблем исходит не от "классических" ошибок, а от неявных зависимостей - времени, форматтеров, побочных эффектов в "чистых" функциях.

• Что чрезмерные абстракции вредят не меньше, чем их отсутствие.

Также в работе теперь буду больше рефлексировать о коде и использовать интуицию. 
На самом деле это сильный подход, способный значительно улучшить качество отдельного участка кода или даже подсистемы.
Однажды я реализовал на UI Compose анимацию перехода из 1 Composable функции в другую, код получился довольно запутанным и связанным, так как при transition анимациях требуется указывать ключи, которыми объединяются анимирующиеся Composable функции.
Из ощущения несовершенства я пытался сделать понятный API и реализовал в итоге удобное DSL решение, описыващюее анимацию. А все API-required вещи оставил под капотом.