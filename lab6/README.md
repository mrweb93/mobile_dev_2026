# Лабораторная работа №6
## Отображение списка задач из предыдущей лабораторной в красивых карточках

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться использовать `RecyclerView` для отображения списка данных, освоить создание адаптера и ViewHolder, применить `CardView` для оформления элементов списка.

---

## 1. Теоретическая справка

### 1.1. RecyclerView
`RecyclerView` – это современный и эффективный компонент для отображения больших списков. Он переиспользует (recycler) элементы списка при прокрутке, что экономит ресурсы.
x
Для работы `RecyclerView` необходимы:
- **LayoutManager** – отвечает за расположение элементов (линейно, сеткой, горизонтально). Обычно используется `LinearLayoutManager`.
- **Adapter** – создаёт элементы списка и связывает данные с View.
- **ViewHolder** – кэширует ссылки на вьюхи элемента для быстрого доступа.

### 1.2. CardView
`CardView` – это компонент из библиотеки Material, который реализует карточку с закруглёнными углами и тенью. Используется для создания красивого оформления каждого элемента списка.

Подключение зависимости (если не добавлена):
```gradle
dependencies {
    implementation 'androidx.cardview:cardview:1.0.0'
    implementation 'androidx.recyclerview:recyclerview:1.3.2'
}
```

### 1.3. Адаптер
Адаптер наследуется от `RecyclerView.Adapter<ViewHolder>` и переопределяет методы:
- `onCreateViewHolder` – создаёт новый элемент (инфлейтит layout).
- `onBindViewHolder` – заполняет элемент данными.
- `getItemCount` – возвращает размер списка.

### 1.4. Data class для задачи
В предыдущей работе мы использовали список строк. Теперь лучше создать класс задачи с дополнительными полями (например, выполнена/не выполнена), но для простоты оставим строку, но обернём в карточку.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом из лабораторной работы №5 (можно использовать его же, скопировав или продолжив).
- Эмулятор или реальное устройство.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `TodoApp`, созданный в лабораторной работе №5. Если его нет, создайте новый проект с Empty Activity и повторите код из Лаб.5 (счётчик, поле ввода, список задач). Либо можно упростить и оставить только функционал списка задач.

Убедитесь, что в файле `build.gradle` (Module) добавлены зависимости (обычно они уже есть, но проверим):
```gradle
dependencies {
    implementation 'androidx.recyclerview:recyclerview:1.3.2'
    implementation 'androidx.cardview:cardview:1.0.0'
}
```
Если нет – добавьте и синхронизируйте проект.

### Этап 2. Создание layout для элемента списка (10 мин)

Создайте новый файл разметки `item_task.xml` в папке `res/layout`. Это будет карточка задачи.

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="8dp"
    app:cardElevation="4dp"
    app:cardBackgroundColor="#FFFFFF">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        android:padding="16dp">

        <TextView
            android:id="@+id/textTask"
            android:layout_width="0dp"
            android:layout_height="wrap_content"
            android:layout_weight="1"
            android:textSize="18sp"
            android:textColor="#333333"/>

        <CheckBox
            android:id="@+id/checkTask"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"/>

    </LinearLayout>

</androidx.cardview.widget.CardView>
```

Этот элемент содержит текст задачи и чекбокс для отметки выполнения. Чекбокс пока не будем обрабатывать, но он добавит интерактивности.

### Этап 3. Создание адаптера (15 мин)

Создайте класс `TaskAdapter` в пакете `com.example.todoapp` (или в отдельном пакете `adapter`).

```kotlin
package com.example.todoapp

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.CheckBox
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class TaskAdapter(private val tasks: MutableList<String>) :
    RecyclerView.Adapter<TaskAdapter.TaskViewHolder>() {

    // ViewHolder хранит ссылки на элементы внутри карточки
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
        // Обработка чекбокса (опционально)
        holder.checkTask.setOnCheckedChangeListener { _, isChecked ->
            // Можно добавить логику отметки выполнения, например, перечеркивание текста
            if (isChecked) {
                holder.textTask.paintFlags = holder.textTask.paintFlags or android.graphics.Paint.STRIKE_THRU_TEXT_FLAG
            } else {
                holder.textTask.paintFlags = holder.textTask.paintFlags and android.graphics.Paint.STRIKE_THRU_TEXT_FLAG.inv()
            }
        }
    }

    override fun getItemCount(): Int = tasks.size

    // Метод для обновления списка
    fun updateData(newTasks: List<String>) {
        tasks.clear()
        tasks.addAll(newTasks)
        notifyDataSetChanged()
    }
}
```

### Этап 4. Обновление разметки главного экрана (10 мин)

В `activity_main.xml` замените старый `TextView` для списка задач на `RecyclerView`. Также можно оставить поле ввода и кнопку добавления.

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <!-- Поле ввода и кнопка добавления (как в Лаб.5) -->
    <EditText
        android:id="@+id/editTextTask"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Введите задачу"
        android:layout_marginBottom="8dp"/>

    <Button
        android:id="@+id/buttonAddTask"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Добавить задачу"
        android:layout_marginBottom="16dp"/>

    <!-- RecyclerView для списка задач -->
    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewTasks"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

</LinearLayout>
```

### Этап 5. Настройка RecyclerView в MainActivity (15 мин)

В `MainActivity.kt` удалите старый `TextView` для задач и добавьте `RecyclerView`, адаптер и `LinearLayoutManager`.

```kotlin
package com.example.todoapp

import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView

class MainActivity : AppCompatActivity() {

    private val tasks = mutableListOf<String>()
    private lateinit var adapter: TaskAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val editTextTask = findViewById<EditText>(R.id.editTextTask)
        val buttonAddTask = findViewById<Button>(R.id.buttonAddTask)
        val recyclerView = findViewById<RecyclerView>(R.id.recyclerViewTasks)

        // Настройка RecyclerView
        recyclerView.layoutManager = LinearLayoutManager(this)
        adapter = TaskAdapter(tasks)
        recyclerView.adapter = adapter

        // Добавление задачи
        buttonAddTask.setOnClickListener {
            val task = editTextTask.text.toString()
            if (task.isNotBlank()) {
                tasks.add(task)
                adapter.notifyItemInserted(tasks.size - 1) // более эффективно, чем notifyDataSetChanged
                editTextTask.text.clear()
            } else {
                Toast.makeText(this, "Введите задачу", Toast.LENGTH_SHORT).show()
            }
        }

        // Восстановление данных при повороте (опционально, см. Лаб.5)
        if (savedInstanceState != null) {
            val savedTasks = savedInstanceState.getStringArrayList("tasks")
            if (savedTasks != null) {
                tasks.clear()
                tasks.addAll(savedTasks)
                adapter.notifyDataSetChanged()
            }
        }
    }

    override fun onSaveInstanceState(outState: Bundle) {
        super.onSaveInstanceState(outState)
        outState.putStringArrayList("tasks", ArrayList(tasks))
    }
}
```

### Этап 6. Запуск и тестирование (10 мин)

Запустите приложение. Добавьте несколько задач. Убедитесь, что они отображаются в виде карточек, каждая с чекбоксом. Проверьте, что чекбокс перечёркивает текст при отметке (если добавили эту логику).

### Этап 7. Добавление функциональности удаления (опционально, 15 мин)

Добавьте возможность удалять задачу свайпом или по долгому нажатию. Например, удаление при долгом нажатии на карточку.

В адаптере добавьте интерфейс обратного вызова:

```kotlin
class TaskAdapter(
    private val tasks: MutableList<String>,
    private val onItemLongClick: (Int) -> Unit
) : RecyclerView.Adapter<TaskAdapter.TaskViewHolder>() {
    // ...
    override fun onBindViewHolder(holder: TaskViewHolder, position: Int) {
        // ...
        holder.itemView.setOnLongClickListener {
            onItemLongClick(position)
            true
        }
    }
}
```

В `MainActivity` при создании адаптера передайте лямбду:
```kotlin
adapter = TaskAdapter(tasks) { position ->
    tasks.removeAt(position)
    adapter.notifyItemRemoved(position)
}
```

Не забудьте добавить импорты и обновить конструктор.

### Этап 8. Дополнительные улучшения (оставшееся время)

- Добавьте разные цвета для карточек в зависимости от четности позиции.
- Добавьте анимацию появления/удаления.
- Используйте `SwipeToDelete` через `ItemTouchHelper` (сложнее, можно показать, но, вероятно, не успеют).

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной доработки:

1. **Удаление свайпом**  
   Реализуйте удаление задачи свайпом влево/вправо с помощью `ItemTouchHelper.SimpleCallback`. При свайпе задача удаляется, показывается `Snackbar` с возможностью отмены.

2. **Редактирование по клику**  
   Сделайте так, чтобы при клике на задачу открывался диалог с предзаполненным текстом для редактирования. После подтверждения текст задачи обновляется.

3. **Счетчик выполненных задач**  
   Добавьте в шапку экрана `TextView`, показывающий количество выполненных задач (отмеченных чекбоксов). Обновляйте его при каждом изменении состояния чекбокса.

4. **Разделители и заголовки**  
   Добавьте разделение задач по категориям (например, «Работа», «Личное») с заголовками секций. Для этого потребуется использовать разные типы View в адаптере.

---

## 5. Контрольные вопросы

1. Для чего нужен `RecyclerView`? Чем он лучше `ListView`?
2. Какие компоненты необходимы для работы `RecyclerView`?
3. Что такое `ViewHolder` и для чего он используется?
4. Чем отличается `notifyDataSetChanged()` от `notifyItemInserted()`?
5. Как добавить обработку кликов на элементы `RecyclerView`?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинг файла `item_task.xml`.
- Листинг класса `TaskAdapter`.
- Листинг `MainActivity.kt` с изменениями.
- Скриншот работающего приложения с несколькими карточками.
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **RecyclerView не отображается** – проверьте, что установлен LayoutManager и адаптер привязан.
- **При добавлении задачи список не обновляется** – убедитесь, что после изменения списка вызван `adapter.notifyItemInserted()` или `notifyDataSetChanged()`.
- **Чекбокс перестаёт перечёркивать текст после прокрутки** – это из-за переиспользования ViewHolder; нужно сохранять состояние чекбокса в модели данных. Для простоты можно игнорировать или сохранять отдельный список Boolean.
- **При удалении позиция сбивается** – если используете удаление через `notifyItemRemoved`, убедитесь, что индекс корректен и список обновлён до уведомления.

---

**Успешной работы!**