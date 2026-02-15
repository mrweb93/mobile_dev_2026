# Лабораторная работа №12
## Выполнение длительных операций (симуляция загрузки) с использованием viewModelScope

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться выполнять длительные операции в фоновом потоке с использованием корутин и `viewModelScope`, управлять состоянием загрузки в UI, реализовать имитацию загрузки данных и обработку ошибок.

---

## 1. Теоретическая справка

### 1.1. Длительные операции и UI
Любые длительные операции (сетевые запросы, работа с базой данных, сложные вычисления) не должны выполняться в главном потоке (UI-потоке), так как это приводит к зависанию интерфейса. В Android для асинхронной работы рекомендуется использовать корутины Kotlin.

### 1.2. viewModelScope
`viewModelScope` — это встроенная область корутин, привязанная к жизненному циклу ViewModel. Она автоматически отменяет все запущенные в ней корутины при уничтожении ViewModel. Это предотвращает утечки памяти и ненужную работу.

Пример запуска корутины:
```kotlin
viewModelScope.launch {
    // длительная операция
}
```

### 1.3. Управление состоянием загрузки
Для отображения прогресса загрузки или состояния ошибки в UI используется подход с хранением состояния в ViewModel (например, с помощью `StateFlow` или `LiveData`). Обычно создают sealed class, описывающий все возможные состояния экрана:

```kotlin
sealed class UiState {
    object Loading : UiState()
    data class Success(val data: List<TaskEntity>) : UiState()
    data class Error(val message: String) : UiState()
}
```

В Compose можно реагировать на изменения состояния и отображать соответствующий UI. В XML-разметке можно использовать `ViewSwitcher`, `ProgressBar` и другие элементы.

### 1.4. Симуляция задержки
Для имитации длительной операции (например, сетевого запроса) используем функцию `delay()`:

```kotlin
viewModelScope.launch {
    _uiState.value = UiState.Loading
    delay(2000) // имитация загрузки
    val data = repository.getAllTasks().first() // получаем данные
    _uiState.value = UiState.Success(data)
}
```

### 1.5. Обработка ошибок
Корутины позволяют обрабатывать исключения с помощью `try-catch`:

```kotlin
viewModelScope.launch {
    try {
        // ...
    } catch (e: Exception) {
        _uiState.value = UiState.Error(e.message ?: "Unknown error")
    }
}
```

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом `TodoApp`, который был разработан в лабораторной работе №11 (с Repository, Room, ViewModel).
- Эмулятор или реальное устройство.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `TodoApp`, созданный в лабораторной работе №11. Убедитесь, что проект компилируется и работает корректно: задачи загружаются из БД, добавляются, удаляются.

### Этап 2. Создание sealed class для состояний (10 мин)

Создайте новый файл `UiState.kt` в пакете `com.example.todoapp.ui` (или рядом с ViewModel):

```kotlin
package com.example.todoapp.ui

import com.example.todoapp.database.TaskEntity

sealed class TasksUiState {
    object Loading : TasksUiState()
    data class Success(val tasks: List<TaskEntity>) : TasksUiState()
    data class Error(val message: String) : TasksUiState()
}
```

### Этап 3. Модификация ViewModel для использования состояний (20 мин)

Измените `MainViewModel`, чтобы он хранил состояние экрана (`TasksUiState`) вместо прямого списка задач. Также добавьте методы для загрузки данных и симуляции длительной операции.

```kotlin
package com.example.todoapp.ui.theme

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.todoapp.data.repository.TaskRepository
import com.example.todoapp.database.TaskEntity
import com.example.todoapp.ui.TasksUiState
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

class MainViewModel(
    private val repository: TaskRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<TasksUiState>(TasksUiState.Loading)
    val uiState: StateFlow<TasksUiState> = _uiState.asStateFlow()

    init {
        loadTasks()
    }

    fun loadTasks() {
        viewModelScope.launch {
            _uiState.value = TasksUiState.Loading
            try {
                // Имитация длительной загрузки (например, сетевой запрос)
                delay(2000) // 2 секунды
                
                // Получаем данные из репозитория
                val tasks = repository.getAllTasks()
                // Так как getAllTasks() возвращает Flow, нужно собрать первый элемент
                // Для простоты предположим, что у нас есть suspend метод в репозитории
                // Но пока оставим как есть, но в репозитории должен быть метод getTasks() suspend?
                
                // На самом деле, чтобы получить текущий список из Flow, можно использовать .first()
                // Но мы пока не будем усложнять: добавим в репозиторий suspend метод
                // Для этой лабораторной изменим репозиторий, добавив suspend fun getTasksOnce()
                
                // Пока упростим: будем использовать имеющийся Flow, но для этого нужно собрать первый элемент
                // Лучше добавить отдельный метод в репозиторий
                // Временно используем репозиторий напрямую
            } catch (e: Exception) {
                _uiState.value = TasksUiState.Error(e.message ?: "Ошибка загрузки")
            }
        }
    }

    fun addTask(title: String) {
        viewModelScope.launch {
            repository.addTask(title)
            // После добавления можно перезагрузить список или оптимистично обновить
            loadTasks()
        }
    }

    fun deleteTask(task: TaskEntity) {
        viewModelScope.launch {
            repository.deleteTask(task)
            loadTasks()
        }
    }

    fun toggleTaskCompletion(task: TaskEntity, isCompleted: Boolean) {
        viewModelScope.launch {
            repository.toggleTaskCompletion(task, isCompleted)
            loadTasks()
        }
    }

    fun refresh() {
        loadTasks()
    }
}
```

**Важно:** В коде выше мы использовали `loadTasks()` после каждой операции, что неэффективно, но для учебных целей допустимо. В реальном проекте лучше обновлять список через Flow из репозитория, но мы сейчас имитируем загрузку, поэтому перезагружаем.

### Этап 4. Добавление suspend-метода в репозиторий (10 мин)

Чтобы получить список задач однократно (для состояния Success), добавим в `TaskRepository` и `TaskRepositoryImpl` новый метод `suspend fun getTasksOnce(): List<TaskEntity>`.

В `TaskRepository.kt`:
```kotlin
interface TaskRepository {
    // ... существующие методы
    suspend fun getTasksOnce(): List<TaskEntity>
}
```

В `TaskRepositoryImpl.kt`:
```kotlin
override suspend fun getTasksOnce(): List<TaskEntity> {
    return taskDao.getAllTasks().first() // first() приостановится до первого элемента
}
```

Не забудьте импортировать `kotlinx.coroutines.flow.first`.

### Этап 5. Обновление ViewModel с использованием getTasksOnce (5 мин)

Замените в `loadTasks()` получение данных:

```kotlin
val tasks = repository.getTasksOnce()
_uiState.value = TasksUiState.Success(tasks)
```

Весь метод теперь:

```kotlin
fun loadTasks() {
    viewModelScope.launch {
        _uiState.value = TasksUiState.Loading
        try {
            delay(2000) // симуляция задержки
            val tasks = repository.getTasksOnce()
            _uiState.value = TasksUiState.Success(tasks)
        } catch (e: Exception) {
            _uiState.value = TasksUiState.Error(e.message ?: "Ошибка загрузки")
        }
    }
}
```

### Этап 6. Обновление UI в MainActivity (20 мин)

Теперь нужно изменить `MainActivity`, чтобы он реагировал на состояние `uiState`. Мы будем использовать подход с отображением разных view в зависимости от состояния.

Добавим в `activity_main.xml` контейнер для переключения между прогресс-баром, списком и сообщением об ошибке. Можно использовать `FrameLayout` или `ViewSwitcher`. Для простоты используем `FrameLayout` и показываем/скрываем view.

Пример разметки (фрагмент):

```xml
<FrameLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewTasks"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:visibility="gone"/>

    <ProgressBar
        android:id="@+id/progressBar"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:visibility="gone"/>

    <TextView
        android:id="@+id/textError"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_gravity="center"
        android:text="Ошибка загрузки"
        android:visibility="gone"/>

</FrameLayout>
```

Также добавьте кнопку "Обновить" (например, в toolbar или отдельную кнопку).

В `MainActivity` подпишитесь на `uiState` и обновляйте видимость элементов и данные адаптера.

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            when (state) {
                is TasksUiState.Loading -> {
                    findViewById<RecyclerView>(R.id.recyclerViewTasks).visibility = View.GONE
                    findViewById<ProgressBar>(R.id.progressBar).visibility = View.VISIBLE
                    findViewById<TextView>(R.id.textError).visibility = View.GONE
                }
                is TasksUiState.Success -> {
                    findViewById<RecyclerView>(R.id.recyclerViewTasks).visibility = View.VISIBLE
                    findViewById<ProgressBar>(R.id.progressBar).visibility = View.GONE
                    findViewById<TextView>(R.id.textError).visibility = View.GONE
                    adapter.updateData(state.tasks)
                }
                is TasksUiState.Error -> {
                    findViewById<RecyclerView>(R.id.recyclerViewTasks).visibility = View.GONE
                    findViewById<ProgressBar>(R.id.progressBar).visibility = View.GONE
                    findViewById<TextView>(R.id.textError).visibility = View.VISIBLE
                    findViewById<TextView>(R.id.textError).text = state.message
                }
            }
        }
    }
}
```

Добавьте обработку кнопки "Обновить":

```kotlin
findViewById<Button>(R.id.buttonRefresh).setOnClickListener {
    viewModel.refresh()
}
```

### Этап 7. Запуск и тестирование (10 мин)

Запустите приложение. При старте вы должны увидеть `ProgressBar` в течение 2 секунд, затем список задач. Попробуйте добавить задачу — снова появится прогресс, затем обновлённый список. Нажмите "Обновить" — тоже прогресс.

Для проверки ошибки можно временно сломать репозиторий (например, выбросить исключение в `getTasksOnce`).

### Этап 8. Дополнительные улучшения (оставшееся время, 10 мин)

- Добавьте обработку ошибок сети: покажите сообщение и кнопку "Повторить".
- Вместо полной перезагрузки списка после добавления задачи, используйте оптимистичное обновление (сразу добавляем в список, но отправляем запрос в БД).
- Добавьте анимацию перехода между состояниями.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Оптимистичное обновление**  
   Измените ViewModel так, чтобы при добавлении задачи она сразу добавлялась в список (без показа загрузки), а фоновая операция выполнялась параллельно. Если операция завершилась ошибкой, откатываем изменение и показываем ошибку.

2. **Pull-to-refresh**  
   Добавьте SwipeRefreshLayout для обновления списка. При свайпе вниз вызывайте `viewModel.refresh()` и показывайте индикатор обновления.

3. **Загрузка с сохранением кэша**  
   Реализуйте стратегию: сначала показываем данные из БД (мгновенно), а затем в фоне обновляем с сервера (с имитацией задержки). После обновления список обновляется. Для этого нужен Flow из БД и отдельная загрузка из сети.

4. **Анимация скелетона**  
   Вместо простого ProgressBar используйте скелетон-эффект (Shimmer). Для XML можно использовать библиотеку Facebook Shimmer, для Compose — встроенные средства.

---

## 5. Контрольные вопросы

1. Почему длительные операции нельзя выполнять в главном потоке?
2. Что такое `viewModelScope` и как он связан с жизненным циклом ViewModel?
3. Какие преимущества даёт использование sealed class для представления состояний UI?
4. Как имитировать задержку в корутине?
5. Как обрабатывать ошибки при выполнении корутин?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинг `UiState.kt`.
- Листинг обновлённого `MainViewModel.kt`.
- Листинг обновлённого `TaskRepository.kt` и `TaskRepositoryImpl.kt` (с новым методом).
- Листинг изменений в `activity_main.xml` и `MainActivity.kt`.
- Скриншоты приложения в состояниях загрузки, успеха и ошибки.
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Ошибка "Flow<T> has no method 'first()' в suspend-функции** – убедитесь, что импортирован `kotlinx.coroutines.flow.first`.
- **ProgressBar не показывается** – проверьте видимость и правильность ID в разметке.
- **После добавления задачи список не обновляется** – убедитесь, что в `addTask` вызывается `loadTasks()` и что `getTasksOnce()` возвращает актуальные данные.
- **Исключение при получении данных** – обработайте `try-catch` и выведите состояние ошибки.

---

**Успешной работы!**
