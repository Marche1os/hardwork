```kotlin
open class DeviceBuilder<T : DeviceBuilder<T>> {
    var deviceName = ""

    fun setName(name: String): T {
        deviceName = name

        @Suppress("UNCHECKED_CAST")
        return this as T
    }
}

class LightBuilder : DeviceBuilder<LightBuilder>() {
    var isAdaptive = false

    fun setAdaptive(isAdaptive: Boolean): LightBuilder {
        this.isAdaptive = isAdaptive

        return this
    }

    fun build(): LightDevice = LightDevice(
        deviceName = deviceName,
        isAdaptive = isAdaptive,
    )
}

data class LightDevice(
    val deviceName: String,
    val isAdaptive: Boolean,
)
```

**Выводы**

Сперва паттерн показался сомнительным и не сразу уловил суть. 
В последствии увидел, что F-bounded даёт компилятору возможность понять, 
что наш билдер работает только со своими же потомками, и не даст скрестить несовместимое. 
Такое ограничение кажется немного избыточным —-но только до первых попыток расширить иерархию билдера или переиспользовать его код. 
Без F-bounded можно случайно передать какой-нибудь неожиданный тип, и тогда вся магия "каскадных" методов превращается в runtime-исключения или просто в бессмысленный код.

Особо приятно, что в рантайме ничего не ломается, и все ошибки можно поймать на этапе компиляции. 
В итоге можно спокойно расширять и развивать API билдера, не боясь, что кто-то случайно использует его не по назначению.

Пожалуй, F-bounded подталкивает к более структурированному мышлению при проектировании иерархий.