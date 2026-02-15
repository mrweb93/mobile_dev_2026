# Лабораторная работа №8
## Перенос логики списка задач из Activity в ViewModel. Использование StateFlow для хранения состояния

**Длительность:** 1 час 30 минут  
**Цель работы:** Изучить архитектурный компонент ViewModel, научиться выносить логику и состояние UI из Activity, использовать StateFlow для реактивного обновления данных, обеспечить сохранение состояния при изменении конфигурации.

---

## 1. Теоретическая справка

### 1.1. ViewModel
`ViewModel` – это компонент архитектуры Android, предназначенный для хранения и управления данными, связанными с UI, с учётом жизненного цикла. ViewModel переживает повороты экрана и другие изменения конфигурации, что предотвращает потерю данных.

Преимущества использования ViewModel:
- Разделение ответственности (UI не занимается загрузкой/хранением данных).
- Устойчивость к изменениям конфигурации.
- Упрощение тестирования (логика изолирована от Android-зависимостей).

### 1.2. StateFlow
`StateFlow` – это поток данных из библиотеки Kotlin Coroutines, который всегда хранит последнее значение и уведомляет подписчиков о новых значениях. Является отличной альтернативой `LiveData` в чисто Kotlin-проектах, особенно при использовании корутин.

Особенности:
- Холодный поток превращается в горячий: `StateFlow` всегда активен.
- Имеет текущее значение (`value`).
- Поддерживает корутины и операторы преобразования.

Для создания StateFlow обычно используют `MutableStateFlow` с начальным значением и предоставляют неизменяемый `StateFlow` наружу.

### 1.3. Зависимости
Для использования ViewModel и StateFlow добавьте в `build.gradle` (Module) следующие зависимости:
```gradle
dependencies {
    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'
    implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.7.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
```
Также требуется плагин `kotlin-kapt` для работы с lifecycle (не обязательно для базового использования).

### 1.4. Сбор и подписка в Activity
В Activity для получения ViewModel используется делегат `by viewModels()`. Для подписки на StateFlow в жизненном цикле Activity применяется `lifecycleScope` и `repeatOnLifecycle`.

Пример:
```kotlin
class MainActivity : AppCompatActivity() {
    private val viewModel: MainViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.tasks.collect { tasks ->
                    // обновление адаптера
                }
            }
        }
    }
}
```

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом `TodoApp` (результат выполнения лабораторной работы №7).
- Эмулятор или реальное устройство.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `TodoApp`, который был создан в лабораторной работе №7. Убедитесь, что проект компилируется и работает: список задач отображается, по клику открывается экран деталей.

### Этап 2. Добавление необходимых зависимостей (5 мин)

Откройте файл `app/build.gradle` и убедитесь, что в разделе `dependencies` присутствуют следующие строки (если нет – добавьте и выполните Sync):

```gradle
dependencies {
    // ... другие зависимости

    implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'
    implementation 'androidx.lifecycle:lifecycle-runtime-ktx:2.7.0'
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
```

### Этап 3. Создание ViewModel (15 мин)

Создайте новый класс `MainViewModel` в пакете `com.example.todoapp` (или в отдельном пакете `viewmodel`).

```kotlin
package com.example.todoapp

import androidx.lifecycle.ViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow

class MainViewModel : ViewModel() {

    // Приватный изменяемый StateFlow с начальным значением (пустой список)
    private val _tasks = MutableStateFlow<List<String>>(emptyList())

    // Публичный неизменяемый StateFlow для подписки из UI
    val tasks: StateFlow<List<String>> = _tasks.asStateFlow()

    // Добавление новой задачи
    fun addTask(task: String) {
        val currentList = _tasks.value.toMutableList()
        currentList.add(task)
        _tasks.value = currentList
    }

    // Удаление задачи по индексу
    fun deleteTask(index: Int) {
        val currentList = _tasks.value.toMutableList()
        if (index in currentList.indices) {
            currentList.removeAt(index)
            _tasks.value = currentList
        }
    }

    // Обновление текста задачи
    fun updateTask(index: Int, newText: String) {
        val currentList = _tasks.value.toMutableList()
        if (index in currentList.indices) {
            currentList[index] = newText
            _tasks.value = currentList
        }
    }

    // Вспомогательный метод для инициализации тестовыми данными (если нужно)
    fun loadTestData() {
        _tasks.value = listOf(
            "Купить продукты",
            "Сделать ДЗ по Android",
            "Позвонить маме",
            "Записаться к врачу"
        )
    }
}
```

### Этап 4. Рефакторинг MainActivity (20 мин)

В `MainActivity.kt` удалите объявление `tasks` как `mutableListOf`. Получите ViewModel с помощью делегата `by viewModels()`. Подпишитесь на `viewModel.tasks` и обновляйте адаптер при изменениях. Измените логику добавления задачи – вместо прямого добавления в список вызывайте `viewModel.addTask()`.

Пример итогового кода `MainActivity.kt`:

```kotlin
package com.example.todoapp

import android.content.Intent
import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.Toast
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import kotlinx.coroutines.launch

class MainActivity : AppCompatActivity() {

    private val viewModel: MainViewModel by viewModels()
    private lateinit var adapter: TaskAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val editTextTask = findViewById<EditText>(R.id.editTextTask)
        val buttonAddTask = findViewById<Button>(R.id.buttonAddTask)
        val recyclerView = findViewById<RecyclerView>(R.id.recyclerViewTasks)

        // Настройка RecyclerView
        recyclerView.layoutManager = LinearLayoutManager(this)
        adapter = TaskAdapter(
            tasks = emptyList(), // адаптер будет обновляться через submitList или подобное
            onItemClick = { position ->
                val taskText = viewModel.tasks.value[position]
                val intent = Intent(this, DetailActivity::class.java)
                intent.putExtra("task_text", taskText)
                startActivity(intent)
            },
            onItemLongClick = { position ->
                viewModel.deleteTask(position)
                Toast.makeText(this, "Задача удалена", Toast.LENGTH_SHORT).show()
            }
        )
        recyclerView.adapter = adapter

        // Подписка на изменения списка задач
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.tasks.collect { tasks ->
                    adapter.updateData(tasks) // предполагаем, что у адаптера есть такой метод
                }
            }
        }

        // Добавление задачи
        buttonAddTask.setOnClickListener {
            val task = editTextTask.text.toString()
            if (task.isNotBlank()) {
                viewModel.addTask(task)
                editTextTask.text.clear()
            } else {
                Toast.makeText(this, "Введите задачу", Toast.LENGTH_SHORT).show()
            }
        }

        // Загрузим тестовые данные при первом запуске (если список пуст)
        if (viewModel.tasks.value.isEmpty()) {
            viewModel.loadTestData()
        }
    }
}
```

Обратите внимание: адаптер теперь не хранит список, а получает его извне через метод `updateData`. Нужно добавить этот метод в `TaskAdapter`.

### Этап 5. Модификация TaskAdapter (10 мин)

Измените адаптер так, чтобы он не содержал внутреннего списка, а принимал список через конструктор и обновлялся методом `updateData`. Также добавьте обработку долгого нажатия.

```kotlin
package com.example.todoapp

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.CheckBox
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class TaskAdapter(
    private var tasks: List<String>,
    private val onItemClick: (Int) -> Unit,
    private val onItemLongClick: (Int) -> Unit
) : RecyclerView.Adapter<TaskAdapter.TaskViewHolder>() {

    class TaskViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val textTask: TextView = itemView.findViewById(R.id.textTask)
        val checkTask: CheckBox = itemView.findViewById(R.id.checkTask)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): TaskViewHolder {
        val view = LayoutInflater.from(parent.context)
            .inflate(R.layout.item_task, parent, false)
        return TaskViewHolder(view)
    }

    override fun onBindViewHolder(holder: TaskViewHolder, position: Int) {
        val task = tasks[position]
        holder.textTask.text = task

        holder.itemView.setOnClickListener {
            onItemClick(position)
        }

        holder.itemView.setOnLongClickListener {
            onItemLongClick(position)
            true
        }

        // (Опционально) логика чекбокса из Лаб.6
        holder.checkTask.setOnCheckedChangeListener { _, isChecked ->
            if (isChecked) {
                holder.textTask.paintFlags = holder.textTask.paintFlags or android.graphics.Paint.STRIKE_THRU_TEXT_FLAG
            } else {
                holder.textTask.paintFlags = holder.textTask.paintFlags and android.graphics.Paint.STRIKE_THRU_TEXT_FLAG.inv()
            }
        }
    }

    override fun getItemCount(): Int = tasks.size

    fun updateData(newTasks: List<String>) {
        tasks = newTasks
        notifyDataSetChanged()
    }
}
```

### Этап 6. Обновление DetailActivity (5 мин)

Оставим `DetailActivity` без изменений – она получает текст задачи через Intent и отображает его. Для удаления с экрана деталей можно было бы использовать result API, но в рамках этой лабораторной мы реализуем удаление по долгому нажатию в списке, что уже добавлено.

### Этап 7. Запуск и тестирование (10 мин)

Запустите приложение. Проверьте:
- При старте отображаются тестовые задачи.
- Добавление новой задачи работает и список обновляется.
- Удаление задачи по долгому нажатию работает (появляется Toast, задача исчезает).
- При повороте экрана список задач сохраняется (ViewModel переживает поворот).
- Клик по задаче открывает экран деталей с правильным текстом.

### Этап 8. Дополнительные улучшения (оставшееся время, 10 мин)

1. **Использование sealed class для состояния**  
   Создайте класс `TasksState` (Loading, Success, Error). Вместо `List<String>` используйте `StateFlow<TasksState>`. При загрузке тестовых данных добавьте имитацию загрузки.

2. **SharedFlow для событий**  
   Вместо прямого вызова Toast в Activity, можно использовать `SharedFlow` для отправки одноразовых событий (например, сообщение об ошибке). Создайте `MutableSharedFlow<String>` в ViewModel и подпишитесь на него в Activity.

3. **Редактирование задачи**  
   Добавьте возможность редактирования задачи через второй экран: передавайте позицию и текущий текст, а после редактирования возвращайте результат через `ActivityResultLauncher` и обновляйте задачу через `viewModel.updateTask()`.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Sealed class для состояния загрузки**  
   Реализуйте состояние `TaskState` (Loading, Success(List<String>), Error(String)). В ViewModel при "загрузке" тестовых данных добавьте задержку (например, через `delay` в корутине) и эмитируйте состояния. В Activity отображайте прогресс-бар или сообщение об ошибке.

2. **SharedFlow для уведомлений**  
   Добавьте в ViewModel `MutableSharedFlow<String>` для отправки сообщений (например, "Задача добавлена", "Ошибка: пустая задача"). В Activity подпишитесь на этот поток и показывайте Toast или Snackbar.

3. **Редактирование с возвратом результата**  
   Реализуйте редактирование задачи на втором экране с помощью `ActivityResultContracts.StartActivityForResult`. Передавайте позицию и текст, после редактирования возвращайте изменённый текст и вызывайте `viewModel.updateTask()`.

4. **Удаление свайпом**  
   Добавьте возможность удаления задачи свайпом с помощью `ItemTouchHelper`. При свайпе вызывайте `viewModel.deleteTask(position)` и показывайте Snackbar с возможностью отмены.

---

## 5. Контрольные вопросы

1. Для чего нужен ViewModel? Как он помогает при повороте экрана?
2. Чем StateFlow отличается от LiveData? В каких случаях предпочтительнее использовать StateFlow?
3. Что такое `lifecycleScope` и `repeatOnLifecycle`? Зачем они нужны при подписке на StateFlow?
4. Как обновить данные в StateFlow?
5. Какие преимущества даёт вынос логики в ViewModel с точки зрения тестирования?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинг класса `MainViewModel`.
- Листинг обновлённого `MainActivity.kt`.
- Листинг обновлённого `TaskAdapter.kt`.
- Скриншоты работающего приложения (список задач, поворот экрана – данные сохранились).
- Ответы на контрольные вопросы.
- Вывод по работе (что нового узнали, какие проблемы решили).

---

## 7. Возможные ошибки и их решение

- **При подписке на StateFlow список не обновляется** – проверьте, что используется `repeatOnLifecycle` и `collect` внутри корутины. Убедитесь, что в адаптере вызывается `updateData` с новым списком.
- **ViewModel не сохраняет состояние после поворота** – убедитесь, что используете `by viewModels()`, а не создаёте экземпляр вручную.
- **Ошибка "Cannot create an instance of class ViewModel"** – добавьте зависимость `implementation 'androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0'` и выполните Sync.
- **При удалении задачи по долгому нажатию адаптер не обновляется** – проверьте, что в `updateData` вызывается `notifyDataSetChanged()`.
- **Корутины не работают** – добавьте импорт `import androidx.lifecycle.lifecycleScope` и `import androidx.lifecycle.repeatOnLifecycle`.

---

**Успешной работы!**