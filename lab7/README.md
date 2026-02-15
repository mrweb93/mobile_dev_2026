# Лабораторная работа №7
## Добавление второго экрана (детали задачи). Переход по клику на элемент списка

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться создавать многоэкранные приложения, осуществлять переход между экранами с передачей данных через Intent, обрабатывать клики на элементах RecyclerView.

---

## 1. Теоретическая справка

### 1.1. Activity и Intent
- **Activity** представляет один экран приложения. Для создания второго экрана нужно создать новую Activity.
- **Intent** – это объект для выполнения различных операций, в частности для запуска другой Activity. Для передачи данных используется метод `putExtra()`.

Пример создания Intent и запуска:
```kotlin
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("key", value)
startActivity(intent)
```

Получение данных в целевой Activity:
```kotlin
val value = intent.getStringExtra("key")
```

### 1.2. Передача сложных объектов
Для передачи объектов данных рекомендуется реализовать интерфейс `Parcelable` или использовать `Serializable`. В Kotlin можно использовать `@Parcelize` (плагин `kotlin-parcelize`). В этой лабораторной для простоты передадим только текст задачи (String).

### 1.3. Обработка кликов в RecyclerView
Существует несколько способов обработки кликов:
- Передать лямбду в адаптер из Activity.
- Использовать интерфейс слушателя.
- Установить `OnClickListener` на корневой элемент в `onBindViewHolder`.

Рекомендуется передавать лямбду при создании адаптера:
```kotlin
adapter = TaskAdapter(tasks) { position ->
    // обработка клика
}
```

### 1.4. Создание новой Activity
В Android Studio: `File -> New -> Activity -> Empty Activity`. Будет создан класс и соответствующий layout-файл.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом, содержащим реализацию списка задач из Лабораторной работы №6.
- Эмулятор или реальное устройство.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `TodoApp`, который был создан в лабораторной работе №6. Убедитесь, что он успешно компилируется и корректно отображает список задач в `RecyclerView` с карточками.

### Этап 2. Создание второго экрана (DetailActivity) (10 мин)

Создайте новую Activity:
- Нажмите правой кнопкой на пакет `com.example.todoapp` → `New` → `Activity` → `Empty Activity`.
- Имя: `DetailActivity`
- Layout name: `activity_detail`
- Launcher Activity: не нужно, оставьте галочку снятой.

Android Studio создаст два файла: `DetailActivity.kt` и `activity_detail.xml`.

### Этап 3. Разметка экрана деталей (10 мин)

Откройте `activity_detail.xml` и создайте простой интерфейс для отображения текста задачи. Добавьте `TextView` для задачи, кнопку "Назад" (или можно использовать системную кнопку Back). Также можно добавить заголовок.

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp">

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Детали задачи"
        android:textSize="24sp"
        android:textStyle="bold"
        android:layout_marginBottom="24sp"/>

    <TextView
        android:id="@+id/textTaskDetail"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:textSize="18sp"
        android:layout_marginBottom="16sp"/>

    <Button
        android:id="@+id/buttonBack"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Назад"
        android:layout_gravity="center_horizontal"/>

</LinearLayout>
```

### Этап 4. Код DetailActivity (10 мин)

В `DetailActivity.kt` получите переданный текст задачи и отобразите его. Добавьте обработку кнопки "Назад" (завершает активность).

```kotlin
package com.example.todoapp

import android.os.Bundle
import android.widget.Button
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity

class DetailActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_detail)

        val textTaskDetail = findViewById<TextView>(R.id.textTaskDetail)
        val buttonBack = findViewById<Button>(R.id.buttonBack)

        // Получаем данные из Intent
        val taskText = intent.getStringExtra("task_text") ?: "Нет данных"
        textTaskDetail.text = taskText

        buttonBack.setOnClickListener {
            finish() // закрывает текущую активность и возвращает к предыдущей
        }
    }
}
```

### Этап 5. Добавление обработки клика в адаптер (15 мин)

Откройте класс `TaskAdapter` из предыдущей лабораторной. Модифицируйте его так, чтобы он принимал лямбду-обработчик клика и вызывал её при клике на элемент списка.

```kotlin
class TaskAdapter(
    private val tasks: MutableList<String>,
    private val onItemClick: (Int) -> Unit   // лямбда, принимающая позицию
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

        // Устанавливаем обработчик клика на всю карточку
        holder.itemView.setOnClickListener {
            onItemClick(position)
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
}
```

### Этап 6. Обновление MainActivity для обработки кликов (15 мин)

В `MainActivity.kt` измените создание адаптера, передав лямбду, которая будет запускать `DetailActivity` с передачей текста задачи.

```kotlin
// Вместо прежнего создания:
// adapter = TaskAdapter(tasks)

// Теперь:
adapter = TaskAdapter(tasks) { position ->
    val taskText = tasks[position]
    val intent = Intent(this, DetailActivity::class.java)
    intent.putExtra("task_text", taskText)
    startActivity(intent)
}
```

Не забудьте добавить импорт `import android.content.Intent`.

### Этап 7. Запуск и тестирование (10 мин)

Запустите приложение. Добавьте несколько задач. Попробуйте нажать на любую карточку задачи. Должен открыться второй экран с текстом этой задачи. Нажмите "Назад" – вы вернётесь к списку.

### Этап 8. Дополнительные улучшения (оставшееся время, 15 мин)

1. **Передача позиции и удаление с подтверждением**  
   На экране деталей добавьте кнопку "Удалить", которая удаляет задачу из списка и возвращает на главный экран. Для этого можно использовать `startActivityForResult` или `ViewModel`, но проще передать позицию и использовать `setResult` с последующим обновлением в `onActivityResult`. Однако в современных приложениях рекомендуется использовать SharedViewModel или другой подход. В рамках лабораторной можно упрощённо: при удалении отправлять broadcast или сохранять результат в глобальной переменной – не очень хорошо. Лучше оставить как дополнительное задание.

2. **Редактирование задачи**  
   Добавьте на экран деталей `EditText` с текущим текстом и кнопку "Сохранить". При сохранении изменения должны вернуться в список. Для этого можно использовать `startActivityForResult` (устаревший) или `ActivityResultContracts.StartActivityForResult` (новый API). Можно показать пример с `setResult`.

3. **Передача объекта Task (Parcelable)**  
   Создайте data class `Task` с полями `text: String`, `isCompleted: Boolean`, `created: Long`. Реализуйте `Parcelable` с помощью аннотации `@Parcelize` (добавив плагин `kotlin-parcelize`). Передавайте объект целиком.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Удаление задачи с экрана деталей**  
   Добавьте на экран деталей кнопку "Удалить". При нажатии задача удаляется из списка, второй экран закрывается, список обновляется. Для этого можно передать позицию задачи, а результат обработать в `onActivityResult` (или использовать современный `ActivityResultLauncher`).

2. **Редактирование задачи**  
   На экране деталей отображайте `EditText` с текущим текстом и кнопку "Сохранить". После сохранения обновите текст задачи в списке и закройте экран.

3. **Передача объекта данных**  
   Создайте data class `Task` с полями: текст, дата создания, статус (выполнена/нет). Реализуйте `Parcelable`. Передавайте объект на второй экран и отображайте все его поля.

4. **Анимация перехода**  
   Добавьте кастомную анимацию перехода между экранами (overridePendingTransition).

---

## 5. Контрольные вопросы

1. Что такое Intent? Какие виды Intent существуют?
2. Как передать данные из одной Activity в другую?
3. Какие способы обработки кликов на элементах RecyclerView вы знаете?
4. Как создать новую Activity в Android Studio?
5. Для чего используется метод `finish()`?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинг файла `activity_detail.xml`.
- Листинг `DetailActivity.kt`.
- Листинг обновлённого `TaskAdapter.kt`.
- Листинг `MainActivity.kt` с изменениями.
- Скриншоты главного экрана и экрана деталей.
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **При клике на элемент ничего не происходит** – проверьте, что лямбда передана в адаптер и что в `onBindViewHolder` вызывается `holder.itemView.setOnClickListener`.
- **Второй экран открывается, но текст не отображается** – убедитесь, что ключ в `putExtra` и `getStringExtra` совпадают (например, `"task_text"`). Проверьте, не `null` ли значение.
- **Ошибка компиляции: "Unresolved reference"** – импортируйте класс `Intent` (`import android.content.Intent`).
- **При повороте экрана данные теряются** – это нормально для данной лабораторной, но можно упомянуть о необходимости сохранения состояния.

---

**Успешной работы!**
