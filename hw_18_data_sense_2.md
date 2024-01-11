<h3>Пример №1 с элементами списка как UI элемента.</h3> 

**Было:**
```kotlin
// Изменяемый список элементов в адаптере RecyclerView.
var items: MutableList<Item> = mutableListOf()

fun setItems(newItems: List<Item>) {
    items.clear()
    items.addAll(newItems)
    notifyDataSetChanged()
}
```

**Стало**
```kotlin
// Иммутабельный список для оптимизации работы со списками.
val items: List<Item> get() = _items
private var _items: List<Item> = listOf()

fun setItems(newItems: List<Item>) {
    _items = newItems
    notifyDataSetChanged()
}
```

**Комментарий:**
Использование мутабельного списка (как структуры данных) в Android списке (как UI элемент) создает ряд проблем. 
Во-первых, зачастую трубется по клику по элементу списка модифицировать значение элемента. В случае мутабельного списка такая модификация является потенциально опасной, так как массив подается в Android список извне, соответственно источник будет работать тоже с модифицированным значением.

Использование иммутабельного списка улучшает контроль над состоянием списка. Это снижает вероятность несогласованного состояния при работе адаптера с данными, упрощает тестирование и поддержку кода.

<h3>Пример №2</h3>

**Было**

В MVI архитектуре изменения состояния происходят реактивно, но использование изменяемых объектов может привести к трудноотлавливаемым багам, когда разные части системы по ошибке мутируют общее состояние.
Пример показывает общее повышение надежности участка кода.

```kotlin
// Класс состояния, содержащий изменяемые поля.
data class ProfileViewState(
    var isLoading: Boolean = false,
    var profile: UserProfile? = null,
    var errorMessage: String? = null
)

// Презентер, который обрабатывает изменения состояния.
class ProfilePresenter {
    val viewState: MutableLiveData<ProfileViewState> = MutableLiveData(ProfileViewState())

    fun loadProfile(userId: String) {
        viewState.value?.isLoading = true
        repository.getUserProfile(userId, onSuccess = { profile ->
            viewState.value?.apply {
                isLoading = false
                this.profile = profile
            }
        }, onError = { error ->
            viewState.value?.apply {
                isLoading = false
                errorMessage = error.message
            }
        })
        viewState.value = viewState.value // Триггер обновления LiveData
    }
}

// View слой, который подписан на изменения состояние.
profilePresenter.viewState.observe(this, Observer { state ->
    progressBar.visible = state.isLoading
    state.profile?.let { profile ->
        // отображение информации пользователя...
    }
    state.errorMessage?.let { errorMessage ->
        // показ ошибки...
    }
})
```

**Стало**

```kotlin
// Иммутабельный класс состояния.
data class ProfileViewState(
    val isLoading: Boolean = false,
    val profile: UserProfile? = null,
    val errorMessage: String? = null
)

// Презентер теперь обновляет состояние, создавая новые экземпляры.
class ProfilePresenter {
    val viewState: MutableLiveData<ProfileViewState> = MutableLiveData(ProfileViewState())

    fun loadProfile(userId: String) {
        updateState { it.copy(isLoading = true) }

        repository.getUserProfile(userId, onSuccess = { profile ->
            updateState {
                it.copy(isLoading = false, profile = profile, errorMessage = null)
            }
        }, onError = { error ->
            updateState {
                it.copy(isLoading = false, errorMessage = error.message)
            }
        })
    }

    private fun updateState(update: (ProfileViewState) -> ProfileViewState) {
        val currentState = viewState.value ?: ProfileViewState()
        viewState.value = update(currentState)
    }
}

// View слой получает чётко определённые обновления состояния.
profilePresenter.viewState.observe(this, Observer { state ->
    progressBar.visible = state.isLoading
    state.profile?.let { profile ->
    }
    state.errorMessage?.let { errorMessage ->
    }
})
```
**Комментарий:**
Этот код делает состояние неизменяемым, управление им происходит через явное создание новых экземпляров ProfileViewState. Любое изменение состояния теперь ясно отражено в коде, и ошибки связанные с мутацией данных между потоками исполнения становятся менее вероятны.

<h3>Пример №3</h3>
Изучаю Redux-архитектуру, следующий пример больше связан с правильной реализацией одной из частей архитектуры Redux. 

**Было:**

```kotlin
data class AppState(
    val user: User,
)

data class User(
    var username: String,   // Изменяемое поле
    var email: String       // Изменяемое поле
)

class UserReducer {
    fun reduce(state: AppState, action: Action): AppState {
        when (action) {
            is UserAction.UpdateUsername -> {
                state.user.username = action.newUsername
            }
            is UserAction.UpdateEmail -> {
                state.user.email = action.newEmail
            }
        }
        return state
    }
}

// В каком-то месте в коде, где происходит диспатчинг действий.
store.dispatch(UserAction.UpdateUsername("newUsername"))
store.dispatch(UserAction.UpdateEmail("newEmail@domain.com"))
```
В этом примере мы изменяем поля AppState напрямую, при этом теряем важное свойство Redux - чистота редьюсеров. Они больше не являются чистыми функциями.

Проблемы, которые это создает:
- В Redux все редьюсеры должны быть чистыми функциями; изменение входящих параметров (таких как AppState) приводит к неопределенному поведению. 
- При многопоточной обработке существует риск состояния гонки.

**Стало:**

```kotlin
data class AppState(
    val user: User,
    // другие поля состояния
)

data class User(
    val username: String,   // Неизменяемое поле
    val email: String       // Неизменяемое поле
)

class UserReducer {
    fun reduce(state: AppState, action: Action): AppState = when (action) {
        is UserAction.UpdateUsername -> {
            state.copy(user = state.user.copy(username = action.newUsername))
        }
        is UserAction.UpdateEmail -> {
            state.copy(user = state.user.copy(email = action.newEmail))
        }
        else -> state
    }
}

store.dispatch(UserAction.UpdateUsername("newUsername"))
store.dispatch(UserAction.UpdateEmail("newEmail@domain.com"))
```

**Комментарий:**

- Каждый редьюсер остается чистой функцией, гарантируя предсказуемость и упрощение отладки. 
- История изменений состояния четко отслеживается благодаря неизменяемым объектам, улучшая возможности Undo/Redo. 
- Исключается состояние гонки, поскольку каждое действие работает с собственной копией состояния.

**Выводы:**

Из лекции получил новый взгляд на восприятие сущностей в программной системе на примере квестов и локации. В ближайшее время стартует разработка проекта, на котором очень желательно правильно спроектировать масштабируемое решение с онбордингом. Сущности, которые планируются - это конкретная страница во флоу онбординга, ее позиция и влияние на общий шаг онбординга. Расчитываю, что опыт этого занятия помогут сделать крутое решение.

В практике сталкивался с ситуациями, когда передача по ссылке объектов упрощает работу с такой структурой и в принципе является естественной. Например, ситуация, когда с бекенда приходит набор данных для редактирования, которые пользователь на клиенте редактирует и далее на сервер отправляется такой же набор полей с уже измененнными значениями.
Но так создаются дополнительные проблемы. Например, при пересоздании экрана может такая структура может быть в неконсистентном состоянии. Возможно, часть данных уже была изменена, а нам требуется знать значения, пришедшие с сервера. Так как мы не работали с копией объекта, может столкнуться с такими трудноуловимыми проблемами.

Хорошо, что сам язык Kotlin наставляет использовать иммутабельные переменные и структуры данных, явно разделяя, например, списки на мутабельные (для чтения и записи) и на иммутабельные (только для чтения). Субъективно ощущается, что по сравнению с разработкой на Java рассуждения о программе и ее отладка в контексте значений переменных стало легче. Например, в Java приходилось задумываться, могла ли переменная принять такое-то значение, поскольку по умолчанию переменные мутабельны, а использовать неизменяемые переменные с модификатором `final` считалось признаком опытного разработчика . 