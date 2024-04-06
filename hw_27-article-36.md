### 1. Фича Разлогин

**Было**

Процесс разлогина в мобильном приложении. Есть 2 класса-перехватчика сетевых запросов. AuthorizationInterceptor и LogoutInterceptor.
- `AuthorizationInterceptor` добавляет в каждый сетевой запрос авторизационный токен. Если сервер возвращает 401-й код, то пытается обновить токен, в случае неудачи - возвращает ответ с 401 кодом ошибки.
- `LogoutInterceptor` перехватывает 401-е коды ошибок с сервера, что означает, что обновить токен не удалось, и посылает в глобальную очередь событий событие, что сессия протухла.

Слушатель глобальной очереди сообщений видит событие протухания сессии и посылает специальный in-app Deeplink, который обрабатывается обработчиком, и уже обработчик закрывает основное приложение и показывает экран с необходимостью перелогиниться.

В коде это выглядит примерно так:
```kotlin
class AuthorizationInterceptor(val tokenProvider: TokenProviderStorage) {
    override fun intercept(chain: Interceptor.Chain): Response {
        // Перед запросом добавляем заголовок с токеном
        val request = chain.request()
        val response = chain.proceed(request)
        if (response.code == UNAUTHORIZED_CODE) {
            withRetry(times = 3) { refreshToken() }
            if (tokenProvider.authToken.isNullOrEmpty()) {
                // Возвращаем ответ, который с ошибкой 401
                return response
            }
     
            // Повторяем запрос с уже новым токеном
            val newRequest = chain.request()
            return chain.proceed(newRequest)
        }
        return response
    }

    private fun refreshToken() {
        // Обновление токена
    }

    companion object {
        private const val UNAUTHORIZED_CODE = 401
    }
}

class LogoutInterceptor(val eventBus: EventBus) {
    fun proceed(chain: Interceptor.Chain): Response {
        // Перед запросом добавляем заголовок с токеном
        val request = chain.request()
        val response = chain.proceed(request)
        if (response.code == UNAUTHORIZED_CODE) {
            eventBus.sendSessionExpired()
            // Дальше подписчик получает событие и шлет deeplink с параметром, что сессия протухла
        }
        return response
    }
}

class LogoutDeeplinkHandler() {
    fun process(uri: Uri) {
        if (uri.isSessionExpiredDeeplink) {
            // Показываем экран перелогина
        }
    }
}
```

**Стало:**

```kotlin
class AuthorizationInterceptor (private val sessionManager: SessionManager, private val sharedViewModel: SharedViewModel) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val request = chain.request()
        val response = chain.proceed(request)
        
        // Код выполнения запроса и попытки обновления токена...
        // Если сессия протухла:
        handleSessionExpired()
        return response
    }

    private fun handleSessionExpired() {
        // Уведомляем SessionManager, что сессия протухла
        sessionManager.notifySessionExpired()

        // Уведомление ViewModel, которое заставляет UI немедленно реагировать
        sharedViewModel.sessionExpiredEvent.postValue(Unit)
    }
}
```

В исходной версии была "разбросанность" частей кода одной фичи по разным местам в проекте, в разных модулях. В улучшенной версии вся логика в одном модуле и в 2-х классах, который являются частью модели Отправитель/Подписчик. 
Количество участвующих в операции разлогина классов сведена по-сути к 2-м, плюс UI. 


### 2. Навигация с условием

**Было:**

```kotlin
class HomeFragment : Fragment() {

    fun navigateBasedOnUserStatus(userType: String, isSubscribed: Boolean) {
        when {
            userType == TYPE_ADMIN -> { /* Направляем пользователя в Админ панель */
                findNavController().navigate(R.id.action_homeFragment_to_adminPanelFragment)
            }
            userType == TYPE_USER && isSubscribed -> { /* Направляем подписчиков в Premium Content */
                findNavController().navigate(R.id.action_homeFragment_to_premiumContentFragment)
            }
            else -> { /* Для всех остальных направляем в Free Content */
                findNavController().navigate(R.id.action_homeFragment_to_freeContentFragment)
            }
        }
    }
    
    companion object {
        private const val TYPE_ADMIN = "admin"
        private const val TYPE_USER = "user" 
    }
}
```

Пример усложненного кода, параметры функции связаны. 
Если условия для навигации станут более комплексными или их количество увеличится, поддерживать такой подход станет сложнее, плюс увеличится количество параметров.

**Стало:**
Воспользуемся шаблонами "Стратегия" и "Фабрика":

```kotlin
interface NavigationStrategy {
    fun navigate(navController: NavController)
}

class AdminNavigationStrategy : NavigationStrategy {
    override fun navigate(navController: NavController) {
        navController.navigate(R.id.action_homeFragment_to_adminPanelFragment)
    }
}

class PremiumContentNavigationStrategy : NavigationStrategy {
    override fun navigate(navController: NavController) {
        navController.navigate(R.id.action_homeFragment_to_premiumContentFragment)
    }
}

class FreeContentNavigationStrategy : NavigationStrategy {
    override fun navigate(navController: NavController) {
        navController.navigate(R.id.action_homeFragment_to_freeContentFragment)
    }
}

class NavigationStrategyFactory {

    fun getNavigationStrategy(user: User): NavigationStrategy {
        return when {
            user.isAdmin() -> AdminNavigationStrategy()
            user.isPremiumUser() -> PremiumContentNavigationStrategy()
            else -> FreeContentNavigationStrategy()
        }
    }
    
    fun User.isAdmin(): Boolean = type == TYPE_ADMIN
    fun User.isPremiumUser(): Boolean = type == TYPE_USER && isSubscribed

    companion object {
        private const val TYPE_ADMIN = "admin"
        private const val TYPE_USER = "user"
    }
}

class HomeFragment : Fragment() {

    private val navigationStrategyFactory = NavigationStrategyFactory()

    fun navigateBasedOnUser(user: User) {
        val strategy = navigationStrategyFactory.getNavigationStrategy(user)
        strategy.navigate(findNavController())
    }
}

data class User(val type: String, val isSubscribed: Boolean)

```

Таким образом, код, отвечающий за выбор конкретного пути навигации, становится более чистым и модульным. Это упрощает добавление новых типов пользователей или изменение условий для существующих без прямого изменения кода, отвечающего за навигацию.

Этот подход позволяет сделать систему навигации в приложении максимально гибкой и адаптируемой к изменениям, при этом поддерживая порядок и чистоту архитектуры приложения.

### 3. Фича Контроллер по управлению устройствами умного дома

**Было:**

```kotlin
class SmartHomeController {
    fun turnOnLight() {
        println("Свет включен")
        // Логика включения света
    }

    fun turnOffLight() {
        println("Свет выключен")
        // Логика выключения света
    }

    fun turnOnAirConditioner() {
        println("Кондиционер включен")
        // Логика включения кондиционера
    }

    fun turnOffAirConditioner() {
        println("Кондиционер выключен")
        // Логика выключения кондиционера
    }
}
```

В этом примере каждое устройство управляется отдельными функциями для включения и выключения, что приводит к повторению кода. 
Если добавить еще устройства, класс быстро разрастется. Потенциально тако SmartHomeController превратится в God-object.

Для рефакторинга применим принцип Command для уменьшения дублирования и улучшения расширяемости кода.

- Определим интерфейс Command с единственным методом execute()
- Создадим конкретные реализации этого интерфейса для каждого действия
- В SmartHomeController заменим детальные методы на универсальный механизм выполнения команд

**Стало:**

```kotlin
interface Command {
    fun execute()
}

class TurnOnLightCommand : Command {
    override fun execute() = println("Свет включен")
}

class TurnOffLightCommand : Command {
    override fun execute() = println("Свет выключен")
}

class TurnOnAirConditionerCommand : Command {
    override fun execute() = println("Кондиционер включен")
}

class TurnOffAirConditionerCommand : Command {
    override fun execute() = println("Кондиционер выключен")
}

class SmartHomeControl {
    fun execute(command: Command) {
        command.execute()
    }
}
```

**Что стало лучше:**
- Уменьшение дублирования кода: Мы удалили повторяющиеся методы включения/выключения для каждого устройства, заменив их на обобщенную структуру команд
- Улучшение расширяемости: Добавление нового устройства теперь просто потребует создания новой команды, без необходимости изменения SmartHomeController
- Гибкость: Мы можем легко добавлять новые команды или изменять существующие без влияния на остальную часть системы управления умным домом
- Слабая связанность: Сам SmartHomeControl теперь не зависит от конкретных реализаций устройств


### 4. Фича Обработка файлов

**Было:**

Возьмем некоторую функцию, обрабатывающую запросы пользователя и принимающую nullable параметры:

```kotlin
// Функция обработки запроса пользователя. В зависимости от того, какие параметры равны null, логика работы различается.
fun processRequest(userId: Int, fileInfo: String?, permissionsInfo: String?) {
    if (fileInfo != null && permissionsInfo == null) {
        println("Запрос на обработку файла для пользователя $userId с именем файла $fileInfo")
        // Обработка запрошенного файла
    } else if (fileInfo == null && permissionsInfo != null) {
        println("Запрос на обработку прав доступа пользователя $userId с информацией $permissionsInfo")
        // Обработка прав доступа
    } else if (fileInfo == null && permissionsInfo == null) {
        println("Общий запрос пользователя $userId без специфических параметров")
        // Обработка общего запроса
    } else {
        println("Некорректный запрос: fileInfo и permissionsInfo не могут быть указаны одновременно")
        // Обработка ошибки запроса
    }
}
```

Это пример раздутого кода из-за смешения логик (функция по-сути была универсальной и нельзя была точно назвать ее зону ответственности).
Также здесь высокая сложность добавления новой логики и высокая сложность покрытия этой функции тестами.

Разделим логику обработки на отдельные функции и использовать sealed классы или интерфейсы для представления разных типов запросов. 
Это позволит упростить и очистить основную логику от условных операторов.

**Стало:**

```kotlin
sealed class UserRequest {
    data class FileInfoRequest(val userId: Int, val fileInfo: String): UserRequest()
    data class PermissionsRequest(val userId: Int, val permissionsInfo: String): UserRequest()
    data class GeneralRequest(val userId: Int): UserRequest()
    data class InvalidRequest(val reason: String): UserRequest()
}

fun processRequest(request: UserRequest) {
    when (request) {
        is UserRequest.FileInfoRequest -> handleFileInfo(request)
        is UserRequest.PermissionsRequest -> handlePermissionsInfo(request)
        is UserRequest.GeneralRequest -> handleGeneralRequest(request)
        is UserRequest.InvalidRequest -> handleInvalidRequest(request)
    }
}

fun handleFileInfo(request: UserRequest.FileInfoRequest) {
    println("Обработка файла для пользователя ${request.userId} с файлом ${request.fileInfo}")
    // Логика обработки файла
}

fun handlePermissionsInfo(request: UserRequest.PermissionsRequest) {
    println("Обработка прав доступа пользователя ${request.userId} с информацией ${request.permissionsInfo}")
    // Логика обработки прав доступа
}

fun handleGeneralRequest(request: UserRequest.GeneralRequest) {
    println("Общий запрос пользователя ${request.userId}")
    // Логика обработки общего запроса
}

fun handleInvalidRequest(request: UserRequest.InvalidRequest) {
    println("Ошибка запроса: ${request.reason}")
    // Логика обработки неверного запроса
}
```

**Что стало лучше:**

- Читаемость: код стал значительно проще к пониманию благодаря явному разделению обработки разных типов запросов.
- Расширяемость: добавление новых типов запросов стало проще и безопаснее, так как не требует внесения изменений в сложную логику условий.
- Ответственность: каждая функция обработки теперь несет ответственность только за один тип запроса, что соответствует принципу единственной ответственности

**Выводы:**

Работая над заданием, обнаружил один неочевидный момент: что на раздутость кода, на сложные взаимосвязи между частями проекта может влиять плохая системная аналитика. 
А плохая системная аналитика зачастую является следствием поверхностного, нетщательно проработонного грумминга фичи. 
Следовательно, убежден, что разработчикам необходимо дать больше власти в принятии решения, касающихся непосредственно качества кодовой базы, и опосредственно качества ПО.
Должно быть право отказать брать фичу в разработку, если поддержка такой фичи будет слишком тяжелой, дорогой и, в силу этой сложности, вести к различным трудноуловимым багам.

Следующий момент, это несоблюдение правило бойскаута в программировании. Раздутый код, после разработки в нем какой-то фичи, становится более раздутым.
Просто сложный, запутанный код становится более сложным и запутанным. Конечно, во многом это связано с дедлайнами и сроками, но я снова убежден, что необходимо учитывать риск плохого кода при оценке задач и проектов.
И требуется иметь и выделять временной ресурс для необходимого рефакторинга, если он необходим. 
Причем на самом деле это даже более выгодно для компании в целом, так как самая большая проблема раздутого кода в его поддержке. 
На поддержку и разбор такого кода уходит очень много времени. Поэтому, рефакторинг еще умеренно сложного кода на самом деле сокращает время разработки в этом коде для других людей из команды разработки, что сокращает время разработки их фичей, а значит в целом среднее время разработки фичи уменьшается в определенных случаях.

Поэтому да, раздутый код влияет на производительность команды. Замедляется процесс разработки. 
Но чтобы справиться с этим, вдобавок к упомянутым предложениям стоит отнести и регулярный рефакторинг. Например, 20% времени спринта или в рамках 2-3 месяцев разработчиков следует уделять на поддержание качества кодовой базы.


 