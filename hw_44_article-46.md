### 1

Стало

```kotlin

@JvmInline
value class Latitude(val value: Float) {
    
    init {
        require(value >= -90f && value <= 90f)
    }
}

@JvmInline
value class Longiude(val value: Float) {
    
    init {
        require(value >= -180f && value <= 180f)
    }
}

data class Location(
    val longitude: Longiude,
    val latitude: Latitude,
)
```

**Как было**

Работая над фичей, изначально сделал явные типы данных с указанием границы допустимых значений.
Но в других модулях встречалось задание координат примитивными типами. Частая ошибка, когда их координаты путают местами. С пользовательскими типами данных такой класс ошибок исключен.

### 2

**Было:**

Код для смены фона дома. Для операций с домом требуется идентификатор текущего дома, который кешируется в локальное хранилище
Локальное хранилище может, конечно, вернуть null. Но в соответствии с архитектурой мы точно знаем, что null не получится в этом классе.
Например, потому что для этот класс создается с экрана, на который осуществлен переход по идентификатору текущего дома.
И этот идентификатор есть во множестве мест и везде требуется обрабатывать нулабельность, как ниже

```kotlin
class HomeInteractor(
    private val localStorage: LocalStorage,
    private val remoteStorage: RemoteStorage,
) {
    
    private val currentHomeId: String?
        get() = localStorage.currentHomeId
    
    fun changeHomeBackground(newBackgroundUrl: Url) {
        if (currentHomeId.isNullOrBlank()) {
            return
        }
        
        remoteStorage.updateHomeBackground(newBackgroundUrl)
    }
}
```

**Стало**

В качестве быстрого решения мы можем использовать `Preconditions` и верифицировать наши утверждения, что значение идентификатора соответствует ожидаемым.
Не лучший способ избавления от if, но в определенных ситуациях имеет место быть. Здесь используется подход, когда мы изменяем код так, чтобы ему не нужны были условные проверки, потому что мы знаем, что эти проверки уже выполнены.

```kotlin
class HomeInteractor(
    private val localStorage: LocalStorage,
    private val remoteStorage: RemoteStorage,
) {
    
    private val currentHomeId: String?
        get() = checkNotNull(localStorage.currentHomeId)
    
    fun changeHomeBackground(newBackgroundUrl: Url) {
        if (currentHomeId.isNullOrBlank()) {
            return
        }
        
        remoteStorage.updateHomeBackground(newBackgroundUrl)
    }
}
```

### 3

Использование алгебраических типов данных

**Было**

```kotlin
fun Content() {
    val isEnabled = false // какая-то логика..
    Card(isEnabled)
}

@Composable
fun Card(
    isEnabled: Boolean,
) {
    Box(modifier = Modifier.size(
        if (isEnabled) {
            width = 112.dp,
            height = 64.dp
        } else {
            width = 64.dp,
            height = 48.dp,
        }
    )) {
        // content..
    }
}
```

**Стало**

На уровне UI модели ушли от завязки на условные конструкции, теперь один 1:1 реализуют свою спецификацию. 
А вызывающий код может, например, через полиморфизм или алгебраические типы данных определять, какую функцию вызвать, чтобы не городить проверок. Особенно если они небинарные.

```kotlin
fun Content() {
    when (isEnabled) {
        true -> MediumCard()
        false -> SmallCard()
    }
}

@Composable
fun MediumCard() {
    Box(modifier = Modifier.size(
        width = 112.dp,
        height = 64.dp
    )) {
        // content
    }
}

@Composable
fun SmallCard() {
    Box(modifier = Modifier.size(
        width = 64.dp,
        height = 48.dp
    )) {
        // content
    }
}
```

### 4

**Было**

```kotlin
@Composable
fun ScreenChooser(screenType: String) {
    if (screenType == "home") {
        HomeScreen()
    } else if (screenType == "settings") {
        SettingsScreen()
    } else if (screenType == "profile") {
        ProfileScreen()
    } else {
        NotFoundScreen()
    }
}

@Composable
fun HomeScreen() {
    Text("Home Screen")
}

@Composable
fun SettingsScreen() {
    Text("Settings Screen")
}

@Composable
fun ProfileScreen() {
    Text("Profile Screen")
}

@Composable
fun NotFoundScreen() {
    Text("Not Found")
}
```

Стало:

```kotlin
sealed class Screen {
    object Home : Screen()
    object Settings : Screen()
    object Profile : Screen()
    object NotFound : Screen()
}

@Composable
fun ScreenChooser(screen: Screen) {
    when (screen) {
        is Screen.Home -> HomeScreen()
        is Screen.Settings -> SettingsScreen()
        is Screen.Profile -> ProfileScreen()
        is Screen.NotFound -> NotFoundScreen()
    }
}

@Composable
fun HomeScreen() {
    Text("Home Screen")
}

@Composable
fun SettingsScreen() {
    Text("Settings Screen")
}

@Composable
fun ProfileScreen() {
    Text("Profile Screen")
}

@Composable
fun NotFoundScreen() {
    Text("Not Found")
}

@Composable
fun MainScreen() {
    val currentScreen = Screen.Home
    ScreenChooser(currentScreen)
}
```

1. Создали sealed class Screen, чтобы представлять различные экраны. Это позволяет нам четко понимать, какие экраны могут существовать, и улучшает проверку типов.

2. С помощью when мы избавились от множества if и улучшили структуру кода, сделав его более читаемым и поддерживаемым.

3. Если в будущем потребуется добавить новый экран, можно просто создать новый объект в с sealed class, а также добавить новый случай в when, не усложняя логику.


### 5

**Было**

```kotlin
fun getDeviceStatus(device: String): String {
    return if (device == "Light") {
        "Лампа включена"
    } else if (device == "Thermostat") {
        "Термостат установлен на 22 градуса"
    } else {
        "Неизвестное устройство"
    }
}
```

**Стало**

```kotlin
sealed class Device {
    abstract fun getStatus(): String

    object Light : Device() {
        override fun getStatus(): String = "Лампа включена"
    }

    object Thermostat : Device() {
        override fun getStatus(): String = "Термостат установлен на 22 градуса"
    }

    object Unknown : Device() {
        override fun getStatus(): String = "Неизвестное устройство"
    }
}

fun getDeviceStatus(device: Device): String {
    return device.getStatus()
}
```

- Вместо передачи строкового идентификатора устройства, мы можем использовать уже известный тип устройства, например, используя sealed классы, как показано ранее.

- И вместо проверки типа устройства, мы можем использовать объектную модель, где для каждого устройства определен свой статус через методы или свойства.


**Выводы**

Тема условных конструкций мне близка, потому что я сам стремлюсь к минимизации количества условных конструкций. Потому что занимаясь на занятиях, я уже строго рассуждаю о программе, анализирую в каких состояниях может находиться код.
И условные конструкции создают сложности в размышлении о программе, ибо с такими конструкциями нельзя однозначно и непротиворечиво делать выводы. Поэтому в своей работе стараюсь и сам не перебарщивать с if-ми, и на код-ревью, встречая условные конструкции, думаю, как можно обойтись без них.

Кроме этого, если я использую if, то выношу логику в виде документирования с пояснениями. Такой подход помогает еще и отслеживать баги (когда код не работает по спецификации).

Порой встречаю на пулл реквестах ужасные составные конструкции из условий, где 2-3 уровня вложенности с блоками else для каждой. Даже не понимаю, почему не возникает ествественного желания немного хотя бы улучшить код. Поэтому, пройдя текущее задание, буду думать над докладом на команду, чтобы поделиться, видимо, сакральными знаниями об условных операторах и почему с ними нужно бороться.