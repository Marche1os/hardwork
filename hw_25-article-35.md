### 1.

```kotlin
class AnimationController {
    var isAnimation = false
    
    fun startAnimation() {
        if (!isAnimation) {
            isAnimation = true
            // код для запуска анимации
        }
    }
    
    fun stopAnimation() {
        if (isAnimation) {
            isAnimation = false
            // код для остановки анимации
        }
    }
}
```

**Стало:**

```kotlin
sealed class AnimationState {
    data object Running : AnimationState()
    data object Stopped : AnimationState()
}

class AnimationController {
    var state: AnimationState = AnimationState.Stopped

    fun startAnimation() {
        when (state) {
            is AnimationState.Stopped -> {
                state = AnimationState.Running
                // код для запуска анимации
            }
            else -> {}
        }
    }

    fun stopAnimation() {
        when (state) {
            is AnimationState.Running -> {
                state = AnimationState.Stopped
                // код для остановки анимации
            }
            else -> {}
        }
    }
}
```

### 2.

**Было:**

```kotlin
class Order(val orderId: String) {
    var status: String = "new" // new, processing, shipped, delivered, canceled
    var products: MutableList<String> = mutableListOf()
    var address: String = ""
    var trackingNumber: String = ""

    fun addToCart(product: String) {
        if (status == "new" || status == "processing") {
            products.add(product)
            status = "processing"
        } else {
            println("Нельзя добавить товар после отправки")
        }
    }

    fun setAddress(address: String) {
        if (status == "processing") {
            this.address = address
        } else {
            println("Нельзя изменить адрес после отправки")
        }
    }

    fun confirmOrder() {
        if (status == "processing" && products.isNotEmpty() && address.isNotEmpty()) {
            status = "shipped"
            trackingNumber = "some number" // Пример генерации трекинг-номера
        } else {
            println("Заказ не может быть подтверждён")
        }
    }

    fun markAsDelivered() {
        if (status == "shipped") {
            status = "delivered"
        } else {
            println("Заказ не был отправлен")
        }
    }

    fun cancelOrder() {
        if (status != "delivered") {
            status = "canceled"
        } else {
            println("Доставленный заказ не может быть отменён")
        }
    }
}
```

**Стало**

```kotlin
sealed class OrderState {
    data object New : OrderState()
    class Processing(val products: List<String>, val address: String) : OrderState()
    class Shipped(val trackingNumber: String) : OrderState()
    data object Delivered : OrderState()
    data object Canceled : OrderState()
}

class Order(val orderId: String) {
    var state: OrderState = OrderState.New
        private set

    fun addToCart(product: String): Boolean {
        if (state is OrderState.New || state is OrderState.Processing) {
            val products = when (state) {
                is OrderState.Processing -> state.products.toMutableList()
                else -> mutableListOf()
            } 
            
            products.add(product)
            
            state = OrderState.Processing(
                products = products,
                address = (state as? OrderState.Processing)
                    ?.address.orEmpty()
            )
            return true
        }
        return false
    }

    fun setAddress(address: String): Boolean {
        if (state is OrderState.Processing) {
            state = OrderState.Processing((state as OrderState.Processing).products, address)
            return true
        }
        return false
    }

    fun confirmOrder() {
        if (state is OrderState.Processing) {
            val processingState = state as OrderState.Processing
            if (processingState.products.isNotEmpty() && processingState.address.isNotEmpty()) {
                startShipping()
            }
        }
    }

    private fun startShipping() {
        // Имитация генерации трекинг-номера
        val trackingNumber = "someNumber"
        state = OrderState.Shipped(trackingNumber)
    }

    fun markAsDelivered() {
        if (state is OrderState.Shipped) {
            state = OrderState.Delivered
        }
    }

    fun cancelOrder() {
        if (state !is OrderState.Delivered && state !is OrderState.Canceled) {
            state = OrderState.Canceled
        }
    }
}
```

### 3.

**Было:**

```kotlin
class NavigationManager {
    var currentPage: String = "Home"
    var isAuthenticated: Boolean = false

    fun navigate() {
        when (currentPage) {
            "Home" -> {
                if (!isAuthenticated) {
                    currentPage = "Login"
                } else {
                    println("Находитесь на странице: Главная")
                }
            }
            "Details" -> {
                if (!isAuthenticated) {
                    currentPage = "Login"
                } else {
                    println("Находитесь на странице: Подробности")
                }
            }
            "Login" -> {
                isAuthenticated = true
                currentPage = "Home"
                println("Вы успешно авторизовались, перенаправление на Главную страницу")
            }
            else -> println("Неизвестная страница")
        }
    }
}
```

**Стало:**

```kotlin
sealed class AuthState {
    data object Authorized : AuthState()
    data object Unauthorized : AuthState()
}

class NavController {
    var authState: AuthState = AuthState.Unauthorized

    fun login() {
        authState = AuthState.Authorized
    }

    fun logout() {
        authState = AuthState.Unauthorized
    }

    fun navigateTo(screen: Screen) {
        when (screen) {
            is Screen.Login -> navigateToLoginScreen()
            is Screen.Main -> handleMainScreenNavigation()
            is Screen.Details -> handleDetailsScreenNavigation()
        }
    }

    private fun navigateToLoginScreen() {
        println("Переход к экрану логина")
        // Логика перехода к экрану авторизации
    }

    private fun handleMainScreenNavigation() {
        when (authState) {
            is AuthState.Authorized -> {
                println("Переход к главному экрану")
                // Логика навигации к главному экрану
            }
            is AuthState.Unauthorized -> navigateToLoginScreen()
        }
    }

    private fun handleDetailsScreenNavigation() {
        when (authState) {
            is AuthState.Authorized -> {
                // логика открытия экрана
            }
            is AuthState.Unauthorized -> navigateToLoginScreen()
        }
    }
}
```

**Выводы.**

Очень ждал занятие по такой теме, потому как на реальных проектах не встречал грамотной работы с состояниями. Как в одной из статей на хабре, на каждое новое состояние добавляется флажок в свойства класса.
И хотелось познакомиться с подходами и решениями, как решать такие ситуации, когда какой-нибудь экран может находиться в разных состояниях.
Проблема эта пыталась решиться сообществом с внедрением архитектурного подхода, называемого MVI (Model-View-Intent), которая имеет разные реализации. 
Большинство, как часто бывает, не поняло или поняло не так подход и в реальных проектах использование MVI сводится к тому, что для каждого экрана описывается набор состояний, в которых может находиться экран (в целом какая-то вьюха). 
Например, `Loading`, `Error`, `Success`. На демопримерах выглядит красиво, а когда стейтов набирается с десяток, получается тяжело разруливаемо. И на этом шаге весь недо-mvi подход ломается.
Но стоит отметить, что по крайней мере через набор стейтов все-таки мы, сообщество, немножечко улучшаем проблему с состояниями, описываем и задаем состояния явно и это улучшает метрики цикломатической сложности кода и является превентивным шагом у решению проблем с невалидными состояниями.
Грубо говоря, вьюха у нас теперь не может быть и в состоянии `Success`, и в состоянии `Error`. 

Но так как такая ситуация складывается больше по-наитию, чем по целостному и осознанному подходу, то у нас бывает состояние в стейтах. 
Например, для ошибки создается класс-состояние Error, который принимает в себя тип ошибки: клиентская, серверная, отсутствие интернета и т.д и какое-нибудь описание и, например, код ответа сервера. 
И чисто технически у нас может быть "комбинация" значений переменных, являющихся невалидной. 

Но все-таки это неплохой пример того, как простое следование какому-то архитектурному подходу, даже без полного понимания всех причин, мотивов и решаемых проблем, все-таки предотвращает определенное число багов, улучшает понимание кода и делает его более "естественным", а не "алгоритмическим".
