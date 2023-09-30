1.1. Есть класс, в котоором существует метод validateProfile, который имеет публичный доступ только из-за того, чтобы вызывать его из теста.

 Исходный код:
 ```
    class ProfileManager {
        fun updateProfile(userProfile: UserProfile) {
            // обновление профиля пользователя
            validateProfile(userProfile)
            // другая логика
        }

        fun validateProfile(userProfile: UserProfile) {
            // проверка валидности данных профиля
        }
    }

    class ProfileManagerTest {
        @Test
        fun testValidateProfile() {
            val profileManager = ProfileManager()
            val userProfile = UserProfile("John", "Doe", 25)
            profileManager.validateProfile(userProfile)
            // проверка результата
        }
    }
```

В качестве улучешния был осуществлен переход на механизм рефлексии, что позволило убрать завязанный на тесты код.

Улучшенный код:

```
    class ProfileManager {
        fun updateProfile(userProfile: UserProfile) {
            // обновление профиля пользователя
            validateProfile(userProfile)
            // другая логика
        }

        private fun validateProfile(userProfile: UserProfile) {
            // проверка валидности данных профиля
        }
    }

    class ProfileManagerTest {
        @Test
        fun testValidateProfile() {
            val profileManager = ProfileManager()
            val userProfile = UserProfile("John", "Doe", 25)

            val validateProfileMethod = ProfileManager::class.java.getDeclaredMethod("validateProfile", UserProfile::class.java)
            validateProfileMethod.isAccessible = true
            validateProfileMethod.invoke(profileManager, userProfile)

            // проверка результата
        }
    }
```


1.2 Со следующим код типовой, не с конкретной задачи. Часто сталкивался с длинными цепочками вызовов методов, которые подаются как параметры. Понимать и поддерижвать такое тяжело.

Исходный код вида:
```
class CartDomain {
    // специфичная логика домена
    ...
}

class CartUi {
    static fromDomain(cartDomain: CartDomain) {
        return this.fromCartDto(
                CartService.DtoFromDomain(
                    CartMapper.DomainToDto(
                        CartAdjustments.Adjust(
                              cartDomain))));
    }
    
    ...
}

class CartService {
    static DtoFromDomain(cartDomain: CartDomain) {
        // конвертация типов
        ...
    }
    ...
}

class CartMapper {
    static DomainToDto(cartDomain: CartDomain) {
        // маппинг сущности
        ...
    }
    ...
}

class CartAdjustments {
    static Adjust(cartDomain: CartDomain) {
        // выполнение дополнений
        ...
    }
    ...
}
```

В данном случае у нас есть цепочка вызовов: fromDomain вызывает DtoFromDomain, которая вызывает DomainToDto, которая в свою очередь вызывает Adjust. Это сложная конструкция, которую сложно понять и отследить.

Мы можем сделать эту цепочку короче и читабельнее, изменив подход к вызову функций и делая этапы преобразования более явными. Соответствующие вызовы функций должны быть вынесены из fromDomain, чтобы каждую функцию можно было вызвать отдельно, и улучшен отслеживаемый порядок выполнения. Это может упростить понимание кода и сделать отладку более простой.


Исправленный код:
```
class CartUi {
    static fromDomain(cartDomain: CartDomain) {
        const adjCart = CartAdjustments.Adjust(cartDomain);
        const dtoCart = CartMapper.DomainToDto(adjCart);
        const finalCart = CartService.DtoFromDomain(dtoCart);
        return this.fromCartDto(finalCart);
    }
    ...
}
```

Здесь каждый шаг преобразования отражен отдельно и легко отслеживается, что делает код более читаемым и поддерживаемым.


1.3 По мотивам кода Telegram - метод с большим количееством параметров

Исходный код:
```
public void processUserData(String name, int age, String address, String email, String phoneNumber, boolean isAdmin, boolean isPremium) {
    // Логика обработки данных
}
```

//Улучшенный код с применением паттерна Builder

```
public class UserData {
    private String name;
    private int age;
    private String address;
    private String email;
    private String phoneNumber;
    private boolean isAdmin;
    private boolean isPremium;

    private UserData(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.address = builder.address;
        this.email = builder.email;
        this.phoneNumber = builder.phoneNumber;
        this.isAdmin = builder.isAdmin;
        this.isPremium = builder.isPremium;
    }

    public static class Builder {
        private String name;
        private int age;
        private String address;
        private String email;
        private String phoneNumber;
        private boolean isAdmin;
        private boolean isPremium;

        public Builder setName(String name) {
            this.name = name;
            return this;
        }

        public Builder setAge(int age) {
            this.age = age;
            return this;
        }

        public Builder setAddress(String address) {
            this.address = address;
            return this;
        }

        public Builder setEmail(String email) {
            this.email = email;
            return this;
        }

        public Builder setPhoneNumber(String phoneNumber) {
            this.phoneNumber = phoneNumber;
            return this;
        }

        public Builder setAdmin(boolean isAdmin) {
            this.isAdmin = isAdmin;
            return this;
        }

        public Builder setPremium(boolean isPremium) {
            this.isPremium = isPremium;
            return this;
        }

        public UserData build() {
            return new UserData(this);
        }
    }

    public void processData(final UserData data) {
        // Логика обработки данных
    }
}
```

1.4. По мотивам прошлого проекта с измененной предметной областью. Когда есть класс-процессинг заказа, работающий с множеством значений типов оплаты.

Исходный код:

```
class OrderProcessor {
    fun processOrder(order: Order) {
        when (order.paymentMethod) {
            "credit_card" -> processCreditCardOrder(order)
            "paypal" -> processPaypalOrder(order)
            "bitcoin" -> processBitcoinOrder(order)
            else -> throw IllegalArgumentException("Unsupported payment method")
        }
    }

    private fun processCreditCardOrder(order: Order) {
        // Process credit card order logic
    }

    private fun processPaypalOrder(order: Order) {
        // Process PayPal order logic
    }

    private fun processBitcoinOrder(order: Order) {
        // Process Bitcoin order logic
    }
}
```

Здесь мы имеем класс OrderProcessor, который имеет метод processOrder, принимающий объект order. В зависимости от типа платежного метода в заказе, разные методы вызываются для обработки заказа. 

Такой подход может привести к несогласованности и трудностям при добавлении новых платежных методов. Каждый раз, когда добавляется новый метод, приходится модифицировать класс OrderProcessor и код становится все более запутанным.

Для решения этой проблемы, я использовал паттерн Strategy. Вместо использования условий внутри одного метода, мы создадим отдельные классы-стратегии для каждого платежного метода и воспользуемся полиморфизмом.


Улучшенный код:
```
interface PaymentStrategy {
    fun processOrder(order: Order)
}

class CreditCardPaymentStrategy : PaymentStrategy {
    override fun processOrder(order: Order) {
        // Process credit card order logic
    }
}

class PaypalPaymentStrategy : PaymentStrategy {
    override fun processOrder(order: Order) {
        // Process PayPal order logic
    }
}

class BitcoinPaymentStrategy : PaymentStrategy {
    override fun processOrder(order: Order) {
        // Process Bitcoin order logic
    }
}

class OrderProcessor(private val paymentStrategy: PaymentStrategy) {
    fun processOrder(order: Order) {
        paymentStrategy.processOrder(order)
    }
}
```

Теперь у нас есть класс OrderProcessor, принимающий объект стратегии в конструкторе. Метод processOrder вызывает processOrder конкретной стратегии. Каждая стратегия предоставляет свою собственную реализацию для обработки заказов, обеспечивая единообразие и удобство добавления новых стратегий без модификации основного класса. 

1.5 Проблема с методами, возвращающими больше данных, чем требуется, заключается в том, что это может затруднить понимание того, какие данные действительно важны, и это может привести к неэффективному использованию памяти.

Исходный код:
```
data class User(val id: String, val name: String, val email: String)
```

```
class UserService {
    fun getUser(userId: String): User {
        val user = service.getUserById(userId)
        return user
    }
}
```


В некоторых случаях нам нужно получить имя пользователя для отображения, а не все детали пользователя. В качестве одного из решений этой проблемы может быть введение нового метода, который возвращает только имя пользователя:

```
class UserService {
    fun getUserDetails(userId: String): User {
        val user = service.getUserById(userId)
        return user
    }

    fun getUserName(userId: String): String {
        val user = service.getUserById(userId)
        return user.name
    }
}
```

Также иногда может быть удобно создавать специальную модель, содержащую необходимые поля. Например, UserInfoUI. 



**Выводы**

Проблемы плохого стиля кодирования, которые мы разбирали, все негативно влияют на ясность кода, что влечет увеличение времени, которое уходил на погружение в такой компонент, что ведет к увеличению времени разработки, что влечет к потерям денег компании. Кстати, по проблеме, когда один компонент возвращает больше данных, чем нужно компоненту, была такая проблема на последнем проекте. Бекенд использовался и для мобильных клиентов, и для фронта. Из-за чего ответ часто был избыточным, притом сильно. И исправление такой пробелы заключалось в создании своего прокси-бекенда силами мобильных команд. Что, конечно, было дорого и заняло много месяцев на разработку, переезд, отладку, обкатывание на части пользователей, исправлении багов и так далее. Хотя, казалось, этого можно было избежать на стадии проектирования. Но, справедливости ради, ситуация с точки зрения бизнеса была непростой, возможно, что иного выбора не было на тот момент. Но хорошо показывает, как некоторые несовершенства в начале разработки и стадии проектирования могут дорого обходиться в будущем.
Итак, можно подытожоить, что проблемы, которые разобрали, ведут к результатам, описанным выше: удоражанию разработки и потере денег и времени.

