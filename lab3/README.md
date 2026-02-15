# Лабораторная работа №3
## Реализация списка объектов с фильтрацией с использованием .map, .filter, .sortedBy

**Длительность:** 1 час 30 минут  
**Цель работы:** Изучить функциональные методы обработки коллекций в Kotlin (`filter`, `map`, `sortedBy`) на примере списка объектов и вывести результаты в интерфейс Android-приложения.

---

## 1. Теоретическая справка

### 1.1. Функции высшего порядка для коллекций
Kotlin предоставляет богатый набор функций для работы с коллекциями, которые принимают лямбда-выражения:

- **`filter`** – возвращает список, содержащий только элементы, удовлетворяющие условию.
  ```kotlin
  val numbers = listOf(1, 2, 3, 4, 5)
  val even = numbers.filter { it % 2 == 0 } // [2, 4]
  ```

- **`map`** – преобразует каждый элемент коллекции по заданному правилу, возвращая новый список.
  ```kotlin
  val numbers = listOf(1, 2, 3)
  val squares = numbers.map { it * it } // [1, 4, 9]
  ```

- **`sortedBy`** – возвращает список, отсортированный по возрастанию значения, возвращаемого селектором.
  ```kotlin
  val people = listOf(Person("Alice", 30), Person("Bob", 25))
  val sorted = people.sortedBy { it.age } // по возрасту
  ```

Эти функции можно комбинировать в цепочки:
```kotlin
val result = list
    .filter { it.price > 100 }
    .sortedBy { it.name }
    .map { it.name }
```

### 1.2. Лямбда-выражения
Лямбда – это анонимная функция, которая может быть передана как аргумент. В Kotlin синтаксис:
```kotlin
{ параметр -> тело }
```
Если параметр один, можно использовать неявное имя `it`.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом (можно использовать проект из лабораторной работы №1 или создать новый).
- Эмулятор или реальное устройство для запуска приложения.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект `MyFirstApp` (или создайте новый с Empty Activity). Убедитесь, что проект компилируется.

### Этап 2. Создание класса данных (10 мин)

Создайте data class `Product` в пакете `com.example.myfirstapp.models` (создайте пакет `models`, если его нет). Класс должен содержать поля:
- `name` (String)
- `category` (String)
- `price` (Double)
- `inStock` (Boolean) – наличие на складе.

```kotlin
package com.example.myfirstapp.models

data class Product(
    val name: String,
    val category: String,
    val price: Double,
    val inStock: Boolean
)
```

### Этап 3. Подготовка интерфейса (10 мин)

В файле `activity_main.xml` создайте простой интерфейс для отображения трёх списков:
- Исходный список (кратко)
- Отфильтрованные товары (в наличии)
- Отсортированные по цене и преобразованные в строки

Используйте `LinearLayout` (вертикальный) и несколько `TextView` с фиксированными id.

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
        android:text="Исходные товары:"
        android:textStyle="bold"
        android:textSize="18sp"/>

    <TextView
        android:id="@+id/textOriginal"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="16dp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Товары в наличии:"
        android:textStyle="bold"
        android:textSize="18sp"/>

    <TextView
        android:id="@+id/textInStock"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:layout_marginBottom="16dp"/>

    <TextView
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Товары после сортировки по цене (название и цена):"
        android:textStyle="bold"
        android:textSize="18sp"/>

    <TextView
        android:id="@+id/textSorted"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"/>

</LinearLayout>
```

### Этап 4. Создание списка товаров (10 мин)

В `MainActivity` создайте функцию, которая возвращает тестовый список товаров.

```kotlin
private fun getProducts(): List<Product> {
    return listOf(
        Product("Ноутбук", "Электроника", 75000.0, true),
        Product("Мышь", "Электроника", 1500.0, true),
        Product("Книга 'Котлин'", "Книги", 1200.0, false),
        Product("Флешка 64GB", "Электроника", 2000.0, true),
        Product("Блокнот", "Канцелярия", 300.0, true),
        Product("Ручка", "Канцелярия", 50.0, false),
        Product("Монитор", "Электроника", 25000.0, true)
    )
}
```

### Этап 5. Применение filter, map, sortedBy (20 мин)

В `MainActivity` внутри `onCreate` получите список товаров и примените цепочки обработки. Результаты выведите в соответствующие `TextView`.

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    setContentView(R.layout.activity_main)

    val products = getProducts()

    // 1. Исходный список (для наглядности преобразуем в строку)
    val originalText = products.joinToString("\n") { "${it.name} – ${it.price} руб. (${if (it.inStock) "в наличии" else "нет"})" }
    findViewById<TextView>(R.id.textOriginal).text = originalText

    // 2. Фильтр: только товары в наличии
    val inStockProducts = products.filter { it.inStock }
    val inStockText = inStockProducts.joinToString("\n") { "${it.name} – ${it.price} руб." }
    findViewById<TextView>(R.id.textInStock).text = inStockText

    // 3. Цепочка: отфильтровать электронику, отсортировать по цене и получить список строк с названием и ценой
    val electronicsSorted = products
        .filter { it.category == "Электроника" && it.inStock }
        .sortedBy { it.price }
        .map { "${it.name} – ${it.price} руб." }
    val electronicsText = electronicsSorted.joinToString("\n")
    findViewById<TextView>(R.id.textSorted).text = electronicsText
}
```

### Этап 6. Запуск приложения (10 мин)

Запустите приложение на эмуляторе или устройстве. Убедитесь, что все `TextView` заполнены корректными данными. Проверьте, что фильтрация и сортировка работают как ожидается.

### Этап 7. Эксперименты (оставшееся время)

Измените условия фильтрации или сортировки, добавьте ещё одну цепочку (например, отобразить все товары дешевле 2000 рублей, отсортированные по названию). Пронаблюдайте изменения.

---

## 4. Полный код MainActivity (для справки)

```kotlin
package com.example.myfirstapp

import androidx.appcompat.app.AppCompatActivity
import android.os.Bundle
import android.widget.TextView
import com.example.myfirstapp.models.Product

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val products = getProducts()

        // Исходный список
        val originalText = products.joinToString("\n") {
            "${it.name} – ${it.price} руб. (${if (it.inStock) "в наличии" else "нет"})"
        }
        findViewById<TextView>(R.id.textOriginal).text = originalText

        // Только в наличии
        val inStockProducts = products.filter { it.inStock }
        val inStockText = inStockProducts.joinToString("\n") { "${it.name} – ${it.price} руб." }
        findViewById<TextView>(R.id.textInStock).text = inStockText

        // Электроника в наличии, отсортированная по цене
        val electronicsSorted = products
            .filter { it.category == "Электроника" && it.inStock }
            .sortedBy { it.price }
            .map { "${it.name} – ${it.price} руб." }
        findViewById<TextView>(R.id.textSorted).text = electronicsSorted.joinToString("\n")
    }

    private fun getProducts(): List<Product> {
        return listOf(
            Product("Ноутбук", "Электроника", 75000.0, true),
            Product("Мышь", "Электроника", 1500.0, true),
            Product("Книга 'Котлин'", "Книги", 1200.0, false),
            Product("Флешка 64GB", "Электроника", 2000.0, true),
            Product("Блокнот", "Канцелярия", 300.0, true),
            Product("Ручка", "Канцелярия", 50.0, false),
            Product("Монитор", "Электроника", 25000.0, true)
        )
    }
}
```

---

## 5. Индивидуальные задания (вариативно)

Выберите одну из предметных областей и реализуйте аналогичную обработку списка:

1. **Список фильмов** (название, жанр, рейтинг, год выпуска).  
   - Показать фильмы с рейтингом выше 8.0.
   - Отсортировать их по году выпуска.
   - Вывести список названий и рейтингов.

2. **Список сотрудников** (имя, отдел, зарплата, стаж).  
   - Показать сотрудников с зарплатой больше 100000.
   - Отсортировать по стажу (по убыванию).
   - Вывести имена и отделы.

3. **Список книг** (название, автор, год, количество страниц).  
   - Показать книги, изданные после 2000 года.
   - Отсортировать по количеству страниц.
   - Вывести названия и авторов.

---

## 6. Контрольные вопросы

1. Что возвращает функция `filter` – новый список или изменяет существующий?
2. В чём разница между `sortedBy` и `sortedByDescending`?
3. Как можно объединить несколько условий в `filter`?
4. Для чего используется функция `map`? Приведите пример.
5. Что такое `joinToString` и как она работает?

---

## 7. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинг класса `Product` и `MainActivity` (с добавленными цепочками).
- Скриншот работающего приложения (с видимыми результатами фильтрации и сортировки).
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 8. Возможные ошибки и их решение

- **Приложение падает с NullPointerException** – проверьте, что все `TextView` имеют правильные id в разметке.
- **Не отображаются все товары** – убедитесь, что список не слишком велик для маленького экрана; можно поместить `TextView` в `ScrollView`.
- **Сортировка работает неправильно** – проверьте, что для чисел используется правильный тип (Double, Int). Для строк сортировка лексикографическая.
- **Не импортируются классы** – добавьте импорт `import com.example.myfirstapp.models.Product`.

---

**Успешной работы!**
