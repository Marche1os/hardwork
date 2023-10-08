**1. Было**

```
@Serializable
data class OrderDto(

    @SerialName("order_id")
    val orderId: String,

    @SerialName("pack_id")
    val packId: String,

    @SerialName("cart_id")
    val cartId: String,

    @SerialName("item_ids")
    val itemIds: List<String>,

    @SerialName("delivery_date")
    val deliveryDate: String,

    @SerialName("total_cost")
    val totalCost: String,
) {

    fun OrderDto.toDomain(): OrderDomain {
        return OrderDomain(
            orderId = orderId,
            packId = packId,
            cartId = cartId,
            itemIds = itemIds,
            deliveryDate = deliveryDate,
            totalCost = totalCost,
        )
    }
}

data class OrderDomain(
    val orderId: String,
    val packId: String,
    val cartId: String,
    val itemIds: List<String>,
    val deliveryDate: String,
    val totalCost: String,
)
```

**Стало**

```
interface Mapper<Input, Output> {

    fun transform(data: Input): Output
}

class OrderDtoToDomainMapper : Mapper<OrderDto, OrderDomain> {

    override fun transform(data: OrderDto): OrderDomain {
        with(data) {
            return OrderDomain(
                orderId = orderId,
                packId = packId,
                cartId = cartId,
                itemIds = itemIds,
                deliveryDate = deliveryDate,
                totalCost = totalCost,
            )
        }
    }
}
```

**2. Было**
   
```
class CurrencyFragment : Fragment(), CoroutineScope {
    override val coroutineContext: CoroutineContext = Dispatchers.Main

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.fragment_currency, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val repository = CorrencyRepository()

        when (arguments?.getInt("key") ?: "") {
            1 -> launch {
                val curency = repository.getCurrency().await()

                loader.visibility=View.GONE
                textAndImage.visibility=View.VISIBLE
                currencyRate.visibility=View.VISIBLE

                curency?.let {
                    currencyText.text = getString(R.string.currenyFragmentTitleUsd)
                    currencyImage.setImageResource(R.drawable.usa)
                    currencyRate.text = getString(R.string.currency).format(it.rub.toString())
                }

            }
            2 ->
                launch {
                    val curency = repository.getCurrency().await()

                    loader.visibility=View.GONE
                    textAndImage.visibility=View.VISIBLE
                    currencyRate.visibility=View.VISIBLE

                    //форматирование евро в формат 1,11
                    val euro = ((1.0 - curency!!.eur) + 1) * curency.rub
                    val df = DecimalFormat("#.##")
                    df.roundingMode = RoundingMode.CEILING

                    curency?.let {
                        currencyText.text = getString(R.string.currenyFragmentTitleEUR)
                        currencyImage.setImageResource(R.drawable.eu_flag)
                        currencyRate.text =
                            getString(R.string.currency).format(df.format(euro).toString())
                    }

                }
        }


    }
}
```

**Стало**
```
class CurrencyFragment : Fragment(), CoroutineScope {
    override val coroutineContext: CoroutineContext = Dispatchers.Main
    private val currencyViewModel: CurrencyViewModel by viewModel()

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.fragment_currency, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val newState = currencyViewModel.state.collectAsState()
        when(newState) {
            is State.Loading -> showLoading()
            
            is State.Error -> showError(newState.errorMessage)
            
            is State.Success -> showContent(newState.rates)
        }
    }

    // показ контента
}

class CurrencyViewModel(
    val currencyInteractor: CurrencyInteractor
) : ViewModel() {
    val state: MutableStateFlow<State> = MutableStateFlow(State.Loading)

    init {
        loadData()
    }

    fun loadData() {
        val exceptionHandler = CoroutineExceptionHandler { _, throwable ->
            state.tryEmit(State.Error(throwable.message ?: ""))
        }

        viewModelScope.launch(exceptionHandler) {
            val rates = currencyInteractor.getCurrency()
            state.tryEmit(State.Success(rates))
        }
    }
}

sealed class State {
    object Loading : State()

    data class Success(val rates: Rates) : State()

    data class Error(val errorMessage: String) : State()
}

class CurrencyInteractor(
    private val currencyRepository: CorrencyRepository,
) {

    suspend fun getCurrency(): Rates {
        return currencyRepository.getCurrency()
    }
}

class CorrencyRepository : CoroutineScope {
    override val coroutineContext: CoroutineContext = Dispatchers.IO

    private val currencyApi = Retrofit.Builder()
        .baseUrl("https://api.exchangerate-api.com")
        .addConverterFactory(GsonConverterFactory.create())
        .build()
        .create(CurrencyApi::class.java)


    suspend fun getCurrency(): Currency {
        return currencyApi.getCurrency().await()
    }
}
```
