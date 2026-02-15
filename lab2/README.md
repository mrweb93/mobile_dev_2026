# Лабораторная работа №2
## Написание консольных утилит на Kotlin внутри Android проекта. Расчеты, работа со строками. Подготовка классов данных для будущего приложения

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться создавать классы данных и функции-утилиты на Kotlin в контексте Android-проекта, освоить базовые приёмы работы со строками и числами, познакомиться с юнит-тестированием для проверки корректности кода.

---

## 1. Теоретическая справка

### 1.1. Классы данных (data class) в Kotlin
Классы данных предназначены для хранения состояния. Компилятор автоматически генерирует для них полезные методы: `toString()`, `equals()`, `hashCode()`, `copy()`.

```kotlin
data class Book(
    val title: String,
    val author: String,
    val year: Int,
    val price: Double
)
```

### 1.2. Работа со строками
Kotlin предоставляет удобные функции для работы со строками: `split`, `substring`, `trim`, шаблоны (`"текст ${переменная}"`), функции проверки (`isNullOrBlank`, `contains`).

### 1.3. Функции расширения (extension functions)
Позволяют добавлять новые функции к существующим классам.

```kotlin
fun String.isEmailValid(): Boolean {
    return this.contains("@") && this.contains(".")
}
```

### 1.4. Юнит-тестирование (JUnit)
Для проверки работоспособности кода без запуска всего приложения используются тесты. Тесты помещаются в директорию `src/test/java` и запускаются на JVM.

Пример теста:
```kotlin
import org.junit.Assert.*
import org.junit.Test

class UtilsTest {
    @Test
    fun testSum() {
        assertEquals(4, 2 + 2)
    }
}
```

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio (проект, созданный в лабораторной работе №1).
- Эмулятор или реальное устройство не требуется – тесты выполняются на компьютере.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `MyFirstApp`, созданный на прошлом занятии. Убедитесь, что проект успешно собирается. В структуре проекта обратите внимание на наличие папки `app/src/test/java` – в ней будут размещаться тесты.

### Этап 2. Создание пакета для утилит (5 мин)

В папке `app/src/main/java/com/example/myfirstapp` создайте новый пакет `utils` (правой кнопкой на `com.example.myfirstapp` → New → Package). В этом пакете будут храниться классы данных и вспомогательные функции.

### Этап 3. Создание класса данных (10 мин)

Создайте класс данных `Book` в пакете `utils` (New → Kotlin Class/File → выберите Data Class).

```kotlin
package com.example.myfirstapp.utils

data class Book(
    val title: String,
    val author: String,
    val year: Int,
    val price: Double
)
```

### Этап 4. Написание функций-утилит (15 мин)

В том же пакете создайте файл `StringUtils.kt` (New → Kotlin Class/File → File) и добавьте функции для работы со строками.

```kotlin
package com.example.myfirstapp.utils

// Проверка, что строка похожа на email (содержит @ и .)
fun String.isValidEmail(): Boolean {
    return this.contains("@") && this.contains(".")
}

// Форматирование автора: "Толстой Л.Н."
fun formatAuthorName(fullName: String): String {
    val parts = fullName.split(" ").filter { it.isNotBlank() }
    return when (parts.size) {
        1 -> parts[0]  // только фамилия
        2 -> "${parts[0]} ${parts[1].first()}."  // фамилия и инициал
        3 -> "${parts[0]} ${parts[1].first()}.${parts[2].first()}."  // фамилия и два инициала
        else -> fullName
    }
}

// Применение скидки к цене книги
fun applyDiscount(price: Double, discountPercent: Double): Double {
    require(discountPercent in 0.0..100.0) { "Скидка должна быть от 0 до 100" }
    return price * (1 - discountPercent / 100)
}
```

### Этап 5. Написание юнит-тестов (25 мин)

Теперь напишем тесты для проверки созданных функций.

1. Откройте папку `app/src/test/java`. Создайте в ней пакет с тем же именем (`com.example.myfirstapp.utils`).
2. В этом пакете создайте новый Kotlin класс `StringUtilsTest` (New → Kotlin Class/File → Class).

Добавьте следующие тесты:

```kotlin
package com.example.myfirstapp.utils

import org.junit.Assert.*
import org.junit.Test

class StringUtilsTest {

    @Test
    fun emailValidation_correct() {
        assertTrue("test@example.com".isValidEmail())
        assertTrue("user.name@domain.co".isValidEmail())
    }

    @Test
    fun emailValidation_incorrect() {
        assertFalse("testexample.com".isValidEmail())
        assertFalse("test@example".isValidEmail())
        assertFalse("".isValidEmail())
    }

    @Test
    fun formatAuthorName_fullName() {
        assertEquals("Толстой Л.Н.", formatAuthorName("Толстой Лев Николаевич"))
        assertEquals("Пушкин А.С.", formatAuthorName("Пушкин Александр Сергеевич"))
    }

    @Test
    fun formatAuthorName_twoParts() {
        assertEquals("Толстой Л.", formatAuthorName("Толстой Лев"))
        assertEquals("Пушкин А.", formatAuthorName("Пушкин Александр"))
    }

    @Test
    fun formatAuthorName_onePart() {
        assertEquals("Толстой", formatAuthorName("Толстой"))
    }

    @Test
    fun applyDiscount_normal() {
        assertEquals(90.0, applyDiscount(100.0, 10.0), 0.001)
        assertEquals(75.0, applyDiscount(150.0, 50.0), 0.001)
    }

    @Test
    fun applyDiscount_zero() {
        assertEquals(100.0, applyDiscount(100.0, 0.0), 0.001)
    }

    @Test(expected = IllegalArgumentException::class)
    fun applyDiscount_invalidLow() {
        applyDiscount(100.0, -5.0)
    }

    @Test(expected = IllegalArgumentException::class)
    fun applyDiscount_invalidHigh() {
        applyDiscount(100.0, 110.0)
    }
}
```

### Этап 6. Запуск тестов и анализ результатов (10 мин)

1. Чтобы запустить все тесты, щелкните правой кнопкой мыши на папке `src/test/java` и выберите **Run Tests** (или на конкретном файле теста).
2. Внизу откроется окно **Run** с результатами. Все тесты должны быть зелёными (успешными).
3. Если какой-то тест упал, проанализируйте ошибку и исправьте код функций.

### Этап 7. Интеграция в Android-приложение (опционально, 10 мин)

Для наглядности можно вывести результаты работы утилит в интерфейс приложения. Добавьте в `activity_main.xml` несколько `TextView` и в `MainActivity` вызовите утилиты, установив текст.

Пример кода в `MainActivity.kt` (после `setContentView`):

```kotlin
val book = Book("Война и мир", "Толстой Лев Николаевич", 1869, 500.0)
val formattedAuthor = formatAuthorName(book.author)
val discountedPrice = applyDiscount(book.price, 15.0)

findViewById<TextView>(R.id.textView1).text = "Книга: ${book.title}"
findViewById<TextView>(R.id.textView2).text = "Автор: $formattedAuthor"
findViewById<TextView>(R.id.textView3).text = "Цена со скидкой: $discountedPrice руб."
```

Не забудьте добавить `import` и соответствующие `TextView` в разметку с id `textView1`, `textView2`, `textView3`.

Запустите приложение на эмуляторе, чтобы увидеть результат.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной работы:

1. **Класс "Студент"**  
   Создайте data class `Student` (имя, фамилия, группа, средний балл). Напишите функцию, которая возвращает строку вида "Иванов И. (группа ПИ-101)" и функцию для определения статуса (отличник/хорошист/троечник) на основе среднего балла. Покройте тестами.

2. **Конвертер валют**  
   Напишите функцию конвертации рублей в доллары и евро по заданному курсу. Создайте класс `CurrencyConverter` с методами `rubToUsd`, `usdToRub` и т.д. Курсы передавайте как параметры. Напишите тесты.

3. **Валидатор пароля**  
   Напишите функцию, проверяющую сложность пароля: длина не менее 8 символов, наличие цифр, заглавных букв и спецсимволов. Возвращайте строку с описанием ошибки или "Пароль надёжный". Покройте тестами.

---

## 5. Контрольные вопросы

1. Для чего в Kotlin используются data class?
2. Чем отличается функция расширения от обычной функции?
3. Как запустить юнит-тесты в Android Studio?
4. Что такое `assertEquals` и для чего нужен третий параметр (дельта) при сравнении вещественных чисел?
5. В какой директории проекта хранятся тесты, выполняющиеся на JVM?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги созданных классов данных и функций-утилит.
- Листинги юнит-тестов.
- Скриншот успешного выполнения тестов (зелёная полоса).
- (Если выполнялась интеграция с UI) скриншот приложения с отображением результатов.
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Тесты не компилируются, не видят классы из основного кода** – убедитесь, что в тестовом файле указан правильный импорт пакета (например, `import com.example.myfirstapp.utils.*`).
- **Ошибка "JUnit not found"** – проверьте, что в файле `build.gradle` (Module) есть зависимость `testImplementation 'junit:junit:4.13.2'`. Если её нет, добавьте и сделайте Sync.
- **Тесты с вещественными числами падают из-за погрешности** – используйте `assertEquals(expected, actual, delta)`, где delta – допустимая погрешность (например, 0.001).
- **Функция `require` выбрасывает исключение, но тест не ловит** – убедитесь, что в тесте используется атрибут `expected` в аннотации `@Test(expected = IllegalArgumentException::class)`.

---

**Успешной работы!**