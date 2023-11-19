1. Есть модуль BitmapUtils, где происходит различная работа с изображениями на устройстве. Типичная ситуация, в функцию передавался путь до файла (на самом деле множество всех строк)
   и далее какая-то логика. В результате изменений удалена низкоуровневая проверка, что функция для работы с изображением получает именно путь до изображения.
   В качестве входного параметра теперь не множество всех строк, а множество путей с расширением .jpg, .png и т.д. Которое в свою очередь принимает множество путей до файлов.

Можно условно представить, что результирующая функция принимает пересечение множеств A ⋂ B, где A - множество всех путей до файла, B - множество путей до изображения.

**Было**

```kotlin
fun getBitmapByPath(path: String): Bitmap {
    if (isPathToImageValid(path)) {
        TODO("Работаем с изображением")
    }
}

fun isPathToImageValid(path: String): Boolean {
    TODO("Проверка, что путь действительно до изображения")
}
```

**Стало**

```kotlin
class ImagePath(pathToImage: Path) {
    private val imageExtensions = buildSet {
        add("png")
        add("jpg")
        add("webp")
        //...
    }

    init {
        val ext = pathToImage.extension
        require(imageExtensions.contains(ext)) { "Не поддерживаемый формат изображения: $ext" }
    }
}

fun getBitmapByPath(imagePath: ImagePath): Bitmap {
    TODO("Работаем с изображением")
}
```


2. Есть функция генерации uri для авторизации и она была сложна тем, что принимала множество параметров, которые подставлялись в query к uri.
   Неудобство заключается в том, что изменений количества/типа параметра влечет за собой изменений в нескольких местах. Кроме того, набор этих параметров передавался в нескольких функциях.
   Но самое главное это опять же отсутствие контроля за данными. Передача примитивов создает множество всех допустимых значений.
   А создания типа, характерного для технической задачи, задает значения в едином месте и гарантируют безопасность функций, работающих с этим типом, ведь
   создание юнит-теста, делающего проверку нашего нового типа в нормальных, пограничных и экстремальных условиях, позволяет сделать такую гарантию.


**Было**

```kotlin
fun generateUri(
    isNewbie: Boolean,
    token: String,
    lastSignIn: Date,
    deviceId: String,
    googleId: String,
    authId: String,
    baseUrl: String,
): Uri {
    TODO("Генерируем uri с переданными параметрами")
}
```

**Стало**

```kotlin
data class AuthorizationParams(
    @QueryId("is_newbie")
    val isNewbie: Boolean,

    @QueryId("token")
    val token: String,

    @QueryId("last_sign_in")
    val lastSignIn: Date,

    @QueryId("auth_id")
    val authId: String,

    val baseUrl: String,

    val adIds: AdIds,
)

data class AdIds(
    @QueryId("device_id")
    val deviceId: String,

    @QueryId("google_id")
    val googleId: String,
    //...
)

fun generateUri(
    authorizationParams: AuthorizationParams,
): Uri {
    TODO("Генерируем uri с переданными параметрами. Кроме того добавлены аннотации, которые формирует query параметр в формате key-value, где ключ - аннотация QueryId" +
            "Таким образом не потребуется в дальнейшем вносить изменения в функцию generateUri вовсе")
}
```

3. **Было**

На экране отображаем данные датчика. Здесь наблюдаем проблему с проверкой данных прямо на UI. Возможно, в предыдущих этапах флоу такие проверки на корректность значений
уже осуществлялись, но мы этого точно не знаем и гарантировать не можем, что на UI слой пришли проверенные данные. Так как опять работаем со множеством всех значений примитивного типа.
Например, из всего диапазона значений типа Int для нашей области валидно только 101 значение.

```kotlin
class SensorsUI() {
    private val idTextView: TextView
    private val nameTextView: TextView
    private val brightnessTextView: TextView
    
    fun show(sensor: SensorItem) {
        if (sensor.brightness >= 0 && sensor.brightness <= 100) {
            brightnessTextView.text = sensor.brightness.toString()
        }
        if (sensor.name.isNotEmpty) {
            nameTextView.text = sensor.name
        }
        //...
    }
}

data class SensorItem(
    val id: Int,
    val name: String,
    val brightness: Int,
)
```

В результате созданы типы предметной области, содержащие необходимые для нашей задачи множества значений.

**Стало**

```kotlin
class SensorsUI() {
    private val idTextView: TextView
    private val nameTextView: TextView
    private val brightnessTextView: TextView

    fun show(sensor: SensorItem) {
        brightnessTextView.text = sensor.brightness.toString()
        nameTextView.text = sensor.name
        //...
    }
}

data class SensorItem(
    val id: SensorId,
    val name: SensorName,
    val brightness: Brightness,
)

data class SensorName(val name: String) {
    init {
        TODO("Проверяем, что $name не пустое, имеет разумное ограничение на количество символов и не содержит спец-символов")
    }
}

data class SensorId(val id: Int) {
    init {
        require(id >= 0) { "Значение id лежит в недопустимом диапазоне:$id" }
    }
}

data class Brightness(val brightness: Int) {
    init {
        require(brightness in 0..100) { "Значение яркости должно лежать в диапазоне от 0 до 100 включительно" }
    }
}

```

4. Пользовательский ввод. В Android мы получаем значений из форм ввода (элемент EditText) в классе, отвечающим за UI представление.
   Очень частый сценарий, когда в проектах гоняют данные, введенные пользователем, по всей системе в сыром виде. Возможно, есть проверки корректности значений с точки зрения наших требований,
   но, как правило, это бессистемно.

Пример:

```kotlin
class UserSettingsFragment(): Fragment() {
    //...
    private val userSettingsViewModel: ViewModel //...
    
    fun setListeners() {
        firstNameEditText.setDataChanged { input ->
            if (!isFirstNameValid(input)) {
                TODO("Подсвечиваем, что текущий ввод некорректен")
            }
        }
        
        submitButton.setOnClickListener { 
            val firstName = firstNameEditText.text
            val lastName = lastNameEditText.text
            // ...
            if (isUserDataValid(firstName, lastName)) {
                userSettingsViewModel.onSubmit(firstName, lastName)
            }
        }
    }
} 

class UserSettingsViewModel(
    userSettingsRepository: UserSettingsRepository,
): ViewModel() {
    
    fun onSubmit(firstName: String, lastName: String) {
        userSettingsRepository.saveNewUserData(firstName, lastName)
    }
}

class UserSettingsRepository(remoteDataSource: UserSettingsApi, localDataSource: RoomDB) {
    fun saveNewUserData(firstName: String, lastName: String) {
        val blockingOperation = remoteDataSource.update(firstName, lastName)
        if (blockingOperation.isSuccessfull) {
            localDataSource.cache(firstName, lastName)
        }
    }
}
```

Суть в том, что мы гоняем сырые данные. Проверка корректности данных по факту гарантирует корректность только в блоке условия:
```kotlin
     if (isUserDataValid(firstName, lastName)) {
         userSettingsViewModel.onSubmit(firstName, lastName)
     }
```
Во всех остальных случаях у нас нет гарантии и максимум, что можем сделать для этого - писать тесты, причем интеграционные: требуется проверить всю цепочку от UI до отправки данных по сети.
Ну и нужно постоянно думать, работая с сырыми данными пользователя, точно ли на предыдущих этапах у нас все проверки выполнялись.

В качестве рефакторинге можно было бы сделать что-то вроде такого:

```kotlin
class UserSettingsFragment(): Fragment() {
    //...
    private val userSettingsViewModel: ViewModel //...
    
    fun setListeners() {
        firstNameEditText.setDataChanged { input ->
            if (!isFirstNameValid(input)) {
                TODO("Подсвечиваем, что текущий ввод некорректен")
            }
        }
        
        submitButton.setOnClickListener { 
            val firstName = firstNameEditText.text
            val lastName = lastNameEditText.text
            // ...
            if (isUserDataValid(firstName, lastName)) {
                userSettingsViewModel.onSubmit(firstName, lastName)
            }
        }
    }
} 

data class FirstName(val firstName: String) {
    init {
        TODO("проверка валидности значения")
    }
}

fun String.asFirstName(): FirstName = FirstName(this)
// asLastName так же

```

И дальше уже работать с типом FirstName/LastName. Но проблема в том, мы завязаны на получение данных пользователя на конкретном экране.
И если получаем такие данные из нескольких экранов, то будет дублирование. По-сути это происходит из-за того, что мы не работаем с первоисточником данных, а получаем их откуда-то еще.

И идейно хочется, чтобы мы уже получали от пользователя корректное множество значений, удовлетворяющее ограничениям предметной области.
И первоисточником будет служить компонент ввода. То есть мы запрограммируем форму ввода данных такую, что в нее в принципе нельзя добавить данные, не удовлетворяющие нашим требованиям.

```kotlin
abstract class UserDataInputForm: EditText

class FirstNameInputForm(): UserDataInputForm {
    fun onDataChanged(newData: String) {
        TODO("следим за вводом и подсвечиваем форму ввода, если она не удовлетворяет условиям") 
    }
    
    fun getCurrentValue(): FirstName {
        // Упрощенная схема. При попытке получить значение, не удовлетворяющее в данных момент требованиям, можно подсветить форму ввода красным и подсказкой, что исправить
        val currentText = editText.text
        return currentText.asFirstName()
    }
    
    data class FirstName(val firstName: String) {
        init {
            TODO("проверка валидности значения")
        }
    }

    fun String.asFirstName(): FirstName = FirstName(this)
}
```

Такая организация требует не использовать стандартные компоненты ввода, а использовать свои, специфичные для задачи. Ответственность за формат данных возложили непосредственно на первоисточник.
Для этого мы приняли за данность, что источником данных служит не некий пользователь, а непосредственно компонент ввода, содержащий данные. Пользователем может быть кто/что угодно и нас это не должно сильно волновать.
Источник данных - то, откуда данные забираем, а не то, кто или что эти данные туда положило.

Так же можно будет обернуть firstName, lastName в единый тип - UserSettingsData, например.
Ведь типов может быть много и каждый раз указывать их все при передаче из одного слоя в другой - неудобно. Кроме того, упростит написание тестов.


5. Сущность события содержит продолжительность события. В качестве улучшения перенес защиту на уровень типа [EventTimeInterval]

**Было**

```kotlin
data class Event(
    private val timeInterval: EventTimeInterval,
    // ...
)

data class EventTimeInterval(val start: Long, val end: Long)

fun createTimeInterval(start: Long, end: Long): EventTimeInterval? {
    val tenMinutesInMs = TimeUnit.MINUTES.toMillis(10)
    return if (end >= start + tenMinutesInMs) TimeInterval(start, end) else null
}
```

**Стало**
```kotlin
data class EventTimeInterval private constructor(val start: Long, val end: Long) {
    
    init {
        val tenMinutesInMs = TimeUnit.MINUTES.toMillis(10)
        require(end >= start + tenMinutesInMs) { "Продолжительность события должна быть как минимум 10 минут" }
    }
}
```


**Выводы**

Рассмотрели проблемные места в дизайне типов. Глобальная проблема заключается в том, что специализированные функции, которые заточены под работы с ограниченным множеством значений, начинают работать с множеством всех значений типа.
Что требует дополнительных проверок непосредственно перед работой с этим множеством. А по-сути такая проверка - это отсечение множества значений, не являющихся валидными для конкретной задачи.
Поэтому мы в текущем решении задания вводили свои специально заточенные под конкретные задачи типы с правилами инициализации.
По-сути создали правила, согласно которым результат пересечения множеств должен давать такое множество значений, которое удовлетворяет условиям задачи или предметной области.

Из плюсов такого решения получаем подход с централизованной проверкой валидности значений, которое должно покрываться юнит-тестом. Гарантию правильного использования типа во всей программе,
ведь, например, тип [Brightness], отвечающий за яркость, может содержать интерфейс для изменения показателя яркости. Собственно, так и должно быть.

Так как мы теперь получаем гарантию правильного использования типа данных, это накладывает определенные ограничения и правила.
Так, теперь использование рефлексии на эти типы может по умолчанию разрушить все гарантии и правила.

И простой совет "быть аккуратным с рефлексией" уже не сработает. Здесь все просто, используем рефлексию - гарантия правильности работы и использования типа слетает.
В больших проектах зачастую используется DI, иногда самописный, не типобезопасный и основанный на рефлексии. Поэтому для надежности всей системы следует отказаться от reflection api вовсе,
DI и прочие подобные инструменты должны быть типобезопасными и компилируемыми. Вероятно, это будет основанное на кодогенерации решение, потенциально замедляющее время сборки проекта.

Частый случай в программировании, снова сталкиваемся с trade-off решением, требующим индивидуального подхода и анализа проекта.     
