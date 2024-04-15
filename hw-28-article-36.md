### 1.

```kotlin
private lateinit var networkApi: AuthApi
private lateinit var logger: Logger
private lateinit var storage: AuthStorage

fun tryUpdateToken(refreshToken: String) {
    logger.logUpdatingToken()

    val result = networkApi.updateToken(refreshToken)

    if (result.isSuccessful) {
        storage.writeToken(result.newToken)
        logger.logUpdateTokenSuccess()
        return
    }

    logger.logUpdateTokenFailure()
    /**
     * Автономный кусок кода: 
     * Операция разлогин (logout) и уведомление остальной части системы о разлогине 
     */
    storage.logout()
    eventBus.onLogout()
}
```

### 2.
```kotlin

private lateinit var viewBinding: ProfileFragmentViewBinding
private lateinit var viewModel: ProfileViewModel
private lateinit var profileData: ProfileData

/**
 * Традиционно в методе-callback'е экрана многое намешано, например: 
 * - Инициализацию UI-элементов значениями
 * - Инициализация списков
 * - Создание слушателей пользовательского ввода. 
 * Соответствующие куски кода выделены в блоки
 */
override fun onViewCreated(inflater: LayoutInflater) {
    super(inflater)

    // Присваиваем значения UI-элементам
    viewBinding.title.text = "Профиль пользователя"
    viewBinding.username.text = profileData.fullName
    viewBinding.profileIcon.image = Glide.with(this).load(profileData.avatarUrl)
    
    // Инициализируем список
    val eventsAdapter = EventsAdapter()
    viewBinding.lastEvents.adapter = eventsAdapter
    
    // Создаем слушателей для отслеживания пользовательского ввода
    viewBinding.profileIcon.setOnClickListener { viewModel.onProfileIconClick() }
    viewBinding.back.setOnClickListener { onBackPressed() }
}
```

### 3.

```kotlin
private lateinit var lightDevicesHolder: Holder

fun generateHomeHeader(homeTree: HomeTree, allDevices: List<Device>): UiItem {
    // Кусок 1: подсчитываем количество всех устройств
    val totalDevicesInHome = allDevivices.size + homeTree.flatDevices().size
    
    val homeBackgroundUrl = allHomes.currentHome.backgroundUrl
    
    // Кусок 2: дополнительно кешируем устройства по управлению светом, 
    // чтобы при открытии только этого вида устройств не тратить время на трансформации,
    // а сразу показать готовые к показу устройства
    lightDevicesHolder.write(allDevicesToLightDevicesMapper.map(allDevices))
    val mappedHomeDevices = allDevicesToHomeDevicesMapper.map(allDevices)
    
    return HomeHeaderUiItem(
        total = totalDevicesInHome,
        backgroundUrl = homeBackgroundUrl,
        devices = mappedHomeDevices,
    )
    
}
```

### 4.

```kotlin
fun selectRoom(roomId: String) {
    val toggleSelector = content.roomItems
        .find { it.groupId == roomId }
        ?.deviceItems
        ?.firstInstanceOf<RoomSelector>()
        ?.first()
        ?: return

    // Просто меняем положение свитчера (выбрана ли комната) в обратную позицию
    toggleSelector.isIncludeAll = !toggleSelector.isIncludeAll

    viewModelScope.launch(dispatchers.io) {
        /**
         * Автономный кусок: высчитывает и меняет свитчер в нужную позицию у каждого устройства в комнате
         */
        content
            .roomItems
            .filter { it.groupId == toggleSelector.roomId }
            .flatMap { it.deviceItems.filterIsInstance<DeviceItem.Item>() }
            .forEach { it.isChecked = toggleSelector.isIncludeAll }

        /**
         * Автономный кусок кода: высчитывает, выбраны ли все комнаты с устройствами
         */
        content.isSelectAll.value = content
            .roomItems
            .flatMap { it.deviceItems.filterIsInstance<RoomSelector>() }
            .all { it.isIncludeAll }
    }
}
```

### 5.

```kotlin
/**
 * Обрабатывает заказы, обновляет инвентарь и рассчитывает общую сумму продаж.
 * @param orders список заказов
 * @param inventory текущий инвентарь
 * @return общая сумма продаж
 */
fun processOrders(orders: List<Pair<Int, Int>>, inventory: MutableMap<Int, Int>): Double {
    var totalSales = 0.0

    orders.forEach { (productId, quantityOrdered) ->
        // Поиск продукта в инвентаре и проверка наличия достаточного количества
        val currentStock = inventory[productId] ?: 0
        if (quantityOrdered <= currentStock) {
            // Это место выполняет: 1) списание товара со склада, 2) расчет суммы продаж
            inventory[productId] = currentStock - quantityOrdered  // Обновление инвентаря
            totalSales += getProductPrice(productId) * quantityOrdered // Добавление к общей сумме продаж
        }
    }

    return totalSales
}

```


**Выводы**

Один из основных инсайтов, который получил в ходе занятия, это то, что раздутость кода необязательно связана только с большим количеством кода. 
Суть в сложности и сложных взаимосвязях между различными блоками кода. 
Нередко можно встретить код (часто у тех, кто только прошел какие-нибудь курсы/лекции по чистому коду) чрезмерное использование функций.
Якобы если любую осмысленную операцию оформлять в виде функции, то код становится как книгой. 
Но упускается момент, что чаще всего этот код в отдельно взятой функции не представляет никакого интереса в отрыве от того кода, где эта функция вызывается.
Нужно комплексное видение всего блока кода того компонента, в котором в настоящий момент работаешь. И это приводит лишь к дополнительной нагрузке, когда приходится прыгать между функциями и не терять контекст между ними.
К слову, одно из последних обновлений Android Studio привносит функцию, помогающую оставаться в контексте кода компонента.
Так вот, часто, если нет дублирования, то вполне достаточно оформить слабо или неявно согласованный код в одном участке кода, при условии правильного разделения в виде комментариев и документации этих "автономных" блоков кода.

Однако, опасность раздутости может быть и в том, что одна и та же конструкция может использоваться для разных целей, 
и иногда это приводит к излишней вычислительной работе. Поэтому, думается, важно учитывать "потенциал" кода содержащего мешанину, к переиспользованию. 

В целом из этого следует то, что мы должны стремиться к уменьшению раздутости кода, сохраняя при этом наглядность, очевидность и простоту к внесению изменений.
Использование структурированных комментариев для разделения логически различных частей кода может облегчить его понимание и последующую поддержку.