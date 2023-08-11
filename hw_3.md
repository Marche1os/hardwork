1) Мобильное приложение, экран избранных товаров. 
Логический дизайн части функционала по загрузке и отображению избранных товаров на экране избранного: 
- отослать метрику открытия экрана
- показать индикатор загрузки 
- Асинхронно загрузить порцию офферов, смаппить их в модели для показа на UI. 
- Для ошибок показывать алерт и логировать throwable

Описав спецификацию, некоторые неточности явно бросаются в глаза. Очевидно, что сознательно на UI потоке делать какие-то форматирования и вычисления - не стоит, особенно если это явно в Rx цепочке. 
Также проблемный момент, что одна функция используется для множества кейсов и вызывается из нескольких мест.

Исходный код функции для загрузки данных:

```
fun loadWishlistPage() {
        if (!isRequestExecuting && hasMore) {
            isRequestExecuting = true
            if (fulfillmentViewObjects.isEmpty()) {
                viewState.showLoading()
            } else {
                viewState.showLoadingFooter(true)
            }
            Singles.zip(
                useCases.loadWishlistPage(pagingToken, configuration.pageSize),
                useCases.isNoviceBadgeEnabled(),
                useCases.getAdultState(),
            )
                .observeOn(schedulers.main)
                .doOnSubscribe(::addDisposable)
                .doOnSuccess { (wishlistPage, _, _) ->
                    offersCache.addOffers(wishlistPage.wishItems.mapNotNull { it.skuInfo?.productOffer() })
                }
                .subscribeBy {
                    onSuccess { (
                                    wishlistPage,
                                    isNoviceBadgeEnabled,
                                    adultState,
                                ) ->
                        val startPosition = fulfillmentViewObjects
                            .lastOrNull()?.position?.plus(1) ?: fulfillmentViewObjects.size
                        val newFulfillmentViewObjects =
                            fulfillmentFormatter.format(
                                wishlistPage.wishItems,
                                Screen.WISHLIST,
                                startPosition,
                                isNoviceBadgeEnabled,
                                isAuthorized.orElse(false),
                                adultState,
                                adultDailyFlagFeatureManager.isEnabled(),
                            )
                        addWishItems(newFulfillmentViewObjects, startPosition == FIRST_INDEX)

                        wishItems.addAll(wishlistPage.wishItems)
                        pagingToken = wishlistPage.token
                        hasMore = wishlistPage.hasMore
                        isRequestExecuting = false
                    }
                    onError {
                        if (it.isNetworkError) {
                            showAlert(R.string.network_error, it)
                        } else {
                            showAlert(R.string.report_dialog_title_crashes, it)
                            val requestId = it.obtainRequestErrorData()?.marketRequestId

                            wishListAnalytics.get().pageError(message = it.message, requestId = requestId)
                        }
                        if (fulfillmentViewObjects.isEmpty()) {
                            viewState.showEmpty(isAuthorized.orElse(false))
                        } else {
                            viewState.showLoadingFooter(false)
                        }
                        Timber.e(it)
                        isRequestExecuting = false
                    }
                }
        }
    }
```

Улучшенный код:

```
private fun doLoadPage(onLoadSuccess: (items: List<FulfillmentVO>) -> Unit, onLoadError: (t: Throwable) -> Unit) {
        if (CHANNEL_LOADING_PAGE.isActive || !hasMore) {
            return
        }

        Singles.zip(
            useCases.loadWishlistPage(pagingToken, configuration.pageSize),
            useCases.isNoviceBadgeEnabled(),
            useCases.getAdultState()
        ).map { (page, badgeEnabled, adultState) ->
            pagingToken = page.token
            hasMore = page.hasMore

            fulfillmentFormatter.format(
                items = page.wishItems,
                screen = Screen.WISHLIST,
                startIndex = 0,
                isNoviceBadgeEnabled = badgeEnabled,
                isLoggedIn = isAuthorized.orElse(false),
                adultState = adultState,
                isAdultDailyFlagEnabled = adultDailyFlagFeatureManager.isEnabled(),
            )
        }.schedule(
            channel = CHANNEL_LOADING_PAGE,
            onSuccess = { items ->
                onLoadSuccess.invoke(items)
            },
            onError = {
                onLoadError.invoke(it)
            }
        )
    }
```

Ввел в программный код концепцию стейта, реагирование на смену состояния экрана в едином месте 

Пример описания стейта:
```
    sealed class ScreenState {
        object FirstLoading : ScreenState()

        data class FirstPageLoaded(
            val items: List<FulfillmentVO>
        ) : ScreenState()

        object StartLoadingMore : ScreenState()

        data class LoadedMore(
            val items: List<FulfillmentVO>
        ) : ScreenState()
    }
```

Подписка на стейт:

```
    fun observeState() {
        stateFlow.asSharedFlow().onEach { state ->
            when (state) {
                is ScreenState.FirstLoading -> {
                    viewState.showLoading()
                    doLoadPage(
                        onLoadSuccess = { items ->
                            stateFlow.tryEmit(ScreenState.FirstPageLoaded(items))
                        },
                        onLoadError = { t ->
                            showAlert(R.string.report_dialog_title_crashes, t)
                        }
                    )
                }

                is ScreenState.FirstPageLoaded -> {
                    viewState.onLoadFirst(items = state.items)
                }

                is ScreenState.StartLoadingMore -> {
                    viewState.showLoadingFooter(isVisible = true)
                    doLoadPage(
                        onLoadSuccess = { items ->
                            stateFlow.tryEmit(ScreenState.LoadedMore(items))
                        },
                        onLoadError = { t ->
                            showAlert(R.string.report_dialog_title_crashes, t)
                        }
                    )
                }

                is ScreenState.LoadedMore -> {
                    viewState.showLoadingFooter(isVisible = false)
                    viewState.onLoadMore(items = state.items)
                }
            }
        }.launchInPresenterScope()
    }
```


2) Функционал инициализации панорамного фото.
Логический дизайн включает в себя
Загрузку видеофайла из videoUrl и сохранение его в локальное хранилище в качестве файла cacheFile. Возвращает Completable, чтобы сигнализировать об успешном выполнении этой задачи.

2. Методы-обработчики для loadVideo():
   - onLoadVideoComplete(): вызывается, когда загрузка видео завершена успешно. Запускает процесс декодирования кадров с использованием метода startDecodingFlow().
   - onLoadVideoError(): вызывается, когда загрузка видео завершилась ошибкой. Выводит сообщение об ошибке и закрывает состояние представления.

3. Метод startDecodingFlow(): Инициализирует процесс декодирования кадров видео, используя FrameExtractor и переданный loadedFile. Обрабатывает результат извлечения кадров через обработчики onDecodingSuccess и onDecodingError.

4. Метод createAndGetExtractor(): Инициализирует компоненты для извлечения кадров с помощью MediaExtractor, MediaCodecHelper, MediaCodec, ShaderProgram, CodecOutputSurface и FrameExtractor.

Исходный код:

```
@CheckResult
    private fun loadVideo(cacheFile: File): Completable {
        return Completable.defer {
            videoUrl.toHttpUrl()
                .toUrl()
                .openStream()
                .use { inStream ->
                    Channels.newChannel(inStream).use { channel ->
                        FileOutputStream(cacheFile).use { fos ->
                            fos.channel.transferFrom(channel, 0, Long.MAX_VALUE)
                        }
                    }
                }
            Completable.complete()
        }
    }

    private fun onLoadVideoComplete(loadedFile: File) {
        startDecodingFlow(loadedFile)
    }

    private fun onLoadVideoError(error: Throwable) {
        viewState.close()
        Timber.e(error)
    }

    private fun startDecodingFlow(loadedFile: File) {
        Single.defer {
            val frameExtractor = createAndGetExtractor(loadedFile)
            val frames = frameExtractor.extractFrames()
            Single.just(frames)
        }.schedule(
            onSuccess = ::onDecodingSuccess,
            onError = ::onDecodingError
        )
    }

    private fun createAndGetExtractor(loadedFile: File): FrameExtractor {
        val mediaExtractor = MediaExtractor().apply { setDataSource(loadedFile.absolutePath, null) }
        return MediaCodecHelper(mediaExtractor)
            .getCodecWithFormat(loadedFile)
            .let { (codec, format) ->
                val width = format.getInteger(MediaFormat.KEY_WIDTH)
                val height = format.getInteger(MediaFormat.KEY_HEIGHT)

                val shaderProgram = ShaderProgram(
                    errorChecker = GlErrorChecker,
                )

                val surface = CodecOutputSurface(
                    shaderProgram = shaderProgram,
                    glVersion = GL_VERSION,
                    width = width,
                    height = height,
                    errorHandler = ErrorHandler,
                )

                val frameExtractor = FrameExtractor(
                    surface = surface,
                    mediaExtractor = mediaExtractor,
                    mediaCodec = codec,
                    format = format,
                )
                frameExtractor
            }
    }

    private fun onDecodingSuccess(frames: List<Bitmap>) {
        viewState.hideProgress()
        this.frames = frames
        frames.firstOrNull()?.let { frame -> viewState.showFrame(frame) }
    }

    private fun onDecodingError(error: Throwable) {
        viewState.close()
        Timber.e(error)
    }
```

Изменения:

1. Заменил Completable.defer на Completable.fromRunnable, так как defer в данном случае не требуется.
2. Отделил создание Single от подписки на него в startDecodingFlow(), чтобы код был более декларативным и легко тестируемым.
3. Отделил инициализацию компонентов в отдельные функции, которые могут быть легко заменены или изменены при необходимости и упрощают рассуждение о функции на уровне дизайна

Новая версия:

```
private fun loadVideo(cacheFile: File): Completable {
    return Completable.fromRunnable {
        videoUrl.toHttpUrl()
            .toUrl()
            .openStream()
            .use { inStream ->
                Channels.newChannel(inStream).use { channel ->
                    FileOutputStream(cacheFile).use { fos ->
                        fos.channel.transferFrom(channel, 0, Long.MAX_VALUE)
                    }
                }
            }
    }
}

private fun onLoadVideoComplete(loadedFile: File) {
    startDecodingFlow(loadedFile)
        .subscribe(::onDecodingSuccess, ::onDecodingError)
}

private fun startDecodingFlow(loadedFile: File): Single<List<Frame>> {
    return Single.fromCallable {
        createAndGetExtractor(loadedFile).extractFrames()
    }
}

private fun createAndGetExtractor(loadedFile: File): FrameExtractor {
    val mediaExtractor = setupMediaExtractor(loadedFile)
    val (codec, format) = MediaCodecHelper(mediaExtractor).getCodecWithFormat(loadedFile)
    val (width, height) = format.getDimensions()
    val shaderProgram = setupShaderProgram()
    val surface = setupCodecOutputSurface(shaderProgram, width, height)

    return FrameExtractor(surface, mediaExtractor, codec, format)
}
```


3) Android приложение, функциональность индикатора страниц для списка. Логический дизайн описывается примерно так:
- Индикатор предназначен для отображения количества страниц и текущей выбранной страницы в списке
- Индикатор состоит из последовательности элементов-индикаторов, представляющих собой визуальные элементы с определенным размером и расстоянием между ними и указывает на текущий элемент списка из общего количества
- Внешний вид активного элемента (текущая выбранная страница) отличается от внешнего вида остальных элементов.
- Индикатор должен быть интегрирован с компонентом (ViewPager, RecyclerView и т.д.), который отображает страницы, для синхронизации активного элемента индикатора с текущим состоянием показываемых страниц и для обработки пользовательского взаимодействия.
Исторически, кастомные view писались в императивном стиле. Но сейчас идет тренд на создание декларативных UI и в андроиде есть относительно новый гугловый фреймворк jetpack compose для создания UI в декларативной парадигме

Исходный код в императивном стиле

```
public class ViewPagerIndicator extends LinearLayout {
    private final LayoutInflater inflater;
    private final Drawable itemDrawable;
    private final Drawable selectedItemDrawable;
    private int itemCount;

    public ViewPagerIndicator(@NonNull Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
        setOrientation(HORIZONTAL);
        setGravity(Gravity.CENTER);
        inflater = LayoutInflater.from(context);

        itemDrawable = getResources().getDrawable(R.drawable.indicator_item, context.getTheme());
        selectedItemDrawable = getResources().getDrawable(R.drawable.indicator_selected_item, context.getTheme());
    }

    public void setItemCount(int count) {
        itemCount = count;
        removeAllViews();
        for (int i = 0; i < count; i++) {
            addIndicatorItem(i);
        }
    }

    public void setSelectedIndex(int index) {
        for (int i = 0; i < getChildCount(); i++) {
            View child = getChildAt(i);
            View indicator = child.findViewById(R.id.indicator);
            indicator.setBackground(i == index ? selectedItemDrawable : itemDrawable);
        }
    }

    private void addIndicatorItem(int index) {
        FrameLayout indicatorItemContainer = (FrameLayout) inflater.inflate(R.layout.indicator_item, this, false);
        View indicator = indicatorItemContainer.findViewById(R.id.indicator);
        indicator.setBackground(index == 0 ? selectedItemDrawable : itemDrawable);
        addView(indicatorItemContainer);
    }
}
```

Код в декларативной парадигме:

```
@Composable
fun ViewPagerIndicator(
    count: Int,
    selectedIndex: Int,
    modifier: Modifier = Modifier,
    itemSize: Int = 8,
    itemSpacing: Int = 8,
    itemShape: Shape,
    itemColor: Color,
    selectedItemColor: Color
) {
    Row(modifier, horizontalArrangement = Arrangement.Center) {
        for (i in 0 until count) {
            val scaleFactor = if (i == selectedIndex) 1.2f else 1f
            Box(
                Modifier
                    .size(itemSize.dp)
                    .scale(scaleFactor)
                    .background(if (i == selectedIndex) selectedItemColor else itemColor, itemShape)
                    .width(size = itemSpacing.dp)
            )
        }
    }
}
```

```
@Composable
fun GalleryView() {
    var selectedIndex by remember { mutableStateOf(0) }
    val items = listOf("Img 1", "Img 2", "Img 3")

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        LazyRow(
            modifier = Modifier.padding(bottom = 16.dp),
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            items(items.indices) { index ->
                Text(
                    items[index], 
                    Modifier.clickable { selectedIndex = index }
                )
            }
        }

        ViewPagerIndicator(
            count = items.size,
            selectedIndex = selectedIndex,
            itemShape = CircleShape,
            itemColor = Color.Gray,
            selectedItemColor = MaterialTheme.colors.primary
        )
    }
}
```


Выводы. Вообще в ходе выполнения задания появилась мысль, что еще один шаг к повышению надежности и выразительности программы, помимо тестирования, является вынесение "запрограммированных" частей кода в дополнительные верхние слои и вынесение на уровень, который находится близко к UI, в случае создания пользовательских приложений, так как это еще и точка входа для других разработчиков, которые изучают код отдельной функциональности или планируют свои доработки в текущую функциональность. Так, создание этого верхнего слоя с декларативной составляющей явно описывает, что должно быть сделано, что в свою очередь заставляет читать будто не программный код, а дизайн компонента, словно пользователь приложения. Ну и в ходе выполнения задания были постоянно вопросы, как уменьшить количество операций, особенно когда с т.з. бизнес-требованием ты знаешь, что нужно сходить сначала в один endpoint, получить результат, который передать в другой. Вместе с тем идут постоянные обращения к UI для программирования нового состояния (убрать загрузку, показать данные или показать ошибку и т.д.). Это пытался описать в экране избранного, просто описав один раз поведение при каждом стейте. И дальше из кода легко сменить стейт для соответствующего случая. Хотя порой этих стейтов может быть много, но, как кажется, свободному рассуждению на уровне дизайна системы это не мешает.