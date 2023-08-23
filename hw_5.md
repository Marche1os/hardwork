Выбрал как раз задачу из курса по алгоритмам. Была выбрана задача по реализации стека, позволяющего получать текущее максимальное значение за константное время без негативного эффекта на функции добавления/удаления.

В ходе написания тестов "не думая" получалось тестирование только базовых сценариев в виде добавления/удаления элементов, проверки на пустоту и т.д.

Так как тип задачи - про реализацию алгоритма, то изменения естественным образом получалось делать небольшими порциями: буквально каждую функцию покрывать тестами, потом переходить к следующей. И так до полной реализации, после чего проверять корректную работу, комбинируя вызовы функций.

В ходе инспекции приходят новые замечания по пограничным случаям, как в тестах, так и в коде. В тестах не было обработано исключение и проверка корректности максимального значения после добавлений и удалений элементов. Что касается того, чтобы сделать изменения еще меньше, то конкретно в этой задаче можно было разделить логику добавления элемента в стек на 2 ветви: непосредственно добавление в основной стек (почти атомарная операция), и логику насчета максимального значения во вспомогательном стеке. Потребовалось 3 итерации с удалением этого кода, чтобы реализовать правильно логику. После чего был осуществлен коммит, в который включена функция добавления и тесты к ней.


```
@Test
    void popTest() {
        MKStack<Integer> stack = new MKStack<>(Integer.class);

        assertThrows(NoSuchElementException.class, () -> {
            stack.pop();
        });

        stack.push(10);
        stack.push(15);
        stack.push(13);

        Integer[] maxValues = stack.getMaxValuesStack();
        Integer[] data = stack.getData();

        assertEquals(15, maxValues[2]);
        assertEquals(13, data[2]);

        stack.pop();

        assertNull(maxValues[2]);
        assertNull(data[2]);
        assertEquals(15, maxValues[1]);
        assertEquals(15, data[1]);

        stack.push(20);
        assertEquals(20, maxValues[2]);
        assertEquals(20, data[2]);
    }
```

```
public class MKStack<T extends Comparable<T>> {
    private final T[] data;
    private final T[] maxValuesStack; //contains current max value.
    private int lastIndexPointer;
    private int count;

    private static final int DEFAULT_CAPACITY = 12;

    public MKStack(Class<T> clazz) {
        this(clazz, DEFAULT_CAPACITY);
    }

    public MKStack(Class<T> clazz, int size) {
        data = (T[]) Array.newInstance(clazz, size);
        maxValuesStack = (T[]) Array.newInstance(clazz, size);
        lastIndexPointer = 0;
        count = 0;
    }

    public void push(final T el) {
        if (count == 0) {
            pushOnEmptyStack(el);
            return;
        }

        data[lastIndexPointer] = el;
        final T currentMaxValue = maxValuesStack[lastIndexPointer - 1];
        if (currentMaxValue.compareTo(el) > 0) {
            maxValuesStack[lastIndexPointer] = currentMaxValue;
        } else {
            maxValuesStack[lastIndexPointer] = el;
        }

        lastIndexPointer++;
        count++;
    }

    private void pushOnEmptyStack(final T el) {
        data[lastIndexPointer] = el;
        maxValuesStack[lastIndexPointer] = el;
        lastIndexPointer++;
        count++;
    }

    public T pop() throws NullPointerException {
        if (lastIndexPointer == 0)
            throw new NoSuchElementException("Stack is empty");

        data[lastIndexPointer - 1] = null;
        maxValuesStack[lastIndexPointer - 1] = null;

        count--;
        return data[lastIndexPointer--];
    }

    public boolean isEmpty() {
        return count == 0;
    }

    public int getCount() {
        return count;
    }

    @VisibleForTestOnly
    public T[] getMaxValuesStack() {
        return maxValuesStack;
    }


    public T[] getData() {
        return data;
    }
}

```

2. 
Следующая задача продуктовая. Нужно обновить номер телефона у пользователя, при этом номер телефона форматируется.

Задача здесь маленькая, но хорошо подходит для подхода. Во-первых, в тестах входные данные перед глазами, во-вторых, их можно постепенно расширять. При этом сама функция форматирования времени примитивная, была написана во время собеседования, но как раз для такого сценария подход с прогоном тестов и корректировкой кода под проходящий тест может неплохо работать.

```
class PhoneFormatterTest {
    @Test
    fun testPhoneNumberFormatting() {
        val phoneNumber = "12345678907"
        val expectedFormattedNumber = "+1 (234) 567 89 07"
        val phoneFormatter = PhoneFormatter()

        assertEquals(expectedFormattedNumber, phoneFormatter.formatPhoneNumber(phoneNumber))
    }

    @Test
    fun testPhoneNumberFormattingWithCountryCode() {
        val phoneNumber = "79123456789"
        val expectedFormattedNumber = "+7 (912) 345 67 89"

        val phoneFormatter = PhoneFormatter()

        assertEquals(expectedFormattedNumber, phoneFormatter.formatPhoneNumber(phoneNumber))
    }

    @Test
    fun testPhoneNumberFormattingWithSpaces() {
        val phoneNumber = "    7 9 1 2 3 4 5 6 7 8 9 "
        val expectedFormattedNumber = "+7 (912) 345 67 89"

        val phoneFormatter = PhoneFormatter()

        assertEquals(expectedFormattedNumber, phoneFormatter.formatPhoneNumber(phoneNumber))
    }

    @Test
    fun testIncorrectPhoneNumberFormat() {
        val phoneNumber = "123"
        val expectedFormattedNumber = null

        val phoneFormatter = PhoneFormatter()

        assertEquals(expectedFormattedNumber, phoneFormatter.formatPhoneNumber(phoneNumber))
    }

    @Test
    fun testEmptyPhoneNumber() {
        val phoneNumber = ""
        val expectedFormattedNumber = null

        val phoneFormatter = PhoneFormatter()

        assertEquals(expectedFormattedNumber, phoneFormatter.formatPhoneNumber(phoneNumber))
    }

    @Test
    fun testPhoneNumberWithoutCountryCode() {
        val phoneNumber = "123456789"
        val expectedFormattedNumber = null

        val phoneFormatter = PhoneFormatter()

        assertEquals(expectedFormattedNumber, phoneFormatter.formatPhoneNumber(phoneNumber))
    }
}
```

```
    fun updatePhone(user: UserData, newPhoneNumber: String): UserData {
        val formattedNumber = phoneFormatter.formatPhoneNumber(newPhoneNumber)
        users.find { user -> id == user.id }
        ?.let { user ->
            val updatedUser = user.copy(
              phoneNumber = formattedNumber
            )  
            users[user] = updatedUser
        }
    }
```

```
    fun formatPhoneNumber(phoneNumber: String): String? {
        val formattedPhoneNumber = phoneNumber.replace(Regex("[^0-9]"), "")

        if (formattedPhoneNumber.length == 10) {
            return "+7 (${formattedPhoneNumber.substring(0, 3)}) ${formattedPhoneNumber.substring(3, 6)} ${formattedPhoneNumber.substring(6, 8)} ${formattedPhoneNumber.substring(8)}"
        } else if (formattedPhoneNumber.length == 11 && formattedPhoneNumber.startsWith("7")) {
            return "+7 (${formattedPhoneNumber.substring(1, 4)}) ${formattedPhoneNumber.substring(4, 7)} ${formattedPhoneNumber.substring(7, 9)} ${formattedPhoneNumber.substring(9)}"
        } else if (formattedPhoneNumber.length == 11 && formattedPhoneNumber.startsWith("1")) {
            return "+1 (${formattedPhoneNumber.substring(1, 4)}) ${formattedPhoneNumber.substring(4, 7)} ${formattedPhoneNumber.substring(7, 9)} ${formattedPhoneNumber.substring(9)}"
        }

        return null
    }
```

Отчет. На задачах, где функция может работать атомарно и независимо, по-сути, от окружения, изменения получаются небольшими. Буквально тестом покрывается каждая функция. Для функции добавления элемента в стек с максимальным элементов потребовалось 3-4 итерации удаления с переписыванием части кода. Задача с форматированием номера делалась в несколько итераций: сначала один шаблон номера, потом другой.  

Успешность применения в работе зависит и от факторов. При разработке некой функциональности с нуля подход будет неплохо работать. Но есть еще эмоциональная составляющая в ситуациях, когда менеджеры просят до пятницы влить задачу, чтобы попасть в недельный релиз. В таком случае, очень хочется броситься делать задачу минимально, чтобы работало. И еще предстоит научиться тому, чтобы не паниковать, а спокойно сесть, провести рефлексию, набросать тесты и написать код. Хотя и от типа задач тоже многое зависит, не для всех такой подход будет работать.

Выводы. Вообще, очень редко, неосознанно практиковал чем-то схожий подход, только это было не через написание а тестов, а для реализации архитектуры некоторого небольшого компонента в приложении: сначала создается каркас в первый раз, после написания архитектура не нравится и сразу все удаляется. И так с примерно 3 итерации создается уже приемлимая архитектура для компонента. Что касается подхода с тестами, думаю, научился держать в фокусе небольшую часть разрабатываемого компонента, которая сразу закрыта тестированием и в случае чего будет удалена. С одной стороны приучает и концентрации, с другой раскрепощает и убирает это желание написать сразу что-то идеально. Так что буду смотреть, когда можно будет применять TDD в новом исполнении