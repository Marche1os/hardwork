### 1

```kotlin
val filterParentId: (DeviceNode) -> Boolean = { node ->
    node.bean.parentUnionId != null
}

fun UnionNode.generateRoomItems() {
    val virtualRooms = room.parentId == home.id
    val devices = home.fetchAllDevices
        .filter { deviceNode -> filterParentId(deviceNode) }
        .filter { deviceNode -> !deviceNode.isSensor }
    val sensors = home.fetchAllDevices
        .filter { deviceNode -> deviceNode.isSensor }
    // дальше другие типы устройств в комнате собираются в список
}

```

- код не является свободным для понимания, так как требует знаний бизнес-требований, логики сортировки и технических нюансов для чего предназначены те или иные идентификаторы
- логика фильтрации должна быть более высокоуровневой без необходимости оперировать примитивными типами. Также нужно сделать код более надежным, сейчас он хрупок из-за отсутствия наглядных спецификаций и требований

### 2

```kotlin
fun handleSwitchAllLightDeeplink(deeplink: Uri) {
    if (canResolve(deeplink)) {
        val homeId = deeplink.extractHomeId()
        val target = deeplink.extractTargetLight() 
        
        if (homeId.isNotNullOrEmpty && target != null) {
            lightController.switchLightTo(target)
                .onSuccess {
                    // ...
                }
                .onError { error ->
                    // ...
                }
        }
    }
}
```

- Это довольно понятный код, обрабатывающий диплинк на переключение всего света в нужное положение
- Можно упростить код, сделав спецфичиную для этого диплинка проверка с учетом наличия обязательных параметров в диплинке (дом, в котором требуется переключение света, и значение)

### 3

```kotlin
/**
 * Загрузить сторис и поставить на загрузку первое изображение стори
 */
suspend fun fetchStories() = storiesInteractor.get()
        .onSuccess { stories ->
            val firstPageOfStory = stories.first()
            async { 
                imagePrefetcherInteractor.prefetch(firstPageOfStory.imageUrl)
            }
        }
```

- Да, здесь мы при успешной загрузке сторисов предзагружаем изображение, не дожидаясь резульата скачивания
- Да, это небольшая функция с понятной целью

### 4

```kotlin
class BlurHashPainter(
    private val blurHash: String?,
    private var width: Int,
    private var height: Int,
    private val punch: Float = 1f,
    private val scale: Float = 0.1f,
) : Painter() {

    private val cacheCosinesX = SparseArrayCompat<DoubleArray>()
    private val cacheCosinesY = SparseArrayCompat<DoubleArray>()

    override val intrinsicSize: Size = Size.Unspecified
    override fun DrawScope.onDraw() {
        val size = 100 * scale
        if (width > height) {
            height = (size * height / width).toInt()
            width = size.toInt()
        } else {
            width = (size * width / height).toInt()
            height = size.toInt()
        }

        if (blurHash == null || blurHash.length < 6) {
            return
        }

        val numCompEnc = decode83(blurHash, 0, 1)
        val numCompX = (numCompEnc % 9) + 1
        val numCompY = (numCompEnc / 9) + 1
        if (blurHash.length != 4 + 2 * numCompX * numCompY) {
            return
        }
        val maxAcEnc = decode83(blurHash, 1, 2)
        val maxAc = (maxAcEnc + 1) / 166f
        val colors = Array(numCompX * numCompY) { i ->
            if (i == 0) {
                val colorEnc = decode83(blurHash, 2, 6)
                decodeDc(colorEnc)
            } else {
                val from = 4 + i * 2
                val colorEnc = decode83(blurHash, from, from + 2)
                decodeAc(colorEnc, maxAc * punch)
            }
        }

        val imageArray = IntArray(width * height)
        val calculateCosX = !cacheCosinesX.containsKey(width * numCompX)
        val cosinesX = getArrayForCosinesX(calculateCosX, width, numCompX)
        val calculateCosY = !cacheCosinesY.containsKey(height * numCompY)
        val cosinesY = getArrayForCosinesY(calculateCosY, height, numCompY)
        for (y in 0 until height) {
            for (x in 0 until width) {
                var r = 0f
                var g = 0f
                var b = 0f
                for (j in 0 until numCompY) {
                    for (i in 0 until numCompX) {
                        val cosX = cosinesX.getCos(calculateCosX, i, numCompX, x, width)
                        val cosY = cosinesY.getCos(calculateCosY, j, numCompY, y, height)
                        val basis = (cosX * cosY).toFloat()
                        val color = colors[j * numCompX + i]
                        r += color[0] * basis
                        g += color[1] * basis
                        b += color[2] * basis
                    }
                }
                imageArray[x + width * y] =
                    Color.rgb(linearToSrgb(r), linearToSrgb(g), linearToSrgb(b))
            }
        }
        drawImage(
            Bitmap.createBitmap(imageArray, width, height, Bitmap.Config.ARGB_8888)
                .asImageBitmap(),
            dstSize = IntSize(
                this@onDraw.size.width.roundToInt(),
                this@onDraw.size.height.roundToInt()
            )
        )

    }
}
```

- Четко не могу сформулировать. Этот код понятно отвечает за отрисовку blurhash, но точные детали как это выглядит требует переместиться в места вызова
- В целом это можно упростить, сделав DSL и используя kotlin context, но в целях производительности количество абстракций сведено к минимуму

### 5

```kotlin
/**
 * Проверка валидности названия комнаты. 
 * Название должно состоять только из букв русского алфавита и цифр
 */
fun isValidRoomName(name: String): Boolean {
    if (name.isBlank()) return false
    if (name.length > 30) return false
    
    val regex = "^[а-яА-Я0-9\\- ]+\$".toRegex()
    return name.matches(regex)
}
```

- Да, это простая функция проверки введенного названия комнаты на валидность
- Да, это пример кода, не нуждающийся в упрощении

### Выводы

Интуиция это сильное качество в программировании. 
Благодаря интуиции в прошлом году я придумал и организовал большой технический проект, изменивший архитектуру большого куска приложения.
На первый взгляд в прошлой архитектуре все было сделано как надо: соблюдались принципы SOLID, Clean Architecture, модули, классы и файлы организованы по слоям, как в гайдах гугла.
Но было ощущение, что работа над фичами с каждым разом все усложняется, а чтобы написать простой один unit test, нужно замокать кучу всего. Строк 20 уходило на моки только чтобы тест запустился.
В конечно итоге все получилось, и интуиция сработала. :)

Выполнение задание показалось эффективным, буду чаще доверять ощущениям при работе с кодом и размышлениями о нем. 