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

