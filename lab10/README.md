# Лабораторная работа №10
## Интеграция Room в проект. Сохранение списка задач в БД

**Длительность:** 1 час 30 минут  
**Цель работы:** Изучить основы работы с Room Database — официальной библиотекой для работы с SQLite в Android. Научиться создавать Entity, DAO, Database, интегрировать Room с ViewModel и корутинами, обеспечить сохранение списка задач между сессиями приложения.

---

## 1. Теоретическая справка

### 1.1. Room Database
Room — это библиотека-обёртка над SQLite, предоставляющая удобный и безопасный способ работы с базами данных в Android . Она обеспечивает:
- **Абстракцию над SQLite** — минимум шаблонного кода.
- **Валидацию запросов на этапе компиляции** — если SQL-запрос некорректен, вы увидите ошибку при компиляции, а не в рантайме.
- **Интеграцию с корутинами и LiveData/Flow** — асинхронная работа "из коробки".

### 1.2. Основные компоненты Room 

1.  **Entity** — класс данных, представляющий таблицу в базе данных. Каждое поле класса — это столбец таблицы.
2.  **DAO (Data Access Object)** — интерфейс, определяющий методы для работы с данными (вставка, удаление, запросы). Room автоматически генерирует реализацию этого интерфейса.
3.  **Database** — абстрактный класс, наследующий от `RoomDatabase`, который служит точкой доступа к базе данных. Он связывает Entity и DAO, а также управляет версионированием.

### 1.3. Взаимодействие с корутинами 
Для асинхронной работы методы DAO объявляются как `suspend`. Это позволяет вызывать их из корутин (например, из `viewModelScope`), не блокируя основной поток.

### 1.4. Миграция и версионирование
При изменении структуры таблиц (добавлении/удалении полей) необходимо увеличивать версию базы данных и писать миграции. В рамках данной лабораторной мы будем использовать начальную версию (1).

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом `TodoApp` (результат выполнения лабораторной работы №8 — с ViewModel и StateFlow).
- Эмулятор или реальное устройство для запуска приложения.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `TodoApp`, созданный в лабораторной работе №8. Убедитесь, что проект успешно компилируется и работает: список задач отображается, добавление и удаление функционируют.

### Этап 2. Добавление зависимостей Room (5 мин)

Откройте файл `app/build.gradle.kts` (Module) и добавьте необходимые плагины и зависимости .

В блоке `plugins` добавьте:
```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt") // Добавить этот плагин для обработки аннотаций
}
```

В блоке `dependencies` добавьте:
```kotlin
dependencies {
    // ... существующие зависимости

    val roomVersion = "2.6.1"
    implementation("androidx.room:room-runtime:$roomVersion")
    implementation("androidx.room:room-ktx:$roomVersion") // Поддержка корутин
    kapt("androidx.room:room-compiler:$roomVersion") // Генерация кода
}
```

Выполните синхронизацию проекта (Sync Now).

### Этап 3. Создание Entity для задачи (10 мин)

Создайте пакет `database` (если его нет). В нём создайте data class `TaskEntity` — это будет представление таблицы задач в базе данных .

```kotlin
package com.example.todoapp.database

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "tasks") // Имя таблицы в БД
data class TaskEntity(
    @PrimaryKey(autoGenerate = true) // Автоматическая генерация ID
    val id: Long = 0,
    val title: String,          // Текст задачи
    val isCompleted: Boolean = false, // Статус выполнения
    val createdTime: Long = System.currentTimeMillis() // Время создания для сортировки
)
```

**Пояснения:**
- `@Entity` указывает, что этот класс — таблица.
- `@PrimaryKey(autoGenerate = true)` — поле ID будет автоматически увеличиваться при вставке новой записи.
- Мы добавили поля `isCompleted` и `createdTime` для расширения функциональности в будущем.

### Этап 4. Создание TaskDao (15 мин)

В том же пакете создайте интерфейс `TaskDao`. Он будет содержать методы для работы с таблицей задач .

```kotlin
package com.example.todoapp.database

import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface TaskDao {

    @Query("SELECT * FROM tasks ORDER BY createdTime DESC")
    fun getAllTasks(): Flow<List<TaskEntity>> // Возвращаем Flow для реактивного обновления

    @Insert(onConflict = OnConflictStrategy.REPLACE) // При конфликте заменять
    suspend fun insertTask(task: TaskEntity)

    @Update
    suspend fun updateTask(task: TaskEntity)

    @Delete
    suspend fun deleteTask(task: TaskEntity)

    @Query("DELETE FROM tasks")
    suspend fun deleteAll()

    @Query("SELECT * FROM tasks WHERE id = :id")
    suspend fun getTaskById(id: Long): TaskEntity?
}
```

**Важные моменты:**
- `getAllTasks()` возвращает `Flow`. Это позволяет автоматически получать обновления при изменении данных в таблице .
- Все операции, изменяющие данные (`suspend`), должны вызываться из корутины.

### Этап 5. Создание Database и синглтона (15 мин)

Создайте абстрактный класс `AppDatabase`, наследующий от `RoomDatabase`, и объект-синглтон для доступа к нему .

```kotlin
package com.example.todoapp.database

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(
    entities = [TaskEntity::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao

    companion object {
        @Volatile
        private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "todo_database"
                )
                    //.fallbackToDestructiveMigration() // Для разработки: при изменении версии БД пересоздавать таблицы
                    .build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

**Пояснения:**
- `@Volatile` и `synchronized` обеспечивают потокобезопасность при создании экземпляра БД (паттерн Singleton).
- `Room.databaseBuilder` создаёт базу данных с именем `todo_database`.

### Этап 6. Модификация TaskAdapter для работы с новыми данными (10 мин)

Адаптер должен работать со списком `TaskEntity`, а не со строками. Обновите `TaskAdapter.kt`:

```kotlin
package com.example.todoapp

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.CheckBox
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView
import com.example.todoapp.database.TaskEntity

class TaskAdapter(
    private var tasks: List<TaskEntity>,
    private val onItemClick: (TaskEntity) -> Unit,
    private val onItemLongClick: (TaskEntity) -> Unit,
    private val onCheckChange: (TaskEntity, Boolean) -> Unit
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
        holder.textTask.text = task.title
        holder.checkTask.isChecked = task.isCompleted

        holder.itemView.setOnClickListener {
            onItemClick(task)
        }

        holder.itemView.setOnLongClickListener {
            onItemLongClick(task)
            true
        }

        holder.checkTask.setOnCheckedChangeListener { _, isChecked ->
            onCheckChange(task, isChecked)
        }
    }

    override fun getItemCount(): Int = tasks.size

    fun updateData(newTasks: List<TaskEntity>) {
        tasks = newTasks
        notifyDataSetChanged()
    }
}
```

### Этап 7. Интеграция Room в MainViewModel (20 мин)

Обновите `MainViewModel`, чтобы он получал данные из базы через DAO, а не хранил их в памяти .

```kotlin
package com.example.todoapp.ui.theme

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.todoapp.database.AppDatabase
import com.example.todoapp.database.TaskEntity
import kotlinx.coroutines.flow.*
import kotlinx.coroutines.launch

class MainViewModel(
    private val database: AppDatabase
) : ViewModel() {

    private val taskDao = database.taskDao()

    // Получаем Flow из DAO и преобразуем в StateFlow для Compose/UI
    val tasks: StateFlow<List<TaskEntity>> = taskDao.getAllTasks()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    fun addTask(title: String) {
        viewModelScope.launch {
            val task = TaskEntity(title = title)
            taskDao.insertTask(task)
        }
    }

    fun deleteTask(task: TaskEntity) {
        viewModelScope.launch {
            taskDao.deleteTask(task)
        }
    }

    fun updateTask(task: TaskEntity) {
        viewModelScope.launch {
            taskDao.updateTask(task)
        }
    }

    fun toggleTaskCompletion(task: TaskEntity, isCompleted: Boolean) {
        viewModelScope.launch {
            val updatedTask = task.copy(isCompleted = isCompleted)
            taskDao.updateTask(updatedTask)
        }
    }

    fun deleteAllTasks() {
        viewModelScope.launch {
            taskDao.deleteAll()
        }
    }
}
```

### Этап 8. Обновление MainActivity и фабрики ViewModel (15 мин)

В `MainActivity` нужно передать экземпляр базы данных в ViewModel. Для этого создадим фабрику (как в Лаб.9) или используем инъекцию зависимостей (упрощённо — через Application).

**Вариант 1 (простой):** Создаём фабрику прямо в Activity.

Создайте `MainViewModelFactory.kt`:
```kotlin
package com.example.todoapp

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import com.example.todoapp.database.AppDatabase
import com.example.todoapp.ui.theme.MainViewModel

class MainViewModelFactory(
    private val database: AppDatabase
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(MainViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return MainViewModel(database) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

В `MainActivity.kt` обновите создание ViewModel:

```kotlin
class MainActivity : AppCompatActivity() {
    // Получаем базу данных
    private val database by lazy { AppDatabase.getInstance(this) }
    private val viewModel: MainViewModel by viewModels {
        MainViewModelFactory(database)
    }

    // ... остальной код (onCreate, подписка на Flow, установка адаптера)
}
```

Обновите подписку на Flow в `onCreate`:

```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.tasks.collect { tasks ->
            adapter.updateData(tasks)
        }
    }
}
```

Обновите обработчики событий для адаптера:

```kotlin
adapter = TaskAdapter(
    tasks = emptyList(),
    onItemClick = { task ->
        val intent = Intent(this, DetailActivity::class.java)
        intent.putExtra("task_text", task.title)
        startActivity(intent)
    },
    onItemLongClick = { task ->
        viewModel.deleteTask(task)
        Toast.makeText(this, "Задача удалена", Toast.LENGTH_SHORT).show()
    },
    onCheckChange = { task, isChecked ->
        viewModel.toggleTaskCompletion(task, isChecked)
    }
)
```

### Этап 9. Запуск и тестирование (5 мин)

Запустите приложение. Проверьте:
- Добавление новых задач (данные должны сохраняться после перезапуска приложения).
- Отметка чекбоксов (статус выполнения должен сохраняться).
- Удаление задач по долгому нажатию.
- После закрытия и повторного открытия приложения список задач должен оставаться неизменным.

### Этап 10. Эксперименты (оставшееся время, 5 мин)

Попробуйте добавить несколько задач, затем закройте приложение (смахните из списка недавних) и откройте снова. Данные должны сохраниться.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Сортировка задач**  
   Добавьте в DAO метод, возвращающий задачи, отсортированные по статусу выполнения (сначала невыполненные) и дате создания. Используйте его в ViewModel.

2. **Поиск задач**  
   Реализуйте поиск по названию задачи. Добавьте `EditText` для ввода поискового запроса и соответствующий `@Query` в DAO с использованием `LIKE`.

3. **Категории задач**  
   Добавьте поле `category` в `TaskEntity` и создайте отдельный экран для просмотра задач по категориям.

4. **Дата выполнения**  
   Добавьте в задачу поле `dueDate` (тип `Long` — timestamp) и отображайте его в карточке. Реализуйте выбор даты через `DatePickerDialog`.

---

## 5. Контрольные вопросы

1. Для чего нужна библиотека Room? Какие проблемы она решает по сравнению с прямым использованием SQLite?
2. Назовите три основных компонента Room и объясните их назначение .
3. Почему методы DAO, изменяющие данные, объявляются как `suspend`?
4. Что такое `Flow` и почему его удобно использовать с Room? 
5. Как Room обеспечивает проверку SQL-запросов на этапе компиляции?
6. Зачем нужен паттерн Singleton для экземпляра базы данных?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги всех созданных/изменённых файлов: `TaskEntity.kt`, `TaskDao.kt`, `AppDatabase.kt`, `MainViewModel.kt`, `MainActivity.kt`, `TaskAdapter.kt`.
- Скриншоты работающего приложения с демонстрацией сохранения данных после перезапуска.
- Ответы на контрольные вопросы.
- Вывод по работе (что нового узнали, как Room улучшает архитектуру приложения).

---

## 7. Возможные ошибки и их решение

- **Ошибка "Cannot create an instance of class ViewModel"** – убедитесь, что правильно реализована фабрика ViewModel и передаётся в `by viewModels`.
- **Ошибка "Room cannot verify the data integrity. Looks like you've changed schema but forgot to update the version number"** – если меняли поля в Entity, увеличьте `version` в `@Database` и напишите миграцию или временно используйте `.fallbackToDestructiveMigration()` в билдере.
- **Таблица не создаётся / данные не сохраняются** – проверьте, что база данных инициализируется один раз (синглтон) и что методы DAO вызываются в корутине.
- **При компиляции ошибка "kapt" not found** – убедитесь, что плагин `kotlin-kapt` добавлен в `plugins` и зависимости добавлены с `kapt`.

---

## 8. Дополнительные материалы

- [Официальная документация Room](https://developer.android.com/training/data-storage/room) 
- [Codelab: Android Room with a View](https://developer.android.com/codelabs/android-room-with-a-view-kotlin)

**Успешной работы!**