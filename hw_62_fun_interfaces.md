### Пример 1 

До

```kotlin
class LightController {
    fun switchOn() {
        // включение света
    }

    fun switchOff() {
        // выключение света
    }
}
```

После

```kotlin
fun interface LightControl {
    fun apply()
}

val switchOn: LightControl = LightControl { 
    
}

val switchOff: LightControl = LightControl {
    
}
```

### Пример 2

До

```kotlin
class Thermostat {
    fun increaseTemperature() {
        
    }

    fun decreaseTemperature() {
        
    }
}
```

После

```kotlin
fun interface TemperatureControl {
    fun change()
}

val increaseTemperature: TemperatureControl = TemperatureControl {
    
}

val decreaseTemperature: TemperatureControl = TemperatureControl {
    
}
```

### Пример 3

До
```kotlin
class CurtainController {
    fun openCurtains() {
        // открытие штор
    }

    fun closeCurtains() {
        // закрытие штор
    }
}
```

После
```kotlin
fun interface CurtainAction {
    fun execute()
}

val openCurtains: CurtainAction = CurtainAction {
    // открытие штор
}

val closeCurtains: CurtainAction = CurtainAction {
    // закрытие штор
}
```


### Выводы

Выполняя задание по организации функционального интерфейса выделил для себя следующие моменты:

1. Функциональные интерфейсы предоставляют возможность инкапсуляции логики, оставляя код чистым и легко читаемым.

2. Использование функциональных интерфейсов упрощает добавление новых функциональностей. Например, если в будущем потребуется добавить новые операции с освещением или мультимедиа, это будет сделать намного проще.

3. Упрощается процесс модульного тестирования. Каждый функциональный интерфейс можно протестировать отдельно, не затрагивая другие части системы.

4. Смена парадигмы мышления. Этот подход побуждает думать о проблемах в терминах действий и операций, а не объектов и состояний, что может значительно улучшить дизайн системы.