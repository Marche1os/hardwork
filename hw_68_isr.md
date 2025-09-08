**Было**  
```kotlin

interface UserService {
    fun login(username: String, password: String)
    fun logout()
    fun register(username: String, password: String, email: String)
    fun getUserProfile(): User
}
```
}


**Стало**  
```kotlin
interface UserLogin {
    fun login(username: String, password: String)
    fun logout()
}

interface UserRegistration {
    fun register(username: String, password: String, email: String)
}

interface UserProfileProvider {
    fun getUserProfile(): User
}
```

---

Пример 2:

**Было**

```kotlin

interface MediaController {
    fun playAudio(file: String)
    fun playVideo(file: String)
    fun stopMedia()
    fun capturePhoto()
}

```

**Стало**

```kotlin

interface AudioPlayer {
    fun playAudio(file: String)
    fun stopAudio()
}

interface VideoPlayer {
    fun playVideo(file: String)
    fun stopVideo()
}

interface PhotoCapture {
    fun capturePhoto()
}

```

---

Пример 3:

**Было**

```kotlin

interface AnalyticsTracker {
    fun trackScreenView(screenName: String)
    fun trackButtonClick(buttonId: String)
    fun logError(error: Throwable)
    fun trackPurchase(amount: Double)
}

```

**Стало**

```kotlin

interface ScreenAnalytics {
    fun trackScreenView(screenName: String)
}

interface InteractionAnalytics {
    fun trackButtonClick(buttonId: String)
}

interface ErrorLogger {
    fun logError(error: Throwable)
}

interface PurchaseAnalytics {
    fun trackPurchase(amount: Double)
}

```

---

Пример 4:

**Было**

```kotlin
interface LocationService {
    fun getCurrentLocation(): Location
    fun startLocationUpdates()
    fun stopLocationUpdates()
    fun getLastKnownLocation(): Location
}
```


**Стало**  

```kotlin

interface LocationProvider {
    fun getCurrentLocation(): Location
    fun getLastKnownLocation(): Location
}

interface LocationUpdater {
    fun startLocationUpdates()
    fun stopLocationUpdates()
}

```

--- 

Пример 5:

**Было**

```kotlin

interface PaymentProcessor {
    fun processCreditCard(cardNumber: String, amount: Double)
    fun processSberPay(account: String, amount: Double)
    fun refundTransaction(transactionId: String)
}

```


**Стало**

```kotlin

interface CreditCardProcessor {
    fun processCreditCard(cardNumber: String, amount: Double)
}

interface SberPayProcessor {
    fun processSberPay(account: String, amount: Double)
}

interface TransactionRefund {
    fun refundTransaction(transactionId: String)
}

```

### Выводы

Выполняя задание, я убедился, что принцип ISP довольно важен и в Android‑разработке, особенно при активном использовании DI‑фреймворков вроде Dagger.  
Разделение больших интерфейсов на узкие контракты действительно снижает связность и уменьшает количество ненужных зависимостей в классах.

Задание оказалось полезным, потому что заставило вспомнить типичные антипримеры из повседневной практики.  
Оно также помогло систематизировать подход к декомпозиции интерфейсов и выработать привычку анализировать необходимость каждого метода.

При проектировании какого-то API, которым будут пользоваться коллеги-программисты я часто закладывал дополнительные функции и возможности, которые "логично" вытекают и обязательно должны понадобиться.
Но данное задание позволило по-другому взглянуть на ситуацию и прийти к выводу, что такой подход был слабым и лучшего от него отказаться.