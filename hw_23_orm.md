Разберем библиотеку Room, которая является официальным ORM решением от google. 

Как работает Room под капотом:

1. Аннотации: В Room, чтобы указать таблицы баз данных, используются аннотации в коде. Пример аннотаций: @Entity, @Dao, @Database и @Query. 

2. Room использует аннотации для генерации дополнительного кода во время компиляции. 

3. Генерация SQL запросов: При определении интерфейсов с аннотациями, такие как @Query, и предоставлении SQL запросов к ним, Room генерирует необходимый код для этих операций в базе данных. Например, если есть метод, аннотированный с @Query("SELECT * FROM users WHERE id = :userId"), Room генерирует соответствующий SQL запрос для выполнения этой операции и осуществляем маппинг сущностей в POJO.

4. Преобразование типов: Программа может использовать типы данных, которые не являются нативными для SQLite, так как Room позволяет использовать конвертеры типов, чтобы маппить эти нестандартные типы в известные SQLite типы данных и обратно.

Рабочего проекта под рукой с использованием Room нет, поэтому возьмем учебный проект: приложение для учета персональных финансов. 
Предметная область включает в себя управление финансами пользователя: отслеживание расходов и доходов, категоризация транзакций, анализ финансовых потоков и планирование бюджета. 
Для упрощения, сконцентрируемся на нескольких ключевых функциях и как они могут быть представлены в коде с использованием ORM:

### Функция 1: Добавление новой транзакции

Смысл предметной области: Пользователь хочет добавить новую транзакцию (расход или доход) в свои финансовые записи. Это включает в себя ввод суммы транзакции, выбор категории (например, обязательные расходы, подписки, зарплата и т.д.), указание дат.

```kotlin
@Dao
interface TransactionDao {
    @Insert
    fun insertTransaction(transaction: Transaction)
}
```

### Функция 2: Получение списка транзакций за определенный период

Смысл предметной области: Чтобы анализировать свои расходы и доходы, пользователь хочет видеть список транзакций за выбранный период времени. Это помогает понять, куда уходят деньги и откуда они поступают, для лучшего планирования бюджета.

```kotlin
@Dao
interface TransactionDao {
    @Query("SELECT * FROM transactions WHERE date BETWEEN :startDate AND :endDate")
    fun getTransactionsByPeriod(startTime: Date, endTime: Date): List<Transaction>
}
```

### Функция 3: Суммирование расходов по категориям

Смысл предметной области: Для эффективного бюджетирования и понимания своих основных статей расходов пользователь хочет видеть общую сумму расходов, разбитую по категориям за определенный период.

```kotlin
@Dao
interface TransactionDao {
    @Query("SELECT category, SUM(amount) as total FROM transactions WHERE date BETWEEN :startDate AND :endDate AND type = 'expense' GROUP BY category")
    fun sumExpensesByCategory(startDate: Date, endDate: Date): List<CategorySum>
}
```

### Подправим код так, чтобы он обращался в обход ORM

### Функция 1: Добавление новой транзакции

```kotlin
fun insertTransaction(db: SQLiteDatabase, transaction: Transaction) {
    val values = ContentValues().apply {
        put("amount", transaction.amount)
        put("category", transaction.category)
        put("date", transaction.date.time) 
        put("type", transaction.type)
    }
    db.insert("transactions", null, values)
}
```

### Функция 2: Получение списка транзакций за определенный период

```kotlin
fun getTransactionsByPeriod(db: SQLiteDatabase, startTime: Long, endTime: Long): List<Transaction> {
    val transactions = mutableListOf<Transaction>()
    val cursor = db.rawQuery(
        "SELECT * FROM transactions WHERE date BETWEEN ? AND ?",
        arrayOf(startTime.toString(), endTime.toString())
    )

    with(cursor) {
        while (moveToNext()) {
            val transaction = Transaction(
                amount = getDouble(getColumnIndex("amount")),
                category = getString(getColumnIndex("category")),
                date = Date(getLong(getColumnIndex("date"))),
                type = getString(getColumnIndex("type"))
            )
            transactions.add(transaction)
        }
        close()
    }
    return transactions
}
```

### Функция 3: Суммирование расходов по категориям

```kotlin
fun sumExpensesByCategory(db: SQLiteDatabase, startDate: Long, endDate: Long): Map<String, Double> {
    val categorySums = mutableMapOf<String, Double>()
    val cursor = db.rawQuery(
        "SELECT category, SUM(amount) AS total FROM transactions WHERE date BETWEEN ? AND ? AND type = 'expense' GROUP BY category", 
        arrayOf(startDate.toString(), endDate.toString())
    )
    
    with(cursor) {
        while (moveToNext()) {
            val category = getString(getColumnIndex("category"))
            val total = getDouble(getColumnIndex("total"))
            categorySums[category] = total
        }
        close()
    }
    
    return categorySums
}
```

### Сценарий 1: sumExpensesByCategory

ORM:

```kotlin

@Dao
interface TransactionDao {
    @Query("SELECT category, SUM(amount) as total FROM transactions WHERE date BETWEEN :startDate AND :endDate AND type = 'expense' GROUP BY category")
    fun sumExpensesByCategory(startDate: Long, endDate: Long): List<CategoryExpense>
}

data class CategoryExpense(val category: String, val total: Double)
```

В обход Room:

```kotlin
fun sumExpensesByCategory(db: SQLiteDatabase, startDate: Long, endDate: Long): Map<String, Double> {
    val categorySums = mutableMapOf<String, Double>()
    db.rawQuery("SELECT category, SUM(amount) AS total FROM transactions WHERE date BETWEEN ? AND ? AND type = 'expense' GROUP BY category", arrayOf(startDate.toString(), endDate.toString())).use { cursor ->
        while (cursor.moveToNext()) {
            val category = cursor.getString(0)
            val total = cursor.getDouble(1)
            categorySums[category] = total
        }
    }

    return categorySums
}
```

Что делает: Эта функция агрегирует и суммирует расходы по категориям за определенный период времени. Это позволяет пользователю или системе взглянуть на распределение расходов по различным категориям в заданном временном диапазоне, что может помочь в планировании бюджета и анализе финансового поведения.

Выигрыш в производительности: Прямой SQL-запрос оказывается чуть быстрее, поскольку выполняется меньше абстракций, и запрос обрабатывается напрямую базой данных без дополнительных проверок типов и конвертации данных, которые делает Room. Выигрыш может быть заметнее при работе с большими объемами данных, но для многих Android приложений разница будет минимальной и едва ли заметной для пользователя.

Бенчмарк (приблизительно):

- Room: 14 мс
- В обход Room: 10 мс
Также бывают аноималии, когда даже и прямое обращение к БД выполняется около 50мс.

### Сценарий 2: getTransactionsByPeriod

Room:

```kotlin
@Dao
interface TransactionDao {
    @Query("SELECT * FROM transactions WHERE date BETWEEN :startDate AND :endDate")
    fun getTransactionsByPeriod(startDate: Long, endDate: Long): List<Transaction>
}
```

В обход Room:

```kotlin

fun getTransactionsByPeriod(db: SQLiteDatabase, startDate: Long, endDate: Long): List<Transaction> { 
    val transactions = mutableListOf<Transaction>()
    db.rawQuery("SELECT * FROM transactions WHERE date BETWEEN ? AND ?", arrayOf(startDate.toString(), endDate.toString())).use { cursor ->
        while (cursor.moveToNext()) {
            val transaction = transactions.add(transaction)
        }
    }

    return transactions
}
```

Что делает: Функция извлекает все транзакции в заданном временном диапазоне. Это полезно для отображения истории транзакций пользователя, анализа трат за определенный период или подготовки финансовой отчетности.

Выигрыш в производительности: Аналогично предыдущей функции, прямой SQL-запрос оказывается немного быстрее за счет упрощения процесса выполнения. Однако, преимущества использования Room, такие как удобство, поддержка архитектурных паттернов и безопасность, часто перевешивают незначительную потерю в скорости.

### Сценарий 3: insertTransaction

Room:

```kotlin
@Dao
interface TransactionDao {
    @Insert
    fun insertTransaction(transaction: Transaction)
}
```

В обход Room:

```kotlin
fun insertTransaction(db: SQLiteDatabase, transaction: Transaction) {
    val values = ContentValues().apply {
        put("amount", transaction.amount)
        put("category", transaction.category)
        put("date", transaction.date)
        put("type", transaction.type) 
    }
    
    db.insert("transactions", null, values)
}
```


Что делает: Функция добавляет новую транзакцию в базу данных. Это основное действие для любого финансового или бухгалтерского приложения, позволяющее отслеживать доходы и расходы.

Выигрыш в производительности: Здесь мы снова видим, что прямое взаимодействие с базой данных может быть чуть быстрее, в основном из-за меньших накладных расходов на конвертацию и валидацию данных, которые осуществляются Room.


Бенчмарк (приблизительно):

- Room: 5 мс за транзакцию
- В обход Room: 3 мс за транзакцию



### Выводы 

В каждом из приведенных случаев выигрыш в производительности при использовании прямого обращение к БД существует, 
но он не всегда оправдывает отказ от преимуществ, которые предоставляет Room. 

По специфике, в типичных Android проектах нет такого объема локальных данных, при котором отказ от ORM даст существенную прибавку к скорости работы с базой данных.
По опыту, чаще всего БД выступает в роли кеша с достаточно ограниченным временем жизни.

Хотя я изучал код приложения Telegram, там достаточно активно используется локальное хранилище данных для сообщений, в том числе видео и аудио сообщения, которые могут занимать существенные объемы.
Например, локальные данные телеграмма занимают около 3,5гб. У них ORM не используется, только прямое обращение с БД, прямые sql-запросы. 

По итогам задания можно прийти к заключению, что при разработке такого приложения, как мессенджер, предполагающий большие объемы данных, стоит отказаться от удобств ORM и в целом от лишних абстракций.
По замерам, прямое обращение с БД оказывается быстрее даже на простых запросах. На более сложных составных запросах предполагается более существенный выигрыш в производительности.

Выбор между Room и прямыми SQL-запросами должен базироваться на комплексном анализе требований к приложению, в том числе на требованиях к скорости, удобстве разработки, поддержке и специфике приложения, как рассмотрели выше пример с тг.

