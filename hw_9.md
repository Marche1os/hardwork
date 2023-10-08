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

**3. Было**

```
class MessagesAdapter(
    private val myUsername: String
) : RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    private val messages = mutableListOf<Message>()
    
    
    companion object {
        private val MESSAGE_TYPE = 0
        private val MY_MESSAGE_TYPE = 1
    }

    val lastMessageId get() = messages.lastOrNull()?.id ?: 0

    fun addMessages(newMessages: List<Message>){
        val startIndex = messages.size
        messages.addAll(newMessages)
        notifyItemRangeInserted(startIndex, newMessages.size)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
        return if (viewType == MY_MESSAGE_TYPE) {
            val vMyMessage =
                LayoutInflater.from(parent.context).inflate(R.layout.item_message_my, parent, false)
            MyMessageViewHolder(vMyMessage)
        } else {
            val vMessage =
                LayoutInflater.from(parent.context).inflate(R.layout.item_message, parent, false)
            MessageViewHolder(vMessage)
        }

    }

    override fun getItemCount() = messages.size

    override fun getItemViewType(position: Int): Int {
        return if (messages[position].author == myUsername) {
            MY_MESSAGE_TYPE
        } else {
            MESSAGE_TYPE
        }
    }

    override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
        if (getItemViewType(position) == MY_MESSAGE_TYPE) {
            (holder as MyMessageViewHolder).bind(messages[position])
        } else {
            (holder as MessageViewHolder).bind(messages[position])
        }
    }

    class MessageViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bind(message: Message) = itemView.apply {
            senderName.text = message.author
            messageText.text = message.text
            sendDate.text =
                SimpleDateFormat("HH:mm MMM dd", Locale.getDefault()).format(message.date)
        }
    }

    class MyMessageViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bind(message: Message) = itemView.apply {
            myName.text = message.author
            myMessageText.text = message.text
            MySendDate.text =
                SimpleDateFormat("HH:mm MMM dd", Locale.getDefault()).format(message.date)
        }
    }
}

data class Message(
    val id: Int,
    val author: String,
    val text: String,
    val date: Date
)
```

**Стало**

```
class MessagesAdapter(
    private val myUsername: String
) : RecyclerView.Adapter<RecyclerView.ViewHolder>() {

    //логика без изменений

    class MessageViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bind(message: Message) = itemView.apply {
            senderName.text = message.author
            messageText.text = message.text
            sendDate.text = message.formattedDate
        }
    }

    class MyMessageViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        fun bind(message: Message) = itemView.apply {
            myName.text = message.author
            myMessageText.text = message.text
            MySendDate.text = message.formattedDate
        }
    }
}

data class Message(
    val id: Int,
    val author: String,
    val text: String,
    val formattedDate: String
)
```

**4. Было**
```
object ImageHelper {

    fun cropToSquare(bitmap: Bitmap): Bitmap {
        return if (bitmap.width >= bitmap.height){
            Bitmap.createBitmap(bitmap, bitmap.width / 2 - bitmap.height / 2, 0, bitmap.height, bitmap.height)
        } else{
            Bitmap.createBitmap(bitmap, 0, bitmap.height / 2 - bitmap.width / 2, bitmap.width, bitmap.width)
        }
    }

    fun rotate(bitmap: Bitmap, degree: Float): Bitmap {
        val matrix = Matrix()
        matrix.postRotate(degree)
        return Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
    }

    fun blur(context: Context, bitmap: Bitmap, radius: Float): Bitmap {
        val rs = RenderScript.create(context)
        val input = Allocation.createFromBitmap(rs, bitmap) // use this constructor for best performance, because it uses USAGE_SHARED mode which reuses memory
        val output = Allocation.createTyped(rs, input.type)
        ScriptIntrinsicBlur.create(rs, Element.U8_4(rs)).apply {
            setRadius(radius)
            setInput(input)
            forEach(output)
        }
        output.copyTo(bitmap)
        return bitmap
    } 
}
```

**Стало**

```
interface ImageUtil

class ImageCropper : ImageUtil {

    fun cropToSquare(bitmap: Bitmap): Bitmap {
        return if (bitmap.width >= bitmap.height) {
            cropFromCenter(bitmap, bitmap.height, bitmap.height)
        } else {
            cropFromCenter(bitmap, bitmap.width, bitmap.width)
        }
    }

    private fun cropFromCenter(bitmap: Bitmap, targetWidth: Int, targetHeight: Int): Bitmap {
        val startX = (bitmap.width - targetWidth) / 2
        val startY = (bitmap.height - targetHeight) / 2
        return Bitmap.createBitmap(bitmap, startX, startY, targetWidth, targetHeight)
    }
}

class ImageRotator : ImageUtil {

    fun rotate(bitmap: Bitmap, degree: Float): Bitmap {
        val matrix = Matrix()
        matrix.postRotate(degree)
        return Bitmap.createBitmap(bitmap, 0, 0, bitmap.width, bitmap.height, matrix, true)
    }
}

class ImageBlurrer(private val context: Context) : ImageUtil {

    fun blur(bitmap: Bitmap, radius: Float): Bitmap {
        val rs = RenderScript.create(context)
        val blurredBitmap = bitmap.copy(Bitmap.Config.ARGB_8888, true)
        val input = Allocation.createFromBitmap(rs, blurredBitmap)
        val output = Allocation.createTyped(rs, input.type)
        ScriptIntrinsicBlur.create(rs, Element.U8_4(rs)).apply {
            setRadius(radius)
            setInput(input)
            forEach(output)
        }
        output.copyTo(blurredBitmap)
        return blurredBitmap
    }
}
```
