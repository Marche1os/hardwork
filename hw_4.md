1)

```
/**
 * Компонент управления навигацией в приложении.
 * Технически навигация в приложении построена через механизм маршрутизации (Router)
 * Сюда входят все действия с навигацией (переход по системной кнопке назад, построение цепочек экранов, запуск flow-сценариев)
 * А также получение/отправка результата между экранами
 */

public class Router implements NavigationDelegate {

    @NonNull
    private final NavigationContext navigationContext;
    @NonNull
    private final Lazy<NavigationDispatcher> navigationDispatcher;
    @NonNull
    private final Lazy<NavigationHealthFacade> navigationHealthFacade;
    @NonNull
    private final Lazy<MetricaSender> metricaSender;

    @Inject
    public Router(
            @NonNull final NavigationContext navigationContext,
            @NonNull final Lazy<NavigationDispatcher> navigationDispatcher,
            @NonNull final Lazy<NavigationHealthFacade> navigationHealthFacade,
            @NonNull final Lazy<MetricaSender> metricaSender) {

        this.navigationContext = Preconditions.checkNotNull(navigationContext);
        this.navigationDispatcher = Preconditions.checkNotNull(navigationDispatcher);
        this.navigationHealthFacade = Preconditions.checkNotNull(navigationHealthFacade);
        this.metricaSender = Preconditions.checkNotNull(metricaSender);
    }

    @Override
    @NonNull
    public Screen getCurrentScreen() {
        return navigationContext.currentScreen();
    }

    @Override
    @NonNull
    public Optional<Screen> getCurrentFlow() {
        return Optional.ofNullable(navigationContext.currentFlow());
    }

    @NonNull
    public Optional<Screen> getSourceScreen() {
        return Optional.ofNullable(navigationContext.sourceScreen());
    }

    @Override
    public void navigateTo(@NonNull final TargetScreen targetScreen) {
        dispatchNavigationTo(targetScreen);
        reportStartScreenOpening(targetScreen);
    }

    private void dispatchNavigationTo(@NonNull final TargetScreen targetScreen) {
        if (targetScreen.getScreen() != Screen.UNKNOWN) {
            final Command forward = NavigationDispatcher.createForward(getCurrentTab(), getCurrentScreen(),
                    targetScreen);
            navigationDispatcher.get().dispatch(forward);
        }
    }

    private void reportStartScreenOpening(@NonNull final TargetScreen targetScreen) {
        metricaSender.get().startScreenOpening(targetScreen);
    }

    @Nullable
    public Tab getCurrentTab() {
        return navigationContext.currentTab();
    }

    public void navigateForResult(
            @NonNull final TargetScreen targetScreen,
            @NonNull final ResultListener listener) {

        Preconditions.requireNotNull(targetScreen);
        Preconditions.requireNotNull(listener);
        navigationDispatcher.get().addResultListener(getCurrentScreen(), targetScreen.getScreen(), listener);
        navigateTo(targetScreen);
    }

    public void replace(@NonNull final TargetScreen targetScreen) {
        dispatchReplace(targetScreen);
        metricaSender.get().startScreenOpening(targetScreen);
    }

    private void dispatchReplace(@NonNull final TargetScreen targetScreen) {
        navigationDispatcher.get().dispatch(
                Replace.builder()
                        .setCurrentTab(getCurrentTab())
                        .setTargetScreen(targetScreen.getScreen())
                        .setCurrentScreen(getCurrentScreen())
                        .setParams(targetScreen.getParams())
                        .build()
        );
    }

    public void back() {
        NavigationDispatcher navigationDispatcher = this.navigationDispatcher.get();
        navigationDispatcher.dispatch(Back.create(getCurrentTab()));
        navigationDispatcher.notifyOnResult(getCurrentScreen(), null);
    }

    public void backTo(@NonNull final TargetScreen targetScreen) {
        navigationDispatcher.get().dispatch(
                BackTo.builder()
                        .currentTab(getCurrentTab())
                        .screen(targetScreen.getScreen())
                        .params(targetScreen.getParams())
                        .build()
        );
    }

    public void backWithResult(@NonNull final Object result) {
        Preconditions.requireNotNull(result);
        NavigationDispatcher navigationDispatcher = this.navigationDispatcher.get();
        navigationDispatcher.dispatch(Back.create(getCurrentTab()));
        navigationDispatcher.notifyOnResult(getCurrentScreen(), result);
    }

    public void setResult(@NonNull final Object result) {
        Preconditions.requireNotNull(result);
        navigationDispatcher.get().notifyOnResult(getCurrentScreen(), result);
    }

    public void setResult(@NonNull final Screen sourceScreen, @NonNull final Object result) {
        Preconditions.requireNotNull(sourceScreen);
        Preconditions.requireNotNull(result);
        navigationDispatcher.get().notifyOnResult(sourceScreen, result);
    }

    public void navigateToChain(@NonNull final ScreenChain screenChain) {
        for (TargetScreen screen : screenChain.getTargetScreens()) {
            navigateTo(screen);
        }
    }

    public void navigateToTab(@NonNull final Tab tab) {
        Preconditions.requireNotNull(tab);
        navigationDispatcher.get().dispatch(ForwardTab.create(tab));
    }

    public void finishFlow() {
        navigationDispatcher.get().dispatch(new FinishFlow());
    }

    public void finishFlow(@NonNull final Screen currentFlow) {
        navigationDispatcher.get().dispatch(new FinishFlow(currentFlow));
    }

    public void setFlowResult(@NonNull final Object result) {
        Preconditions.requireNotNull(result);
        NavigationDispatcher navigationDispatcher = this.navigationDispatcher.get();
        navigationDispatcher.notifyOnResult(getCurrentFlow().orElse(getCurrentScreen()), result);
    }

    public void finishFlowWithResult(@NonNull final Object result) {
        Preconditions.requireNotNull(result);
        NavigationDispatcher navigationDispatcher = this.navigationDispatcher.get();
        navigationDispatcher.dispatch(new FinishFlow());
        navigationDispatcher.notifyOnResult(getCurrentFlow().orElse(getCurrentScreen()), result);
    }

    public void unregisterResultListeners() {
        navigationDispatcher.get().removeResultListeners(getCurrentScreen());
    }
}
```


2) Здесь представлена отредактированная часть исходного кода, т.к. NDA


```
/**
 * Компонент для обработки оплаты. В контексте всей системы мобильного приложения маркетплейса, 
 * данный компонент по-сути является обработчиком выбранного способа оплаты и отвечает за обраотку соответствующего способа оплаты
 * Примеры способов оплаты:
 * - Наличные
 * - картой онлайн
 * - Кредит от банка-партнера
 */
class PaymentLauncherPresenter constructor(
    private val useCases: PaymentLauncherUseCases,
    private val router: Router,
    private val paymentResultMapper: NativePaymentResultMapper,
    private val checkoutParamsMapper: CheckoutParamsMapper,
    private val commonErrorPresentationClassifier: CommonErrorPresentationClassifier,
    private val syncServiceMediator: SyncServiceMediator,
    private val analyticsService: AnalyticsService,
    private val eventBus: EventBus,
    private val checkoutCreditLimitHealthFacade: CheckoutCreditLimitHealthFacade,
) : BasePresenter() {

    fun onLaunchPaymentRequest(
        paymentParams: PaymentParams,
        sourceScreen: Screen,
        redirectStrategy: PaymentRedirectStrategy,
        replaceStrategyForThreeDs: Boolean,
        shouldSetSelectedCardOnStationSubscription: Boolean,
    ) {
        when (paymentParams) {
            is PaymentParams.CardPayment.CreditLimit -> {
                launchCreditLimitPayment(paymentParams, sourceScreen, redirectStrategy)
            }

            is PaymentParams.CreditBroker -> {
                launchCreditBrokerPayment(paymentParams)
            }

            is PaymentParams.CardPayment.Regular -> {
                launchRegularPayment(
                    paymentParams,
                    sourceScreen,
                    redirectStrategy,
                    replaceStrategyForThreeDs
                )
            }
        }
    }

    private fun launchCreditLimitPayment(
        paymentParams: PaymentParams.CardPayment.CreditLimit,
        sourceScreen: Screen,
        redirectStrategy: PaymentRedirectStrategy
    ) {
        //payment by credit
    }

    fun onCancelPaymentProgressRequest() {
        viewState.showPaymentProgress(false)
    }

    fun onRetryPaymentClick() {
        //retry payment
    }

    private fun getCashierOrderDetails(orders: List<OrderPaymentMethodInfo>): CashierOrderDetails? {
        return if (orders.all { order -> order.cashierOrderInfo == null }) {
            null
        } else {
            CashierOrderDetails(
                orderParams = orders.map { order ->
                    OrderParams(
                        id = order.id,
                        buyerTotal = order.cashierOrderInfo?.totalAmount ?: BigDecimal.ZERO,
                        isCashierAvailable = order.cashierOrderInfo?.isCashier.orFalse()
                    )
                },
                isCashierAvailable = orders.all { order ->
                    order.cashierOrderInfo?.isCashier.orFalse()
                })
        }
    }

    private fun prepareNativePayment(
        paymentParams: PaymentParams.CardPayment,
        paymentMethod: PaymentMethod,
        sourceScreen: Screen,
        redirectStrategy: PaymentRedirectStrategy,
        creditLimitLengthInMonths: Int?,
        selectedCard: SelectedCard,
    ) {
        val userContact = paymentParams.payer?.let { payer ->
            UserContact(
                fullName = payer.fullName,
                email = payer.email,
                phoneNum = payer.phoneNum
            )
        }
        useCases.prepareNativePayment(
            orderIds = paymentParams.getPrePaidOrders(),
            paymentMethod = paymentMethod,
            userContact = userContact,
            selectedCard = selectedCard,
            creditLimitLengthInMonths = creditLimitLengthInMonths,
            cashierOrderParams = CashierOrderParams(
                paymentMethod = paymentParams.paymentMethod?.name,
                paymentSubMethod = paymentParams.paymentSubMethod.name,
                spendCashback = paymentParams.cashback,
                cashierOrderDetails = getCashierOrderDetails(paymentParams.orders)
            ),
        ).schedule(
            onSuccess = { nativePayment ->
                val isAsyncPayment = nativePayment.isAsyncPayment

                when {
                    isAsyncPayment -> {
                        launchAsyncPayment(
                            payment = nativePayment,
                            paymentParams = paymentParams,
                            paymentMethod = paymentMethod,
                            sourceScreen = sourceScreen,
                            redirectStrategy = redirectStrategy
                        )
                    }

                    else -> {
                        launchNativePayment(
                            payment = nativePayment,
                            paymentParams = paymentParams,
                            paymentMethod = paymentMethod,
                            sourceScreen = sourceScreen,
                            redirectStrategy = redirectStrategy
                        )
                    }
                }
            },
            onError = { error ->
                //logging error
            }
        )
    }
```

3) 

```
/**
 * Компонент, отвечающий за получение адреса от пользователя, синхронизация локального адреса с прочими сессиями пользователя (десктоп)
 * Должен предоставлять ui, легко вызываемый из любого места для получения адреса от пользователя, и api для получение адреса
 */

class UserAddressController constructor(
    private val useCases: HyperlocalEnrichAddressUseCases,
    private val userAddressMapper: AddressToUserAddressMapper,
    private val router: Router
) {

    private var currentCoords: GeoCoordinates? = null
    private var currentAddress: UserAddress? = null

    private var waitForExpandState = true
    private var bottomSheetExpandedEmitter: CompletableEmitter? = null
    private val bottomSheetExpandedWaiter = Completable.create { emitter ->
        bottomSheetExpandedEmitter = emitter
        if (!waitForExpandState) {
            bottomSheetExpandedEmitter?.onComplete()
        }
    }

    fun onAllowLoad() {
        if (waitForExpandState) {
            waitForExpandState = false
            bottomSheetExpandedEmitter?.onComplete()
        }
    }

    fun loadUserAddress() {
        useCases.getAddress()
            .flatMap {
                bottomSheetExpandedWaiter.andThen(Observable.just(it))
            }
            .schedule(
                onNext = { address ->
                    if (address is UserAddress.Exists) {
                        currentCoords = address.coordinates
                        currentAddress = address.userAddress
                        viewState.showInputAddressForm(userAddressMapper.map(address.userAddress))
                    } else {
                        viewState.close()
                    }

                },
                onError = {
                    Timber.e(it)
                    viewState.close()
                }
            )
    }

    private fun openMap() {
        router.navigateTo(
            CheckoutMapTargetScreen(
                CheckoutMapArguments(
                    orderIdsMap = args.orderIdsMap,
                    sourceScreen = router.currentScreen,
                )
            )
        )
        viewState.close()
    }

    fun onCloseClick() {
        router.back()
    }

    fun onChangeAddress() {
        openMap()
    }

    fun onAddressChanged(address: Address) {
        userAddressMapper.map(address).orNull?.let {
            currentAddress = it
        }
    }

    fun onContinueClick() {
        val currentUserAddress = currentAddress
        val currentUserCoords = currentHyperlocalCoordinates
        if (currentUserAddress != null && currentUserCoords != null) {
            useCases
                .setHyperlocalAddress(
                    currentUserCoords,
                    currentUserAddress
                )
                .schedule(
                    onComplete = {
                        useCases.updateAddresses().schedule(onError = Timber::e, doFinally = {
                            screen.close()
                        })
                    },
                    onError = {
                        //logging
                    }
                )
        }
    }
}
```

4) Код изменен и сокращен, т.к. NDA.

```
/**
 * Для проверки гипотез используется концепция экспериментов. 
 * Практически все большие фичи катаются под экспериментом для оценки успешности изменений по метрикам
 * Данный компонент отвечает за корректное применение в приложении экспериментов, розданных с удаленного сервиса и за активацию/деактивацию кода, 
 * находящегося под соответствующим экспериментом 
 * 
 */
public final class ExperimentManager {

    @NonNull
    private final ExperimentRegistry register;
    @NonNull
    private final Context applicationContext;
    @NonNull
    private final ExperimentConfigServiceHolder experimentConfigServiceFactory;

    public ExperimentManager(
            @NonNull final Context applicationContext,
            @NonNull final ExperimentRegistry register,
            @NonNull final ExperimentConfigServiceHolder experimentConfigServiceFactory
    ) {
        this.register = Preconditions.checkNotNull(register);
        this.applicationContext = Preconditions.checkNotNull(applicationContext);
        this.experimentConfigServiceFactory =
                Preconditions.checkNotNull(experimentConfigServiceFactory);
    }

    @NonNull
    private <T extends ExperimentSplit> T getExperiment(
            @NonNull final Context context,
            @NonNull final Class<? extends T> className) {
                //logic of getting experiments from server
    }

    @NonNull
    public <T extends ExperimentSplit> T getExperiment(@NonNull final Class<? extends T> className) {
        return getExperiment(applicationContext, className);
    }

    @NonNull
    private List<AliasRearrAssociation> getFlags() {
        return experimentConfigServiceFactory.getExperimentConfigServiceInstance().getExperimentFlags();
    }
}
```


P.S. Отмечу, что код подвергался форматированию, дабы не нарушать NDA.


Выводы.
По истечении лет могу полностью согласиться с антипаттерном "самодокументирующегося" кода. 
Изначально, веря этой идеи, думал я, что это мои способности не столь велики, если не могу сходу понять всю систему, ведь, как заявлено, код написан по всем правилам Клина и хорошо читаем. 
Читаем, может, и хорошо, но понимание от этого не улучшается.

В ходе выполнения задания, когда приходилось осматривать компоненты нашего приложения, которые не писал сам, стало проще рассуждать о приложении как о большой, понятной системе.
Становится более понятно, какое место каждый компонент занимает в программной системе и общее понимание всего приложение становится более ясным.
Известно, что программисты привыкли говорить о частях кода не в контексте, как оно должно работать, а как они сами запрограммировали эти части кода. 
И в таком положении закомментированный класс, описывающий его как часть предметной области, позволяет и быстрее погружаться в код, и поддерживать логику программы более корректной.
Кроме этого, появились мысли, что класс presenter как часть архитектуры MVP не следует принципу single responsibility, в виду того, что туда помещается вся "бизнес-логика".
Лучшим вариантом будет создание собственной архитектуры для конкретного приложения с разделением ответственности между классами, документированным описанием, и как следствие, логичным дизайном.


