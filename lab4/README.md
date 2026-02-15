# Лабораторная работа №4
## Верстка экрана профиля пользователя (аватар, имя, кнопка «Редактировать»)

**Длительность:** 1 час 30 минут  
**Цель работы:** Освоить создание пользовательского интерфейса в Android с использованием `ConstraintLayout`, изучить основные компоненты: `ImageView`, `TextView`, `Button`. Научиться работать с ресурсами (строки, цвета, размеры) и обрабатывать нажатия кнопок.

---

## 1. Теоретическая справка

### 1.1. ConstraintLayout
`ConstraintLayout` – это гибкий и мощный менеджер расположения, который позволяет создавать плоские иерархии представлений. Он используется по умолчанию в шаблонах Android Studio. Основные понятия:
- **constraints (ограничения)** – привязки краёв виджета к другим виджетам или родителю.
- **margin** – отступы от ограничений.
- **bias** – смещение (например, 0.5 – центр).

### 1.2. Основные виджеты
- `ImageView` – для отображения изображений (аватар).
- `TextView` – для текстовой информации (имя, статус и т.д.).
- `Button` – кнопка для выполнения действий.

### 1.3. Ресурсы
Все строки, цвета, размеры рекомендуется выносить в ресурсные файлы:
- `res/values/strings.xml` – тексты.
- `res/values/colors.xml` – цвета.
- `res/values/dimens.xml` – размеры (отступы, размер шрифта).

### 1.4. Обработка нажатий
В Kotlin можно установить слушатель на кнопку:
```kotlin
button.setOnClickListener {
    // действие
}
```

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с установленным SDK.
- Эмулятор или реальное устройство для запуска приложения.

---

## 3. Порядок выполнения работы

### Этап 1. Создание нового проекта (5 мин)

Создайте новый проект с шаблоном **Empty Views Activity**. Заполните:
- **Name:** `ProfileApp`
- **Package name:** `com.example.profileapp`
- **Language:** Kotlin
- **Minimum SDK:** API 24

### Этап 2. Подготовка ресурсов (10 мин)

1. **Строки:** откройте `res/values/strings.xml` и добавьте:
   ```xml
   <resources>
       <string name="app_name">ProfileApp</string>
       <string name="profile_name">Иван Иванов</string>
       <string name="profile_status">Android-разработчик</string>
       <string name="button_edit">Редактировать</string>
       <string name="toast_message">Редактирование профиля</string>
   </resources>
   ```

2. **Цвета:** в `res/values/colors.xml` определите основные цвета:
   ```xml
   <resources>
       <color name="purple_200">#FFBB86FC</color>
       <color name="purple_500">#FF6200EE</color>
       <color name="teal_200">#FF03DAC5</color>
       <color name="black">#FF000000</color>
       <color name="white">#FFFFFFFF</color>
       <color name="gray_light">#F5F5F5</color>
   </resources>
   ```

3. **Размеры:** создайте файл `res/values/dimens.xml`:
   ```xml
   <resources>
       <dimen name="avatar_size">120dp</dimen>
       <dimen name="margin_normal">16dp</dimen>
       <dimen name="margin_small">8dp</dimen>
       <dimen name="text_size_name">24sp</dimen>
       <dimen name="text_size_status">16sp</dimen>
       <dimen name="button_corner_radius">8dp</dimen>
   </resources>
   ```

4. **Изображение аватара:** скачайте или создайте векторную иконку профиля. Можно использовать стандартную из библиотеки Material Icons. Для этого:
   - В Android Studio: `File -> New -> Vector Asset`.
   - Выберите иконку "Account Circle" (или любую другую).
   - Назовите файл `ic_profile.xml` и поместите в `res/drawable`.

### Этап 3. Верстка экрана профиля (25 мин)

Откройте `activity_main.xml`. Убедитесь, что корневой элемент – `ConstraintLayout`. Сверстайте экран по следующей схеме:

- **Аватар** – по центру по горизонтали, отступ сверху.
- **Имя** – под аватаром.
- **Статус** – под именем.
- **Кнопка «Редактировать»** – под статусом.

Итоговый код разметки:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:background="@color/gray_light"
    tools:context=".MainActivity">

    <ImageView
        android:id="@+id/imageAvatar"
        android:layout_width="@dimen/avatar_size"
        android:layout_height="@dimen/avatar_size"
        android:src="@drawable/ic_profile"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintBottom_toTopOf="@+id/textName"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        android:layout_marginTop="@dimen/margin_normal"
        android:contentDescription="@string/profile_name" />

    <TextView
        android:id="@+id/textName"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/profile_name"
        android:textSize="@dimen/text_size_name"
        android:textColor="@color/black"
        android:textStyle="bold"
        app:layout_constraintTop_toBottomOf="@id/imageAvatar"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        android:layout_marginTop="@dimen/margin_small" />

    <TextView
        android:id="@+id/textStatus"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/profile_status"
        android:textSize="@dimen/text_size_status"
        android:textColor="@color/purple_500"
        app:layout_constraintTop_toBottomOf="@id/textName"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        android:layout_marginTop="@dimen/margin_small" />

    <Button
        android:id="@+id/buttonEdit"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="@string/button_edit"
        android:backgroundTint="@color/purple_200"
        app:cornerRadius="@dimen/button_corner_radius"
        app:layout_constraintTop_toBottomOf="@id/textStatus"
        app:layout_constraintLeft_toLeftOf="parent"
        app:layout_constraintRight_toRightOf="parent"
        android:layout_marginTop="@dimen/margin_normal"/>

</androidx.constraintlayout.widget.ConstraintLayout>
```

### Этап 4. Обработка нажатия кнопки (10 мин)

В `MainActivity.kt` добавьте обработчик для кнопки, который будет показывать всплывающее сообщение (Toast).

```kotlin
package com.example.profileapp

import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.widget.Button
import android.widget.Toast

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val buttonEdit = findViewById<Button>(R.id.buttonEdit)
        buttonEdit.setOnClickListener {
            Toast.makeText(this, R.string.toast_message, Toast.LENGTH_SHORT).show()
        }
    }
}
```

### Этап 5. Запуск приложения (10 мин)

Запустите приложение на эмуляторе или устройстве. Убедитесь, что:
- Аватар отображается корректно.
- Имя и статус соответствуют заданным строкам.
- При нажатии на кнопку появляется Toast с сообщением.

### Этап 6. Добавление функциональности (оставшееся время, 20 мин)

Усовершенствуйте экран профиля:
1. Добавьте вторую кнопку «Назад» (можно «Выйти») и обработайте её нажатие.
2. Добавьте `EditText` для возможности редактирования имени прямо на экране (не обязательно, если успеваете).
3. Измените цвета, шрифты, добавьте тени, используя `android:elevation`.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной доработки:

1. **Профиль с контактами**  
   Добавьте под статусом информацию о телефоне и email (два `TextView` с иконками). Используйте `LinearLayout` горизонтальный внутри `ConstraintLayout`.

2. **Редактирование по нажатию**  
   Сделайте так, чтобы по нажатию на кнопку «Редактировать» имя и статус заменялись на `EditText` с возможностью ввода, а кнопка меняла текст на «Сохранить». При сохранении данные обновляются.

3. **Профиль с фотографией из галереи**  
   Добавьте возможность нажатия на аватар, чтобы открыть галерею и выбрать новое изображение (требуется разрешение и intent – сложно, но можно упростить: просто показать Toast).

---

## 5. Контрольные вопросы

1. Для чего используется `ConstraintLayout`? Какие у него преимущества перед `LinearLayout`?
2. Что такое `app:layout_constraint...` атрибуты?
3. Как вынести размеры и цвета в ресурсы? Зачем это нужно?
4. Каким образом можно обработать клик на кнопке в Kotlin-коде?
5. Как добавить обработчик нажатия на `ImageView`?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинг файла `activity_main.xml` (итоговый).
- Листинг `MainActivity.kt`.
- Скриншот работающего приложения на эмуляторе/устройстве.
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Элементы не отображаются или съехали** – проверьте все `layout_constraint` атрибуты. У каждого виджета должны быть как минимум привязки по горизонтали и вертикали (например, `left` и `right` или `start` и `end`).
- **Ресурс не найден** – убедитесь, что имя файла и идентификатор написаны правильно. После добавления новых ресурсов выполните `Build -> Clean Project` и `Rebuild`.
- **Кнопка не реагирует на нажатие** – проверьте, что `setOnClickListener` назначен и нет синтаксических ошибок.
- **Изображение слишком большое/маленькое** – используйте размеры в `dimens.xml` и при необходимости укажите `android:scaleType` для `ImageView`.

---

**Успешной работы!**
