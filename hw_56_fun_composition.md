### Пример 1. Обработка событий пользовательского ввода

```kotlin
open class InputDataValidator {
    open fun validate(input: String): Boolean {
        return input.isNotEmpty()
    }
}

// Класс для валидации email
class EmailValidator : InputDataValidator() {
    override fun validate(input: String): Boolean {
        return input.contains("@") && super.validate(input)
    }
}

class PhoneNumberValidator : InputDataValidator() {
    
    override fun validate(input: String): Boolean {
        TODO("логика проверки номера телефона")
    }
    
}
```

С использованием функциональной композиции:

```kotlin
// Функция для базовой проверки
fun isNotEmpty(input: String): Boolean = input.isNotEmpty()

// Функция для проверки email
fun isEmailValid(input: String): Boolean = isNotEmpty(input) && input.contains("@")

// Функция для проверки номера телефона..
// ...
```

### Пример 2

```kotlin
open class ErrorHandler {
    open fun handleError(errorCode: Int): String {
        return "Сообщение об ошибка"
    }
}

class NetworkErrorHandler : ErrorHandler() {
    override fun handleError(errorCode: Int): String {
        return when (errorCode) {
            404 -> "Not Found"
            500, 
            502 -> "Server Error"
            else -> super.handleError(errorCode)
        }
    }
}
```

С использованием композиции:

```kotlin
val defaultError = { _: Int -> "Ошибка!" }
val networkError = { errorCode: Int ->
    when (errorCode) {
        404 -> "Not Found"
        500, 502 -> "Server Error"
        else -> defaultError(errorCode)
    }
}
```

### Пример 3

```kotlin
// Базовый класс адаптера
open class BaseAdapter : RecyclerView.Adapter<RecyclerView.ViewHolder>() {
    open fun bindData(data: Any) {}
}

class TextAdapter : BaseAdapter() {
    override fun bindData(data: Any) {
        // Обработка специфичной привязки данных
        if (data is String) {
            println(data)
        }
    }
}
```

После функциональной композиции:

```kotlin
// Функция конфигурации адаптера
fun createAdapter(bindData: (Any) -> Unit): RecyclerView.Adapter<RecyclerView.ViewHolder> {
    return object : RecyclerView.Adapter<RecyclerView.ViewHolder>() {
        override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): RecyclerView.ViewHolder {
            // Возвращаем заглушку
            return object : RecyclerView.ViewHolder(View(parent.context)) {}
        }

        override fun onBindViewHolder(holder: RecyclerView.ViewHolder, position: Int) {
            bindData.invoke(position)
        }

        override fun getItemCount(): Int = 10
    }
}
```

### Выводы

В ходе выполнения задания мне удалось погрузиться в концепцию функциональной композиции и практически применить её в проектировании программного обеспечения. В сравнении с классическим наследованием, этот подход продемонстрировал ряд преимуществ, которые помогли оптимизировать разработку.

Использование функциональной композиции позволило избавиться от иерархий классов. 
Вместо жесткой структуры наследования, я смог организовать код как набор независимых модулей/функций, которые можно комбинировать. 
Это снизило связанность компонентов и повысило гибкость системы. 
В итоге, стало легче добавлять и изменять функционал, не затрагивая существующий код.

Примером может служить разделение логики для обработки данных. 
Вместо того чтобы наследовать общие методы в огромной цепочке классов, я создал функции, каждую из которых можно переиспользовать в различных контекстах.

Структурируя, можно выделить следующие преимущества:

- Логика становится более плоской без создания глубоких иерархий классов;
- Поведение настраивается через параметры функций, а не через наследование;
- Функции легко тестировать отдельно от контекста приложения.