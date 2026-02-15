# Лабораторная работа №5
## Счетчик нажатий, поле ввода и отображение текста. Реализация ToDo-списка

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться обрабатывать пользовательский ввод, работать с состоянием (счетчик, список задач), динамически обновлять интерфейс приложения на Kotlin.

---

## 1. Теоретическая справка

### 1.1. Компоненты для ввода и отображения
- **`EditText`** – поле для ввода текста пользователем. Получить текст можно методом `text.toString()`.
- **`TextView`** – для вывода текста.
- **`Button`** – кнопка для выполнения действия.

### 1.2. Обработка событий
Установка слушателя на кнопку:
```kotlin
button.setOnClickListener {
    // действия при нажатии
}
```

### 1.3. Работа со списками
Для хранения задач удобно использовать `MutableList<String>`:
```kotlin
val tasks = mutableListOf<String>()
tasks.add("Новая задача")
```
Для отображения всех задач в одном `TextView` можно преобразовать список в строку с разделителем:
```kotlin
textView.text = tasks.joinToString("\n") // каждая задача с новой строки
```

### 1.4. Обновление интерфейса
При изменении данных (счетчика, списка задач) нужно вручную обновить соответствующие `TextView` (присвоить новые значения).

### 1.5. Сохранение состояния при повороте экрана
По умолчанию при повороте экрана активность пересоздаётся и данные теряются. Для их сохранения можно использовать `onSaveInstanceState` или `ViewModel`, но в рамках лабораторной достаточно отметить эту особенность.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с установленным SDK.
- Эмулятор или реальное устройство для запуска приложения.

---

## 3. Порядок выполнения работы

### Этап 1. Создание нового проекта (5 мин)

Создайте новый проект с шаблоном **Empty Views Activity**:
- **Name:** `TodoApp`
- **Package name:** `com.example.todoapp`
- **Language:** Kotlin
- **Minimum SDK:** API 24

### Этап 2. Подготовка ресурсов (5 мин)

В файл `res/values/strings.xml` добавьте необходимые строки:
```xml
<resources>
    <string name="app_name">TodoApp</string>
    <string name="counter_text">Счётчик: %d</string>
    <string name="button_increment">+1</string>
    <string name="hint_input">Введите текст</string>
    <string name="button_show">Показать</string>
    <string name="button_add_task">Добавить задачу</string>
    <string name="label_entered">Вы ввели:</string>
    <string name="label_tasks">Список задач:</string>
</resources>
```

### Этап 3. Верстка интерфейса (20 мин)

Откройте `activity_main.xml`. Создайте экран с тремя функциональными блоками, используя `LinearLayout` (вертикальный) или `ConstraintLayout`. Пример разметки на `LinearLayout`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <!-- Блок 1: Счётчик -->
    <TextView
        android:id="@+id/textCounter"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/counter_text"
        android:textSize="24sp"
        android:layout_marginBottom="16dp"/>

    <Button
        android:id="@+id/buttonIncrement"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/button_increment"
        android:layout_marginBottom="24dp"/>

    <!-- Блок 2: Поле ввода и отображение текста -->
    <EditText
        android:id="@+id/editTextInput"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="@string/hint_input"
        android:inputType="text"
        android:layout_marginBottom="8dp"/>

    <Button
        android:id="@+id/buttonShow"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/button_show"
        android:layout_marginBottom="8dp"/>

    <TextView
        android:id="@+id/textEntered"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/label_entered"
        android:textSize="18sp"
        android:layout_marginBottom="24dp"/>

    <!-- Блок 3: ToDo список -->
    <EditText
        android:id="@+id/editTextTask"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="@string/hint_input"
        android:inputType="text"
        android:layout_marginBottom="8dp"/>

    <Button
        android:id="@+id/buttonAddTask"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/button_add_task"
        android:layout_marginBottom="8dp"/>

    <TextView
        android:id="@+id/textTasks"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:text="@string/label_tasks"
        android:textSize="18sp"
        android:background="#F0F0F0"
        android:padding="8dp"/>

</LinearLayout>
```

### Этап 4. Реализация счётчика (10 мин)

В `MainActivity` объявите переменную-счётчик и обновляйте её при нажатии кнопки.

```kotlin
class MainActivity : AppCompatActivity() {
    private var counter = 0

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val textCounter = findViewById<TextView>(R.id.textCounter)
        val buttonIncrement = findViewById<Button>(R.id.buttonIncrement)

        // Начальное значение
        updateCounterDisplay(textCounter)

        buttonIncrement.setOnClickListener {
            counter++
            updateCounterDisplay(textCounter)
        }
    }

    private fun updateCounterDisplay(textView: TextView) {
        textView.text = getString(R.string.counter_text, counter)
    }
}
```

### Этап 5. Отображение введенного текста (10 мин)

Добавьте обработку кнопки "Показать", чтобы текст из `EditText` отображался в отдельном `TextView`.

```kotlin
val editTextInput = findViewById<EditText>(R.id.editTextInput)
val buttonShow = findViewById<Button>(R.id.buttonShow)
val textEntered = findViewById<TextView>(R.id.textEntered)

buttonShow.setOnClickListener {
    val inputText = editTextInput.text.toString()
    textEntered.text = getString(R.string.label_entered) + " $inputText"
}
```

### Этап 6. Реализация ToDo-списка (20 мин)

Создайте список задач, кнопку добавления и обновление отображения.

```kotlin
val editTextTask = findViewById<EditText>(R.id.editTextTask)
val buttonAddTask = findViewById<Button>(R.id.buttonAddTask)
val textTasks = findViewById<TextView>(R.id.textTasks)

val tasks = mutableListOf<String>()

buttonAddTask.setOnClickListener {
    val task = editTextTask.text.toString()
    if (task.isNotBlank()) {
        tasks.add(task)
        updateTasksDisplay()
        editTextTask.text.clear() // очищаем поле ввода
    } else {
        Toast.makeText(this, "Введите задачу", Toast.LENGTH_SHORT).show()
    }
}

private fun updateTasksDisplay() {
    if (tasks.isEmpty()) {
        textTasks.text = getString(R.string.label_tasks)
    } else {
        textTasks.text = tasks.joinToString("\n• ") { "• $it" }
    }
}
```

### Этап 7. Запуск и тестирование (10 мин)

Запустите приложение на эмуляторе или устройстве. Проверьте:
- Счётчик увеличивается при каждом нажатии.
- Введённый текст отображается после нажатия "Показать".
- Задачи добавляются в список и отображаются.

### Этап 8. Дополнительные улучшения (оставшееся время, 10 мин)

1. Добавьте кнопку для сброса счётчика.
2. Добавьте кнопку для удаления последней задачи.
3. Обратите внимание, что при повороте экрана данные теряются. Для сохранения можно использовать `onSaveInstanceState` (по желанию).

Пример сохранения состояния:

```kotlin
override fun onSaveInstanceState(outState: Bundle) {
    super.onSaveInstanceState(outState)
    outState.putInt("counter", counter)
    outState.putStringArrayList("tasks", ArrayList(tasks))
}

override fun onRestoreInstanceState(savedInstanceState: Bundle) {
    super.onRestoreInstanceState(savedInstanceState)
    counter = savedInstanceState.getInt("counter")
    tasks.clear()
    tasks.addAll(savedInstanceState.getStringArrayList("tasks") ?: emptyList())
    updateCounterDisplay(findViewById(R.id.textCounter))
    updateTasksDisplay()
}
```

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной доработки:

1. **Удаление задач**  
   Добавьте второе поле ввода для номера задачи (индекса) и кнопку "Удалить по индексу". При нажатии задача с указанным индексом удаляется из списка (с проверкой границ).

2. **Счётчик задач**  
   Добавьте `TextView`, показывающий общее количество задач в списке. Обновляйте его при каждом добавлении.

3. **Очистка всех задач**  
   Добавьте кнопку "Очистить всё", которая удаляет все задачи и обновляет отображение.

4. **Редактирование задачи**  
   Добавьте возможность редактирования: при долгом нажатии на задачу (в списке) она появляется в поле ввода для изменения, и кнопка "Добавить" меняет текст на "Обновить". (Сложно, требует динамического управления)

---

## 5. Контрольные вопросы

1. Как получить текст из `EditText`?
2. Почему при повороте экрана данные (счётчик, список задач) сбрасываются? Как это можно исправить?
3. Для чего используется `joinToString`? Как изменить разделитель?
4. В чём разница между `List` и `MutableList`?
5. Как очистить поле ввода после добавления задачи?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинг файла `activity_main.xml`.
- Листинг `MainActivity.kt` с полным кодом (включая дополнительные улучшения, если выполнялись).
- Скриншот работающего приложения с демонстрацией всех функций (счётчик, отображение текста, список задач).
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Поле `EditText` не очищается** – проверьте, что вызван метод `editTextTask.text.clear()`.
- **Список задач не обновляется на экране** – убедитесь, что после изменения списка вызвана `updateTasksDisplay()`.
- **При повороте экрана данные пропадают** – это ожидаемо без сохранения. Для демонстрации можно показать, как использовать `onSaveInstanceState`.
- **Текст в `TextView` не переносится на новую строку** – используйте `\n` в строке или `joinToString("\n")`.

---

**Успешной работы!**
