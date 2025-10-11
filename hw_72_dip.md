### 1

```kotlin

// data слой
interface ScenarioRepository {
    fun fetchScenariosByDeviceId(deviceId: String): List<Scenario>
}

interface ScenarioRemoteDataSource
interface ScenarioLocalDataSource

class ScenarioRepositoryImpl @Inject constructor(
    private val remoteDataSource: ScenarioRemoteDataSource,
    private val localDataSource: ScenarioLocalDataSource,
) : ScenarioRepository {

    override fun fetchScenariosByDeviceId(deviceId: String): List<Scenario> {
        TODO("Not yet implemented")
    }
}

// domain слой
interface ScenarioInteractor 
```

### 2 

```kotlin
interface AuthService {
    fun authenticate(token: String): Boolean
}

@Singleton
class AuthServiceImpl @Inject constructor(
    private val api: AuthApi
) : AuthService {
    override fun authenticate(token: String) = api.checkToken(token)
}

@HiltViewModel
class LoginViewModel @Inject constructor(
    private val authService: AuthService
) : ViewModel() {
    fun login(token: String) = authService.authenticate(token)
}
```

### 3

```kotlin
interface Logger {
    fun log(message: String)
}

class DebugLogger : Logger {
    override fun log(message: String) = Log.d("AppLog", message)
}

class UserRepositoryImpl(
    private val logger: Logger
) : UserRepository {
    override suspend fun fetchUser(id: Int): User {
        logger.log("Fetching user $id")
        // ...
    }
}
```

### 4

```kotlin
interface Navigator {
    fun navigateToDetails(userId: Int)
}

class AppNavigator(
    private val navController: NavController
) : Navigator {
    override fun navigateToDetails(userId: Int) {
        navController.navigate("details/$userId")
    }
}

class UserViewModel(
    private val navigator: Navigator
) : ViewModel() {
    fun onUserClick(id: Int) = navigator.navigateToDetails(id)
}
```

### 5

```kotlin
interface Notifier {
    
    fun send(title: String, message: String)
}

class SystemNotifier(private val context: Context) : Notifier {
    
    override fun send(title: String, message: String) {
        val notification = NotificationCompat.Builder(context, "channel")
            .setContentTitle(title)
            .setContentText(message)
            .build()
        NotificationManagerCompat.from(context).notify(1, notification)
    }
}

class MessageService(private val notifier: Notifier) {
    
    fun showWaterLeakMessage() {
        notifier.send("Протечка!", "Датчик воды обнаружил протечку")
    }
}
```


### Выводы

В Android разработке в целом сам Google помогает правильно организовывать зависимости, предлагая шаблоны проектов, в которых фундамент правильной архитектуры уже выстроен. 
И кроме этого библиотечные решения из jetpack также по своей реализации правильно направляют на соблюдение базовых архитектурных принципов, в т.ч. DIP.

В целом вывод такой:
Применение DIP в разработке улучшает архитектуру и поддерживаемость проекта. 
Код становится модульным: можно легко подменять источники данных, логеры или уведомления, не затрагивая остальную систему. 
Это снижает риск ошибок при изменениях и ускоряет внедрение новых функций. Кроме того, такой подход облегчает написание unit тестов и повышает качество кода.