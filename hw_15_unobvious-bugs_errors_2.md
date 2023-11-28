1. Первый пример рефакторинга. Представим архитектуру MVI (Model-View-Intent), в команде утвердили следование этой архитектуре.

Но реализация такого подхода была "ручной": требовалось создавать состояния, структуру, управляющую состоянием.
Что приводило к различиям в реализации у разных людей в команде.

Поэтому был реализован открытый к наследованию тип, задающий структуру архитектуру. Так, границы нашего подхода установлены на уровне интерфейса.

Код задает границы использования подхода MVI у наследников, направляет на корректную и правильную реализацию ViewModel в проекте. 

```kotlin
class CheckouterViewModel(
    private val bucketIds: List<String>,
    private val defaultDeliveryOptions: List<DeliveryOption>,
    private val deliveryOptionsRepository: DeliveryOptionsRepository,
    //...
) : ViewModel() {

    vat deliveryOptionsData: LiveData<State> = MutableLiveData<DeliveryOption>

    init {
        loadActualDeliveryOptions()
    }

    private fun loadActualDeliveryOptions() {
        viewModelScope.launch { deliveryOptionsRepository.getDeliveryOptions(bucketIds) }
    }
}
```


```kotlin
class CheckouterViewModel(
    private val bucketIds: List<String>,
    private val defaultDeliveryOptions: List<DeliveryOption>,
    private val deliveryOptionsRepository: DeliveryOptionsRepository,
    //...
) : MviViewModel<CheckouterState, CheckouterIntent, CheckouterEffect>() {

    vat deliveryOptionsData = FlowData<DeliveryOption>
    
    ovveride fun initialState() = CheckouterState.Loading

    fun processIntent(intent: CheckouterIntent) {
        when (intent) {
            is CheckouterIntent.LoadDeliveryOptions -> loadActualDeliveryOptions()
        }
    }

    private fun loadActualDeliveryOptions() {
        viewModelScope.launch { deliveryOptionsRepository.getDeliveryOptions(bucketIds) }
    }
}

sealed class CheckouterIntent {
    data object LoadDeliveryOptions : CheckouterIntent()
}

sealed class CheckouterState {
    object Loading : CheckouterState()
}

sealed class CheckouterEffect {
    data object ShowNotification(data: AlertData) : CheckouterEffect()
}

```

2. Пример с системой лайков 

```kotlin
class FeedFragment : Fragment() {

    private fun setupLikeSystem() {
        likeButton.setOnClickListener { 
            val postId = getPostIdFromView(it)
            if (isPostLiked(postId)) {
                unlikePost(postId)
            } 
            if (!isPostLiked(postId)) {
                likePost(postId)
            }
        }
    }

    private fun likePost(postId: String) {
        // Логика добавления лайка к посту
    }

    private fun unlikePost(postId: String) {
        // Логика удаления лайка с поста
    }

    private fun isPostLiked(postId: String): Boolean {
        // Проверка состояния лайка поста
        return true
    }

    private fun getPostIdFromView(view: View): String {
        // Логика получения идентификатора поста из представления
        return "some_post_id" // Заглушка примера
    }
    
    // ... остальная часть активности ...
}
```

При рефакторинге вынесли из монолита в отдельный модуль логику управления лайками. Была переосмыслена архитектура, оформлена отдельной фичей и теперь может использоваться из множества мест.

Граница like_system_feature обеспечивает просто и понятный интерфейс взаимодействия с отметкой "Нравится" у публикации.

Влияние на систему: использование фичи становится независимым от места ее применения, фича изолировано покрыта тестами и не влияет на код другой функциональности, например, комментариев.

```kotlin
/**
 * Компонент, отвечающий за лайки публикациям. 
 * Оперирует id публикации, может использоваться из ленты, профиля и других мест.
 */
interface ILikeSystem {
    fun likePost(postId: String)
    
    fun unlikePost(postId: String)
    
    fun isPostLiked(postId: String): Boolean
}

class LikeSystem : ILikeSystem {
    override fun likePost(postId: String) {
        // Реализация логики добавления лайка
    }

    override fun unlikePost(postId: String) {
        // Реализация логики удаления лайка
    }

    override fun isPostLiked(postId: String): Boolean {
        // Реализация проверки состояния лайка
        return true 
    }
}

class FeedFragment : Fragment() {
    private lateinit var likeSystem: ILikeSystem

    override onViewCreated(view: View?) {
        likeSystem = LikeSystem()

        setupLikeSystem()
    }
    
    private fun setupLikeSystem() {
        likeButton.setOnClickListener {
            val postId = getPostIdFromView(it)
            if (likeSystem.isPostLiked(postId)) {
                likeSystem.unlikePost(postId)
            } else {
                likeSystem.likePost(postId)
            }
        }
    }
}
```

3. Реализация навигации в приложении. Проект новой навигации у нас в проекте в разработке на данный момент и у меня задача выработать решение. Примера кода нет, опишу словами.

До рефакторинга: Запутанная логика навигации, использующаяся внутри активити и фрагментов

Рефакторинг: Разработка собственного решения с DSL для гибкой навигации в приложении между экранами, на основе так называемого графа навигации

Границы: Навигационный граф определяет явные пути перехода между фрагментами и активити.

Влияние на систему: Упрощение логики навигации, более надёжное управление бэкстэка и параметрами между экранами.

4. Тест на базовый сценарий покупки "Открыть приложение, выбрать товар и перейти на карточку товара, добавить товар и перейти к немедленной покупке, совершить оплату" 

```kotlin
@Test
fun testNavigationThroughPurchaseProcess() {
    // 1. Открытие главной страницы
    onView(withId(R.id.homeFragment)).check(matches(isDisplayed()))

    // 2. Выбор продукта
    onView(withId(R.id.productList)).perform(
        RecyclerViewActions.actionOnItemAtPosition<ProductViewHolder>(
            0,
            click()
        )
    )

    // Переход к детальной странице продукта
    onView(withId(R.id.productDetailFragment)).check(matches(isDisplayed()))

    // 3. Нажатие кнопки "Купить"
    onView(withId(R.id.buyButton)).perform(click())

    // Проверка перехода к процессу оформления заказа
    onView(withId(R.id.checkoutFragment)).check(matches(isDisplayed()))

    // 4. Подтверждение покупки
    onView(withId(R.id.confirmButton)).perform(click())

    // Проверка перехода к экрану подтверждения покупки
    onView(withId(R.id.purchaseConfirmationFragment)).check(matches(isDisplayed()))

    // Проверка того, что покупка была совершена
    onView(withId(R.id.confirmationMessage)).check(matches(withText(containsString("Покупка подтверждена"))))
}

```


**Выводы**

Разобрал материал про 7 неочевидных ошибок, думаю полезно будет к нему возвращаться время от времени. 
Успел опробовать провести ревью пулл-реквеста с новым пониманием и видением того, на что обращать внимание. Заметил, что такой подход помогает лучше понимать суть изменений, выделять идею пулл-реквеста и мержить ее с компонентом, в который вносятся изменения.

По реорганизации кода по-другому. Заметил полезную методику рассуждения, которая поможет прийти к правильному результату. 
Суть в том, что не стоит вчитываться в код, стоит только пробежаться по нему, чтобы понять идею. Лучше всего, если будет документация или описание происходящего.
И после понимания продуктового смысла кода начать рассуждения о нем. Зачастую в голову приходят более краткие, лаконичные конструкции. 
Можно в стиле Льва Толстова на 2 книжные страницы описывать, как девушка встала с кровати и влезла в тапки, а можно это выразить одним предложением. :)

Основываясь на третьей идее я предложил и буду продвигать идею навигации. Ключевое будет в том, чтобы сделать API взаимодействия с навигацией универсальным.
В сущности, навигация может содержать любую реализацию, но API должно быть постоянным и пережить любые изменения и трансформации.

Также в последнее время начал находить идеи для создания DSL на конкретную часть функционала. Попробую создать DSL для одного из компонент и поделиться результатами.
