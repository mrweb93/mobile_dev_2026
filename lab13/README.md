# Лабораторная работа №13
## Создание простого API клиента. Запрос списка постов с jsonplaceholder.typicode.com

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться выполнять сетевые запросы в Android-приложении с использованием библиотеки Retrofit и корутин, обрабатывать ответы сервера, парсить JSON-данные и отображать их в RecyclerView.

---

## 1. Теоретическая справка

### 1.1. REST API и JSONPlaceholder
**REST API** — это архитектурный стиль взаимодействия компонентов распределённого приложения. В основе REST лежат HTTP-методы: GET (получение данных), POST (создание), PUT/PATCH (обновление), DELETE (удаление) .

**JSONPlaceholder** — это бесплатный тестовый REST API, который предоставляет фейковые данные для прототипирования и тестирования . Он содержит следующие ресурсы:
- `/posts` — 100 постов
- `/comments` — 500 комментариев
- `/albums` — 100 альбомов
- `/photos` — 5000 фотографий
- `/todos` — 200 задач
- `/users` — 10 пользователей

В этой лабораторной мы будем работать с ресурсом `/posts`, который возвращает массив объектов с полями `id`, `title`, `body` .

### 1.2. Retrofit
**Retrofit** — это типобезопасный HTTP-клиент для Android и Java, разработанный компанией Square. Он позволяет декларативно описывать API с помощью аннотаций и автоматически преобразовывать JSON-ответы в Kotlin-объекты .

Основные компоненты Retrofit:
- **Интерфейс API** — определяет методы запросов с аннотациями (`@GET`, `@POST` и т.д.)
- **Retrofit.Builder** — создаёт экземпляр Retrofit с настройками (baseUrl, конвертер)
- **Конвертер** — преобразует JSON в объекты (например, `GsonConverterFactory`)

### 1.3. Корутины для сетевых запросов
Сетевые запросы должны выполняться в фоновом потоке, чтобы не блокировать UI. В Kotlin для этого используются корутины .

Ключевые понятия:
- `suspend fun` — функция, которая может приостанавливаться без блокировки потока
- `Dispatchers.IO` — планировщик для операций ввода-вывода (сеть, диск)
- `viewModelScope` — область корутин, привязанная к жизненному циклу ViewModel 
- `withContext(Dispatchers.IO)` — переключение контекста для выполнения блока в фоновом потоке 

### 1.4. Модель данных (Entity)
Для представления данных, полученных от сервера, создаётся data class. Поля класса должны соответствовать полям JSON-объекта .

Пример для поста:
```kotlin
data class Post(
    val id: Int,
    val title: String,
    val body: String
)
```

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio (проект можно создать новый или продолжить существующий).
- Эмулятор или реальное устройство с доступом в интернет.

---

## 3. Порядок выполнения работы

### Этап 1. Создание нового проекта (5 мин)

Создайте новый проект с шаблоном **Empty Views Activity**:
- **Name:** `PostsApp`
- **Package name:** `com.example.postsapp`
- **Language:** Kotlin
- **Minimum SDK:** API 24

### Этап 2. Добавление зависимостей (5 мин)

Откройте файл `app/build.gradle.kts` (Module) и добавьте зависимости для сетевых запросов и парсинга JSON :

```kotlin
dependencies {
    // ... существующие зависимости

    // Retrofit
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")

    // Корутины
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")

    // ViewModel и Lifecycle (если ещё не добавлены)
    implementation("androidx.lifecycle:lifecycle-viewmodel-ktx:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.7.0")
}
```

Выполните синхронизацию проекта (Sync Now).

### Этап 3. Создание модели данных (5 мин)

Создайте пакет `models`. В нём создайте data class `Post` :

```kotlin
package com.example.postsapp.models

data class Post(
    val id: Int,
    val title: String,
    val body: String
)
```

### Этап 4. Создание API интерфейса (10 мин)

Создайте пакет `api`. В нём создайте интерфейс `ApiService` с методом для получения списка постов :

```kotlin
package com.example.postsapp.api

import com.example.postsapp.models.Post
import retrofit2.http.GET

interface ApiService {
    @GET("posts")
    suspend fun getPosts(): List<Post>
}
```

**Пояснения:**
- `@GET("posts")` указывает, что это GET-запрос к ресурсу `/posts` .
- `suspend` означает, что функция может вызываться только из корутины и выполняется асинхронно .

### Этап 5. Создание Retrofit клиента (10 мин)

В том же пакете `api` создайте объект `RetrofitClient`, который будет предоставлять экземпляр `ApiService` :

```kotlin
package com.example.postsapp.api

import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {
    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"

    private val retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .addConverterFactory(GsonConverterFactory.create())
        .build()

    val apiService: ApiService = retrofit.create(ApiService::class.java)
}
```

**Важно:** URL должен заканчиваться на `/` .

### Этап 6. Создание репозитория (10 мин)

Создайте пакет `repositories`. В нём создайте класс `PostsRepository`, который будет использовать `ApiService` для получения данных :

```kotlin
package com.example.postsapp.repositories

import com.example.postsapp.api.RetrofitClient
import com.example.postsapp.models.Post
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

class PostsRepository {
    private val apiService = RetrofitClient.apiService

    suspend fun getPosts(): List<Post> = withContext(Dispatchers.IO) {
        apiService.getPosts()
    }
}
```

**Пояснения:**
- `withContext(Dispatchers.IO)` переключает выполнение на поток для ввода-вывода .
- Репозиторий скрывает источник данных от ViewModel.

### Этап 7. Создание ViewModel (10 мин)

Создайте пакет `viewmodels`. В нём создайте `PostsViewModel`, который будет управлять состоянием UI :

```kotlin
package com.example.postsapp.viewmodels

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.postsapp.models.Post
import com.example.postsapp.repositories.PostsRepository
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

sealed class PostsUiState {
    object Loading : PostsUiState()
    data class Success(val posts: List<Post>) : PostsUiState()
    data class Error(val message: String) : PostsUiState()
}

class PostsViewModel : ViewModel() {
    private val repository = PostsRepository()
    
    private val _uiState = MutableStateFlow<PostsUiState>(PostsUiState.Loading)
    val uiState: StateFlow<PostsUiState> = _uiState.asStateFlow()

    init {
        loadPosts()
    }

    fun loadPosts() {
        viewModelScope.launch {
            _uiState.value = PostsUiState.Loading
            try {
                val posts = repository.getPosts()
                _uiState.value = PostsUiState.Success(posts)
            } catch (e: Exception) {
                _uiState.value = PostsUiState.Error(e.message ?: "Unknown error")
            }
        }
    }
}
```

### Этап 8. Создание адаптера для RecyclerView (10 мин)

Создайте адаптер для отображения списка постов. Сначала создайте layout для элемента списка `item_post.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.cardview.widget.CardView
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:layout_margin="8dp"
    app:cardCornerRadius="8dp"
    app:cardElevation="4dp">

    <LinearLayout
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        android:padding="16dp">

        <TextView
            android:id="@+id/textPostId"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="ID: "
            android:textStyle="bold"
            android:textSize="14sp"/>

        <TextView
            android:id="@+id/textPostTitle"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Title"
            android:textSize="18sp"
            android:textStyle="bold"
            android:layout_marginTop="4dp"/>

        <TextView
            android:id="@+id/textPostBody"
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:text="Body"
            android:textSize="14sp"
            android:layout_marginTop="8dp"/>

    </LinearLayout>
</androidx.cardview.widget.CardView>
```

Создайте класс `PostsAdapter`:

```kotlin
package com.example.postsapp.adapters

import android.view.LayoutInflater
import android.view.ViewGroup
import androidx.recyclerview.widget.RecyclerView
import com.example.postsapp.databinding.ItemPostBinding
import com.example.postsapp.models.Post

class PostsAdapter : RecyclerView.Adapter<PostsAdapter.PostViewHolder>() {
    
    private var posts = emptyList<Post>()
    
    fun submitList(newPosts: List<Post>) {
        posts = newPosts
        notifyDataSetChanged()
    }
    
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): PostViewHolder {
        val binding = ItemPostBinding.inflate(LayoutInflater.from(parent.context), parent, false)
        return PostViewHolder(binding)
    }
    
    override fun onBindViewHolder(holder: PostViewHolder, position: Int) {
        holder.bind(posts[position])
    }
    
    override fun getItemCount() = posts.size
    
    inner class PostViewHolder(private val binding: ItemPostBinding) : 
        RecyclerView.ViewHolder(binding.root) {
        
        fun bind(post: Post) {
            binding.textPostId.text = "ID: ${post.id}"
            binding.textPostTitle.text = post.title
            binding.textPostBody.text = post.body
        }
    }
}
```

### Этап 9. Верстка главного экрана (5 мин)

Обновите `activity_main.xml`:

```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <Button
        android:id="@+id/buttonRefresh"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Обновить"/>

    <FrameLayout
        android:layout_width="match_parent"
        android:layout_height="match_parent">

        <androidx.recyclerview.widget.RecyclerView
            android:id="@+id/recyclerViewPosts"
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

</LinearLayout>
```

### Этап 10. Реализация MainActivity (15 мин)

В `MainActivity.kt` реализуйте подписку на состояние и отображение данных :

```kotlin
package com.example.postsapp

import android.os.Bundle
import android.widget.Button
import android.widget.ProgressBar
import android.widget.TextView
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import com.example.postsapp.adapters.PostsAdapter
import com.example.postsapp.viewmodels.PostsUiState
import com.example.postsapp.viewmodels.PostsViewModel
import kotlinx.coroutines.launch

class MainActivity : AppCompatActivity() {

    private val viewModel: PostsViewModel by viewModels()
    private lateinit var adapter: PostsAdapter
    private lateinit var recyclerView: RecyclerView
    private lateinit var progressBar: ProgressBar
    private lateinit var textError: TextView
    private lateinit var buttonRefresh: Button

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        recyclerView = findViewById(R.id.recyclerViewPosts)
        progressBar = findViewById(R.id.progressBar)
        textError = findViewById(R.id.textError)
        buttonRefresh = findViewById(R.id.buttonRefresh)

        setupRecyclerView()
        observeUiState()

        buttonRefresh.setOnClickListener {
            viewModel.loadPosts()
        }
    }

    private fun setupRecyclerView() {
        adapter = PostsAdapter()
        recyclerView.layoutManager = LinearLayoutManager(this)
        recyclerView.adapter = adapter
    }

    private fun observeUiState() {
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is PostsUiState.Loading -> showLoading()
                        is PostsUiState.Success -> showPosts(state.posts)
                        is PostsUiState.Error -> showError(state.message)
                    }
                }
            }
        }
    }

    private fun showLoading() {
        recyclerView.visibility = android.view.View.GONE
        progressBar.visibility = android.view.View.VISIBLE
        textError.visibility = android.view.View.GONE
    }

    private fun showPosts(posts: List<com.example.postsapp.models.Post>) {
        recyclerView.visibility = android.view.View.VISIBLE
        progressBar.visibility = android.view.View.GONE
        textError.visibility = android.view.View.GONE
        adapter.submitList(posts)
    }

    private fun showError(message: String) {
        recyclerView.visibility = android.view.View.GONE
        progressBar.visibility = android.view.View.GONE
        textError.visibility = android.view.View.VISIBLE
        textError.text = "Ошибка: $message"
    }
}
```

### Этап 11. Запуск и тестирование (5 мин)

Запустите приложение. Убедитесь, что:
- При запуске отображается ProgressBar.
- Через несколько секунд появляется список постов (заголовки и текст).
- При нажатии кнопки "Обновить" данные загружаются снова.
- При отключении интернета (можно включить авиарежим) отображается ошибка.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Детальный экран поста**  
   При клике на элемент списка открывайте новый экран с полной информацией о посте (id, title, body). Используйте Intent для передачи данных.

2. **Запрос комментариев**  
   Создайте новую модель `Comment` (id, postId, name, email, body) и метод в API для получения комментариев к посту (`/posts/{id}/comments`). Отображайте их на детальном экране.

3. **Бесконечная прокрутка**  
   Реализуйте пагинацию: API JSONPlaceholder поддерживает параметры `_page` и `_limit`. Загружайте по 20 постов и подгружайте следующие при прокрутке.

4. **Сохранение в БД**  
   Добавьте Room и сохраняйте загруженные посты в базу данных. При запуске сначала показывайте кэшированные данные, а затем обновляйте с сервера.

---

## 5. Контрольные вопросы

1. Для чего используется библиотека Retrofit? Какие аннотации вы знаете?
2. Почему сетевые запросы нельзя выполнять в главном потоке?
3. Что такое `suspend` функция и как она работает с корутинами? 
4. Для чего нужен `Dispatchers.IO`? 
5. Как обрабатывать ошибки при сетевых запросах?
6. Что такое JSONPlaceholder и для чего он используется? 

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги всех созданных файлов: `Post.kt`, `ApiService.kt`, `RetrofitClient.kt`, `PostsRepository.kt`, `PostsViewModel.kt`, `PostsAdapter.kt`, `MainActivity.kt`, `activity_main.xml`, `item_post.xml`.
- Скриншоты работающего приложения (список постов, состояние загрузки, состояние ошибки).
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Ошибка "Unable to create call adapter"** – убедитесь, что метод в интерфейсе объявлен как `suspend`. Без этого Retrofit не сможет адаптировать вызов для корутин.
- **Пустой список** – проверьте URL (должен заканчиваться на `/`). Проверьте доступ к интернету. Добавьте логирование через `Log.d()` в ViewModel.
- **Ошибка "NetworkOnMainThreadException"** – убедитесь, что вызов API выполняется в корутине с `Dispatchers.IO` или через `suspend` функцию.
- **Ошибка парсинга JSON** – проверьте соответствие полей в data class полям в JSON-ответе. Используйте аннотацию `@SerializedName` если имена отличаются.
- **Приложение падает при повороте экрана** – это нормально, так как ViewModel переживает поворот. Если падает – проверьте инициализацию в `onCreate`.

---

## 8. Дополнительные материалы

- [Официальная документация JSONPlaceholder](https://jsonplaceholder.typicode.com/) 
- [Документация Retrofit](https://square.github.io/retrofit/)
- [Корутины в Android (официальная документация)](https://developer.android.com/kotlin/coroutines) 

**Успешной работы!**
