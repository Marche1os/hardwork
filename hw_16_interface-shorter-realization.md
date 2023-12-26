**Примеры призрачного состояния.** 

В Android проекте объявлена глобальная переменная, отвечающая за позицию Switcher, это UI элемент. 
У UI фреймворка есть тип mutableState, при изменении значения переменной все UI элементы, получающие значение из этой переменной, автоматически обновятся.

Проблема заключается в том, что переменная `isChekedAll` объявлена в 1 файле, а изменяется в другом. Так как в правильной архитектуре UI подписывается на изменения в ViewModel и соответствующим образом отрисовывается, то переменная так же должна быть размещена во ViewModel.

Такая ошибка была допущена ввиду того, что это первый реализованный экран на новом UI стеке.  

Выдержка кода:
```kotlin
var isCheckedAll = mutableStateOf(false)

@Composable
class DeviceSelectorUI() {
    checkAllSelector.value = isCheckedAll
}

class DeviceSelectorViewModel() {
    fun onSelectDevice(device: Device) {
        device.isSelect = !device.isSelect
        isCheckedAll = areAllDevicesSelected()
    }
}
```

Второй пример расположен в компоненте, который мне предстоит рефакторить. Есть локальная переменная `uuid`, которая генерируется при открытии веб-вью для авторизации.
И есть функция getToken, которая вызывается по deeplink, соответственно нет контроля за состоянием uuid. Это проблема в архитектуре, причем на стыке интеграции нескольких сервисов, требуется совместно выработать решение таких архитектурных проблем и неясности.


Выдержка кода:
```kotlin
var uuid: String = ""

fun generateUuid(): String {
    uuid = UUID.randomUUID().toString()
    return uuid
}

fun openWebView() {
    val uuid = generateUuid()
    // открытие вебвью с переданным [uuid]
}

fun getToken() {
    analytics.sendUuid(uuid)
}
```

**Примеры погрешностей/неточности**

До рефакторинга:

```kotlin
suspend fun <T> retry(block: suspend () -> Unit): T {
    val maxDelayMs = 4000L
    val factor = 2.0

    var sumDelayMs = 0L
    var currentDelayMs = 100L
    while (sumDelayMs < maxDelayMs) {
        runCatching {
            return block()
        }
        delay(currentDelayMs)
        sumDelayMs += currentDelayMs
        currentDelayMs = (currentDelayMs * factor).toLong().coerceAtMost(maxDelayMs)
    }
    return block()
}
```

После:

```kotlin
// Новая спецификация позволяет управлять политикой ретрая через параметры
suspend fun <T> retry(
    initialDelayMs: Long = 100, 
    maxDelayMs: Long = 1000, 
    factor: Double = 2.0,
    block: suspend () -> T,
): T {
    var sumDelay = 0L
    var currentDelay = initialDelayMs
    while (sumDelay < maxDelayMs) {
        runCatching {
            return block()
        }
        delay(currentDelay)
        sumDelay += currentDelay
        currentDelay = (currentDelay * factor).toLong().coerceAtMost(maxDelayMs)
    }
    return block()
}
```

До рефакторинга:
```kotlin
@Composable
fun PrimaryButton(text: String, onClick: () -> Unit) {
    Button(
        onClick = onClick,
        colors = ButtonDefaults.buttonColors(backgroundColor = Color.Blue)
    ) {
        Text(text = text, color = Color.White)
    }
}
// Кнопка имеет фиксированный цвет, который нельзя изменить без изменения самой функции
```

После:

```kotlin
Composable
fun PrimaryButton(
    text: String, 
    onClick: () -> Unit, 
    backgroundColor: Color = Color.Blue, 
    contentColor: Color = Color.White
) {
    Button(
        onClick = onClick,
        colors = ButtonDefaults.buttonColors(backgroundColor = backgroundColor)
    ) {
        Text(text = text, color = contentColor)
    }
}
// Теперь можно задать backgroundColor и contentColor при вызове функции, что делает её более универсальной и легко выносимой в библиотеку UI компонентов
```

До рефакторинга:


```kotlin
fun Context.loadImage(imageUrl: String, imageView: ImageView) {
    Glide.with(this)
        .load(imageUrl)
        .centerCrop()
        .into(imageView)
}
// Функция загружает изображение из URL и применяет жёстко закодированную обрезку
```

После:

```kotlin
fun Context.loadImage(
    imageUrl: String, 
    imageView: ImageView, 
    placeholder: Drawable? = null,
    errorPlaceholder: Drawable? = null,
    requestOption: RequestOptions = RequestOptions().centerCrop()
) {
    val glideRequest = Glide.with(this)
        .load(imageUrl)
        .apply(requestOption)
        
    if (placeholder != null) {
        glideRequest.placeholder(placeholder)
    }

    if (errorPlaceholder != null) {
        glideRequest.error(errorPlaceholder)
    }

    glideRequest.into(imageView)
}
// Функция теперь позволяет задать placeholder, errorPlaceholder и опции запроса, делая её более гибкой для использования
```

**Интерфейс не должен быть проще реализации**

- Работа с криптографией. Недавно была задача по хранению ключей в криптостойком хранилище в изолированной системе на Android. Все методы API по шифрованию, генерации ключей справедливо содержат большое количество параметров. Конкретно здесь
- Библиотеки для создания и обработки мультимедиа: Работа с мультимедийными данными часто требует точного управления форматами, кодеками и т.д. Операции могут быть сложными, и упрощение интерфейса может ограничить возможности обработки. 
- API для работы графикой, например, OpenGL. Почти все методы API для работы с OpenGL достаточно сложны, содержат и массив, который заполняется ошибками при их возникновении, вариантов этих ошибок может быть с 10-20.

Практически всегда можно обернуть сложный API в собственный API, создав ориентированную на предметную область архитектуру и более высокоуровневые конструкции и структуры данных. Тот же DSL позволит создать более выразительные и читабельные структуры.


**Выводы**

Как раз в последнее время занимался рефакторингом модуля, пересмотром функций, анализом удобства использования. Так, исправление ограниченности функций позволило некоторые из них вынести в utils модуль, удалить дублирование подобных задач в других модулях и сделать централизованное решение.
Такие задачи улучшают видение функций и помогают находить места для пересмотра их сигнатуры, сделать функции более масштабируемыми, не увеличивая их зону ответственности.

Попытка упростить API, особенно в сложных областях, как криптография, показывает, что перегруженность функций параметрами иногда неизбежна для достижения высокой степени контроля и безопасности. Обратной стороной было бы раздувание функций с длинными названиями. Или же создание переусложненной структуры, как вариант.

По призрачному состоянию, как один из способов предотвращение таких проблем - написание тестов. Если требуется для теста создавать множество моков, создавать "окружение", то это триггер, что, возможно, есть проблемы и стоит пересмотреть участок кода. 
