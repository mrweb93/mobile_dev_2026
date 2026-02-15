# Лабораторная работа №11
## Рефакторинг: добавление слоя Repository между ViewModel и Room

**Длительность:** 1 час 30 минут  
**Цель работы:** Изучить архитектурный паттерн Repository, научиться выделять слой доступа к данным, отделяя его от бизнес-логики, выполнить рефакторинг существующего приложения для использования репозитория.

---

## 1. Теоретическая справка

### 1.1. Паттерн Repository
**Repository** (репозиторий) — это архитектурный компонент, который инкапсулирует логику доступа к данным, предоставляя единый API для работы с данными из различных источников (база данных, сеть, кэш). В классической Clean Architecture Repository находится между источниками данных (Data Sources) и бизнес-логикой (Use Cases / ViewModel).

Основные задачи репозитория:
- Абстрагировать источник данных от остального приложения.
- Предоставлять чистый API для работы с данными (например, методы `getTasks()`, `addTask()`, `deleteTask()`).
- Управлять кэшированием, синхронизацией и обработкой ошибок.

### 1.2. Зачем нужен Repository?
- **Разделение ответственности**: ViewModel не знает, откуда берутся данные (БД, сеть, файлы) и как они сохраняются.
- **Тестирование**: можно легко заменить реальный репозиторий на mock-объект при тестировании ViewModel.
- **Гибкость**: при изменении источника данных (например, замена Room на другую БД или добавление сетевого источника) меняется только реализация репозитория, но не ViewModel.
- **Единая точка доступа**: все операции с данными проходят через репозиторий, что упрощает добавление кэширования или логирования.

### 1.3. Структура после рефакторинга

До:
```
ViewModel → DAO (Room)
```

После:
```
ViewModel → Repository (интерфейс) → RepositoryImpl → DAO (Room)
```

### 1.4. Репозиторий и корутины
Методы репозитория, как и методы DAO, должны быть `suspend` для асинхронной работы, либо возвращать `Flow`. Это позволяет вызывать их из ViewModel в корутинах.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом `TodoApp`, который был разработан в лабораторной работе №10 (с Room и ViewModel).
- Эмулятор или реальное устройство для проверки работоспособности после рефакторинга.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `TodoApp`, созданный в лабораторной работе №10. Убедитесь, что проект компилируется и работает корректно: задачи добавляются, сохраняются после перезапуска, чекбоксы работают.

### Этап 2. Создание интерфейса репозитория (10 мин)

Создайте пакет `data` (если его нет) и внутри него — пакет `repository`. Создайте интерфейс `TaskRepository`, который будет определять контракт для работы с задачами.

```kotlin
package com.example.todoapp.data.repository

import com.example.todoapp.database.TaskEntity
import kotlinx.coroutines.flow.Flow

interface TaskRepository {
    fun getAllTasks(): Flow<List<TaskEntity>>
    suspend fun addTask(title: String)
    suspend fun deleteTask(task: TaskEntity)
    suspend fun updateTask(task: TaskEntity)
    suspend fun toggleTaskCompletion(task: TaskEntity, isCompleted: Boolean)
    suspend fun deleteAllTasks()
}
```

Обратите внимание: сигнатуры методов повторяют методы ViewModel, но теперь они принадлежат репозиторию.

### Этап 3. Создание реализации репозитория (15 мин)

В том же пакете создайте класс `TaskRepositoryImpl`, реализующий интерфейс `TaskRepository`. Он будет принимать `TaskDao` в конструкторе и делегировать вызовы к DAO.

```kotlin
package com.example.todoapp.data.repository

import com.example.todoapp.database.TaskDao
import com.example.todoapp.database.TaskEntity
import kotlinx.coroutines.flow.Flow
import javax.inject.Inject

class TaskRepositoryImpl(
    private val taskDao: TaskDao
) : TaskRepository {

    override fun getAllTasks(): Flow<List<TaskEntity>> = taskDao.getAllTasks()

    override suspend fun addTask(title: String) {
        val task = TaskEntity(title = title)
        taskDao.insertTask(task)
    }

    override suspend fun deleteTask(task: TaskEntity) {
        taskDao.deleteTask(task)
    }

    override suspend fun updateTask(task: TaskEntity) {
        taskDao.updateTask(task)
    }

    override suspend fun toggleTaskCompletion(task: TaskEntity, isCompleted: Boolean) {
        val updatedTask = task.copy(isCompleted = isCompleted)
        taskDao.updateTask(updatedTask)
    }

    override suspend fun deleteAllTasks() {
        taskDao.deleteAll()
    }
}
```

### Этап 4. Рефакторинг MainViewModel (15 мин)

Теперь измените `MainViewModel`, чтобы он использовал репозиторий, а не напрямую DAO.

```kotlin
package com.example.todoapp.ui.theme

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.todoapp.data.repository.TaskRepository
import com.example.todoapp.database.TaskEntity
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class MainViewModel(
    private val repository: TaskRepository
) : ViewModel() {

    val tasks: StateFlow<List<TaskEntity>> = repository.getAllTasks()
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = emptyList()
        )

    fun addTask(title: String) {
        viewModelScope.launch {
            repository.addTask(title)
        }
    }

    fun deleteTask(task: TaskEntity) {
        viewModelScope.launch {
            repository.deleteTask(task)
        }
    }

    fun toggleTaskCompletion(task: TaskEntity, isCompleted: Boolean) {
        viewModelScope.launch {
            repository.toggleTaskCompletion(task, isCompleted)
        }
    }

    fun deleteAllTasks() {
        viewModelScope.launch {
            repository.deleteAllTasks()
        }
    }
}
```

### Этап 5. Обновление фабрики ViewModel (10 мин)

Измените `MainViewModelFactory`, чтобы она принимала репозиторий, а не базу данных.

```kotlin
package com.example.todoapp

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import com.example.todoapp.data.repository.TaskRepository
import com.example.todoapp.ui.theme.MainViewModel

class MainViewModelFactory(
    private val repository: TaskRepository
) : ViewModelProvider.Factory {
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(MainViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return MainViewModel(repository) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

### Этап 6. Обновление MainActivity (10 мин)

В `MainActivity` нужно создать экземпляр репозитория и передать его в фабрику. Так как у нас уже есть база данных, мы можем получить DAO и создать репозиторий.

```kotlin
class MainActivity : AppCompatActivity() {
    private val database by lazy { AppDatabase.getInstance(this) }
    private val repository by lazy { TaskRepositoryImpl(database.taskDao()) }
    private val viewModel: MainViewModel by viewModels {
        MainViewModelFactory(repository)
    }

    // ... остальной код без изменений
}
```

Убедитесь, что импорты корректны.

### Этап 7. Проверка работоспособности (10 мин)

Запустите приложение. Проверьте все функции:
- Добавление новой задачи.
- Отметка чекбокса (изменение статуса).
- Удаление по долгому нажатию.
- Закройте приложение и откройте снова — данные должны сохраниться.

Если всё работает как раньше — рефакторинг выполнен успешно.

### Этап 8. Анализ изменений (5 мин)

Обсудите, что изменилось:
- ViewModel больше не зависит от Room напрямую, только от интерфейса `TaskRepository`.
- При необходимости заменить Room на другой источник данных (например, Firebase), достаточно создать новую реализацию репозитория, не трогая ViewModel.
- Код стал более модульным и тестируемым.

### Этап 9. Дополнительное задание (оставшееся время)

Если осталось время, добавьте в репозиторий метод поиска задач по тексту (как в индивидуальных заданиях Лаб.10) и используйте его в ViewModel.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Добавление источника данных "In-Memory"**  
   Создайте альтернативную реализацию `TaskRepository` — `InMemoryTaskRepository`, которая хранит список задач в памяти (без БД). Переключите приложение на неё и убедитесь, что ViewModel работает без изменений (данные не сохраняются между сессиями, но это демонстрирует гибкость).

2. **Тестирование ViewModel с mock-репозиторием**  
   Напишите простой тест для `MainViewModel`, используя mock-репозиторий (например, с помощью `Mockito` или вручную создав поддельную реализацию). Проверьте, что при вызове `addTask` вызывается соответствующий метод репозитория.

3. **Добавление обработки ошибок**  
   Модифицируйте репозиторий так, чтобы методы могли возвращать `Result` (успех/ошибка). Добавьте в ViewModel обработку ошибок (например, показывать Toast).

4. **Внедрение зависимостей через Dagger Hilt**  
   (Если студенты знакомы с DI) Добавьте в проект Hilt и настройте внедрение репозитория и базы данных. Замените ручное создание в `MainActivity` на инъекцию.

---

## 5. Контрольные вопросы

1. Какую роль выполняет слой Repository в архитектуре приложения?
2. Какие преимущества даёт использование Repository по сравнению с прямым обращением к DAO из ViewModel?
3. Как изменится ViewModel, если мы захотим добавить ещё один источник данных (например, сетевое API)?
4. Почему методы репозитория объявлены как `suspend`? 
5. Что такое инверсия зависимостей и как она применяется в данном рефакторинге?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги всех созданных/изменённых файлов: `TaskRepository.kt`, `TaskRepositoryImpl.kt`, `MainViewModel.kt`, `MainViewModelFactory.kt`, `MainActivity.kt`.
- Скриншоты работающего приложения (можно те же, что и в Лаб.10, но важно показать, что функциональность сохранена).
- Ответы на контрольные вопросы.
- Вывод по работе (что дало добавление слоя Repository, какие перспективы открывает).

---

## 7. Возможные ошибки и их решение

- **Ошибка компиляции "Unresolved reference"** – проверьте импорты, особенно для `TaskRepositoryImpl` и `MainViewModelFactory`.
- **NullPointerException при создании репозитория** – убедитесь, что `database.taskDao()` не возвращает `null` и что база данных инициализирована корректно.
- **Приложение не видит методы репозитория** – проверьте, что класс `TaskRepositoryImpl` реализует все методы интерфейса.
- **Flow не обновляется после изменений** – убедитесь, что в репозитории методы, возвращающие `Flow`, правильно делегируются DAO, а изменяющие методы вызываются в `viewModelScope`.

---

## 8. Заключение

В результате рефакторинга мы выделили слой Repository, что улучшило архитектуру приложения, сделало её более гибкой и тестируемой. Теперь приложение соответствует рекомендациям Google по архитектуре Android-приложений (Guide to app architecture).

**Успешной работы!**