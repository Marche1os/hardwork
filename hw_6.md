**Первый случай. Фильтры в маркетплейсе**
"Нестинное" наследование предполагает означает не расширение поведения класса/метода, а полное его переопределение с потерей функциональности класса-родителя. На последнем рабочем месте сталкивался с несколькими случаями такой реализации, нарушающей принцип подстановки Барбары Лисков.

Рассмотрим выдержку из реального кода на моем последнем рабочем проекте. Был суперкласс фильтров (фильтр - реальный фильтр в поиске) с множеством наследников и некой базовой логикой. 

```
abstract class Filter {
    abstract fun applyFilter(products: List<Product>): List<Product>

    open fun toHumanReadable(): String {
        //некоторое поведение в родительском классе, которое некоторые наследники расширяли, а некоторые переопределяли.
        return "Filter"
    }
}

class BrandFilter(private val brands: List<String>) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> brands.contains(product.brand) }
    }

    override fun toHumanReadable(): String {
        return "Brand Filter"
    }
}

class PriceRangeFilter(private val minPrice: Double, private val maxPrice: Double) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> product.price in minPrice..maxPrice }
    }

    override fun toHumanReadable(): String {
        return "Price Range Filter"
    }
}

class CategoryFilter(private val category: String) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> product.category == category }
    }

    override fun toHumanReadable(): String {
        return "Category Filter"
    }
}

class ColorFilter(private val colors: List<String>) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> colors.contains(product.colorHex) }
    }

    override fun toHumanReadable(): String {
        return "Color Filter"
    }
}

class RatingFilter(private val minRating: Double) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> product.rating >= minRating }
    }

    override fun toHumanReadable(): String {
        return "Rating Filter"
    }
}


data class Product(
    val brand: String,
    val price: Double,
    val category: String,
    val colorHex: String,
    val rating: Int,
)
```


**Использовал паттерн Visitor. Так, клиентская логика переместилась в клиентский класс, а классы фильтров просто поддерживают паттерн.**

```
abstract class Filter {
    abstract fun applyFilter(products: List<Product>): List<Product>

    abstract fun toHumanReadable(visitor: Visitor): String
}

class BrandFilter(private val brands: List<String>) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> brands.contains(product.brand) }
    }

    override fun toHumanReadable(visitor: Visitor): String {
        return visitor.toHumanReadableBrandFilter(this)
    }
}

class PriceRangeFilter(private val minPrice: Double, private val maxPrice: Double) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> product.price in minPrice..maxPrice }
    }

    override fun toHumanReadable(visitor: Visitor): String {
        return visitor.toHumanReadablePriceFilter(this)
    }
}

class CategoryFilter(private val category: String) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> product.category == category }
    }

    override fun toHumanReadable(visitor: Visitor): String {
        return visitor.toHumanReadableCategoryFilter(this)
    }
}

class ColorFilter(private val colors: List<String>) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> colors.contains(product.colorHex) }
    }

    override fun toHumanReadable(visitor: Visitor): String {
        return visitor.toHumanReadableColorFilter(this)
    }
}

class RatingFilter(private val minRating: Double) : Filter() {
    override fun applyFilter(products: List<Product>): List<Product> {
        return products.filter { product -> product.rating >= minRating }
    }

    override fun toHumanReadable(visitor: Visitor): String {
        return visitor.toHumanReadableRatingFilter(this)
    }
}

data class Product(
    val brand: String,
    val price: Double,
    val category: String,
    val colorHex: String,
    val rating: Int,
)

class FilterVisitor : Visitor {
    override fun toHumanReadableCategoryFilter(filter: CategoryFilter): String {
        return "Выбрать категорию"
    }

    override fun toHumanReadableBrandFilter(filter: BrandFilter): String {
        return "Выбрать производителя"
    }

    override fun toHumanReadableColorFilter(filter: ColorFilter): String {
        return "Выбрать цвет"
    }

    override fun toHumanReadablePriceFilter(filter: PriceRangeFilter): String {
        return "Указать ценовой диапазон"
    }

    override fun toHumanReadableRatingFilter(filter: RatingFilter): String {
        return "Указать рейтинг"
    }

    fun export(vararg filters: Filter): List<String> {
        return filters.fold(listOf()) { acc, filter: Filter ->
            acc + filter.toHumanReadable(this)
        }
    }
}

class ViewFragment : Fragment() {
    fun logFilter(vararg filter: Filter) {
        val visitor = FilterVisitor()
        val filterNames = visitor.export(
            *filter
        )

        view.showFilters(filterNames)
    }
}

interface Visitor {
    fun toHumanReadableCategoryFilter(filter: CategoryFilter): String

    fun toHumanReadableBrandFilter(filter: BrandFilter): String

    fun toHumanReadableColorFilter(filter: ColorFilter): String

    fun toHumanReadablePriceFilter(filter: PriceRangeFilter): String

    fun toHumanReadableRatingFilter(filter: RatingFilter): String
}
```

Плюс, в реальном проекте было множественное instanceOf в клиентском классе. Отказ от сравнения типов в пользу паттерна Посетитель также на уровне гипотезы должен улучшить производительность. Понятно, реальный код несколько отличался, старался лишь передать концепцию.


**Второй момент.**

Был код, реализующий переключатель экранов. По контракту, необходимо наследоваться от класса Android SDK. Но был переопределен метод destroyItem, не был вызван метод суперкласса, следовательно наследования поведения не приозошло. При этом, согласно логике переопределенного метода, код не переопределял поведение, а использовал метод как callback для метрики. Так был нарушен принцип Лисков, что класс-наследник ломает поведение класса-родителя. В данном случае это приводило к багу с черным экраном.

```
class GameStateAdapter(f: Fragment): FragmentStatePagerAdapter(f.childFragmentManager) {
    override fun getCount(): Int {
        return pages.count
    }

    override fun getItem(position: Int): Fragment {
        return if (position == 0) {
            OnBoardingFragment()
        } else {
            PageContentFragment()
        }
    }

    override fun destroyItem(container: View, position: Int, `object`: Any) {
        // логика суперкласса переопределена. Отсутствует наследование поведения, из-за чего экран при пересоздании становился черным.
    }
}
```

Что стало лучше: 
- Убрали размазывание по иерархии классов логику, относящуюся к UI-части;
- Исправили нарушение принципа подстановки классов;
- Такая правка должна быть рассказана на команду, что должно привести к стандатризации такого подхода с использованием паттерна, когда он необходим. И научить команду видеть применение паттерна.
- Как следует из предыдущего, внедрение стандарта, что класс-наследник должен расширять суперкласс (вызовом super), соответственно проектировать классы, особенно открытые к расширению, с контрактом, что реализация в таком класса должна быть соответственно общей. Общая надежность иерархических структур должна увеличиться, т.к. есть контракт, которому надо следовать.

Что стало хуже:
- Как всегда, добавление абстракции может принести боль для погружения в проект новичками;
- менее опытным программистом тяжелее дается понимание кода на более высоком уровне абстракции. Небходимо знать паттерн и иметь опыт работы с ним;
- Из одной только поддержки паттерна "Посетитель" на уровне класса не очевидны все "завязки" на класс и его сценарии использования.

**Выводы.**
Прежде всего, познакомился с научным взглядом на паттерны, их истинное предназначение с т.з. ООП. Также в моих глазах повысилась значимость этих паттернов, так как на почти всех статьях/книгах было сказано, что паттерны появились как бы "сами собой", когда программисты обнаружили, что решают проблемы, которые уже были решены. Но оказалось, это результат научной работы, что очень круто! Задумался о существовании "неистинного" наследования, в контексте отсутствия вызова super у родительского класса такое наследование становится очевидным. Если принять, что есть контракт на обязательный вызов родительского метода, а если нет - то расширять поведение с помощью Visitor, это может повысить надежность работы с этим классом. Также отмечу, что паттерн Visitor один из немногих, до которых самого тяжело дойти. Так, например, Singleton или Factory рано или поздно жизнь сама заставляет использовать, и как-то к этому приходят сами. Также есть в Android паттерн Adapter, который соединяет модель данных и модель View, для отображения элементов списка. Если начать описывать этот паттерн андроид-разработчику, вероятно - он ничего не поймет. Но даже сам не задумывается, что каждый раз, реализуя список, он реализует этот паттерн. Поэтому соглашусь, что иногда некоторые паттерны просто есть и знать их теоритическую базу необязательно для вполне успешного применения.