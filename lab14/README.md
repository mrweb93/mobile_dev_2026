# Лабораторная работа №14
## Маппинг JSON ответа в Data классы. Вывод данных из сети в LazyColumn

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться интегрировать сетевые запросы в Jetpack Compose приложение, выполнять маппинг JSON-ответов в Kotlin Data классы с помощью Retrofit и отображать полученные данные в `LazyColumn`.

---

## 1. Теоретическая справка

### 1.1. Retrofit и JSON маппинг
Как и в лабораторной работе №13, для сетевых запросов используется библиотека **Retrofit**. Она автоматически преобразует JSON-ответы в объекты Kotlin с помощью конвертера (например, `GsonConverterFactory`). Для этого необходимо создать Data класс, поля которого соответствуют полям JSON.

Пример Data класса:
```kotlin
data class Post(
    val id: Int,
    val title: String,
    val body: String
)
```

### 1.2. Jetpack Compose и LazyColumn
**LazyColumn** — это аналог RecyclerView в Compose. Он лениво загружает только видимые элементы, что обеспечивает высокую производительность для больших списков .

Базовое использование:
```kotlin
LazyColumn {
    items(items = postList) { post ->
        PostItem(post = post)
    }
}
```

### 1.3. Управление состоянием в Compose
Для асинхронной загрузки данных удобно использовать ViewModel с `StateFlow` (или `mutableStateOf`). В Compose подписка на `StateFlow` осуществляется через `collectAsState()` или `collectAsStateWithLifecycle()` (рекомендуется) .

Пример в ViewModel:
```kotlin
private val _uiState = MutableStateFlow<PostsUiState>(PostsUiState.Loading)
val uiState: StateFlow<PostsUiState> = _uiState.asStateFlow()
```

В Composable функции:
```kotlin
val uiState by viewModel.uiState.collectAsStateWithLifecycle()
```

### 1.4. Отображение состояний
С помощью sealed class можно элегантно обрабатывать различные состояния UI (загрузка, успех, ошибка) и отображать соответствующие компоненты.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с установленным SDK.
- Эмулятор или реальное устройство с доступом в интернет.

---

## 3. Порядок выполнения работы

### Этап 1. Создание нового проекта с Compose (5 мин)

Создайте новый проект с шаблоном **Empty Activity** и обязательно выберите **Jetpack Compose** в качестве UI toolkit:
- **Name:** `PostsComposeApp`
- **Package name:** `com.example.postscompose`
- **Language:** Kotlin
- **Minimum SDK:** API 24
- **UI toolkit:** Compose

### Этап 2. Добавление зависимостей (5 мин)

Откройте файл `app/build.gradle.kts` (Module) и добавьте зависимости для сетевых запросов и жизненного цикла:

```kotlin
dependencies {
    // ... существующие зависимости Compose

    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
}
```

Выполните синхронизацию проекта.

### Этап 3. Создание модели данных (5 мин)

Создайте пакет `model` и внутри него data class `Post`:

```kotlin
package com.example.postscompose.model

data class Post(
    val id: Int,
    val title: String,
    val body: String
)
```

### Этап 4. Создание API интерфейса (5 мин)

Создайте пакет `api` и интерфейс `ApiService`:

```kotlin
package com.example.postscompose.api

import com.example.postscompose.model.Post
import retrofit2.http.GET

interface ApiService {
    @GET("posts")
    suspend fun getPosts(): List<Post>
}
```

### Этап 5. Создание Retrofit клиента (5 мин)

В том же пакете создайте объект `RetrofitClient`:

```kotlin
package com.example.postscompose.api

import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {
    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"

    val instance: ApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ApiService::class.java)
    }
}
```

### Этап 6. Создание репозитория (5 мин)

Создайте пакет `repository` и класс `PostsRepository`:

```kotlin
package com.example.postscompose.repository

import com.example.postscompose.api.RetrofitClient
import com.example.postscompose.model.Post
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext

class PostsRepository {
    private val api = RetrofitClient.instance

    suspend fun getPosts(): List<Post> = withContext(Dispatchers.IO) {
        api.getPosts()
    }
}
```

### Этап 7. Создание ViewModel с состоянием (10 мин)

Создайте пакет `viewmodel`. Внутри создайте sealed class `PostsUiState` и класс `PostsViewModel`:

```kotlin
package com.example.postscompose.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.postscompose.model.Post
import com.example.postscompose.repository.PostsRepository
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

### Этап 8. Создание Composable компонента для отображения поста (5 мин)

Создайте файл `PostItem.kt` (или добавьте в основной файл) с функцией `PostItem`:

```kotlin
package com.example.postscompose.ui

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.material3.Card
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import com.example.postscompose.model.Post

@Composable
fun PostItem(post: Post) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 8.dp, vertical = 4.dp)
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            Text(
                text = "ID: ${post.id}",
                style = MaterialTheme.typography.labelMedium
            )
            Text(
                text = post.title,
                style = MaterialTheme.typography.titleLarge
            )
            Text(
                text = post.body,
                style = MaterialTheme.typography.bodyMedium,
                modifier = Modifier.padding(top = 4.dp)
            )
        }
    }
}
```

### Этап 9. Создание главного экрана (15 мин)

Обновите `MainActivity.kt` или создайте отдельный файл `PostsScreen.kt`:

```kotlin
package com.example.postscompose

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Button
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Scaffold
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.example.postscompose.ui.PostItem
import com.example.postscompose.viewmodel.PostsUiState
import com.example.postscompose.viewmodel.PostsViewModel

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                PostsScreen()
            }
        }
    }
}

@Composable
fun PostsScreen(viewModel: PostsViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            // Можно добавить TopAppBar с кнопкой обновления
        }
    ) { paddingValues ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
        ) {
            when (uiState) {
                is PostsUiState.Loading -> {
                    CircularProgressIndicator(
                        modifier = Modifier.align(Alignment.Center)
                    )
                }
                is PostsUiState.Success -> {
                    LazyColumn(
                        modifier = Modifier.fillMaxSize()
                    ) {
                        items(
                            items = (uiState as PostsUiState.Success).posts,
                            key = { post -> post.id }
                        ) { post ->
                            PostItem(post = post)
                        }
                    }
                }
                is PostsUiState.Error -> {
                    Column(
                        modifier = Modifier.align(Alignment.Center),
                        horizontalAlignment = Alignment.CenterHorizontally
                    ) {
                        Text(
                            text = "Ошибка: ${(uiState as PostsUiState.Error).message}",
                            color = MaterialTheme.colorScheme.error
                        )
                        Button(
                            onClick = { viewModel.loadPosts() },
                            modifier = Modifier.padding(top = 8.dp)
                        ) {
                            Text("Повторить")
                        }
                    }
                }
            }
        }
    }
}
```

### Этап 10. Запуск и тестирование (5 мин)

Запустите приложение на эмуляторе или устройстве. Убедитесь, что:
- При запуске отображается индикатор загрузки.
- Через несколько секунд появляется список постов.
- При возникновении ошибки (например, отключить интернет) отображается сообщение и кнопка "Повторить".
- При нажатии "Повторить" данные загружаются снова.

### Этап 11. Добавление кнопки обновления (10 мин)

Добавьте в `Scaffold` параметр `topBar` с `TopAppBar` и кнопкой обновления. Модифицируйте `PostsScreen`:

```kotlin
import androidx.compose.material3.TopAppBar
import androidx.compose.material3.IconButton
import androidx.compose.material3.Icon
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Refresh

// Внутри Scaffold:
topBar = {
    TopAppBar(
        title = { Text("Посты с JSONPlaceholder") },
        actions = {
            IconButton(onClick = { viewModel.loadPosts() }) {
                Icon(Icons.Default.Refresh, contentDescription = "Обновить")
            }
        }
    )
}
```

### Этап 12. Детальный экран (если останется время, опционально)

При клике на элемент можно открывать детальный экран. Для этого потребуется навигация. Можно предложить как дополнительное задание.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Детальный экран поста**  
   Добавьте навигацию: при клике на пост открывается новый экран с полным текстом поста и комментариями (используйте второй запрос к `/posts/{id}/comments`).

2. **Поиск по постам**  
   Добавьте `TextField` для фильтрации списка по заголовку. Фильтрацию выполняйте на клиенте (после загрузки всех постов).

3. **Бесконечная прокрутка (пагинация)**  
   Реализуйте загрузку постов порциями (например, по 20) с использованием параметров `_page` и `_limit` из JSONPlaceholder. При прокрутке к концу списка подгружайте следующую страницу.

4. **Кэширование с Room**  
   Сохраняйте загруженные посты в локальную базу данных Room. При запуске сначала показывайте кэшированные данные, а затем обновляйте с сервера.

---

## 5. Контрольные вопросы

1. Как происходит маппинг JSON-ответа в Data классы в Retrofit?
2. Для чего используется `collectAsStateWithLifecycle()` в Compose?
3. Чем LazyColumn отличается от обычного Column при отображении больших списков?
4. Как обрабатывать различные состояния UI (загрузка, успех, ошибка) в Compose?
5. Почему сетевые запросы нужно выполнять в фоновом потоке, и как это обеспечивается в Retrofit с корутинами?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги всех созданных файлов: `Post.kt`, `ApiService.kt`, `RetrofitClient.kt`, `PostsRepository.kt`, `PostsViewModel.kt`, `PostItem.kt`, `MainActivity.kt`.
- Скриншоты работающего приложения (список постов, состояние загрузки, состояние ошибки).
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Ошибка "Unable to create call adapter"** – убедитесь, что метод в интерфейсе объявлен как `suspend`.
- **Список не отображается, но загрузка проходит** – проверьте, что в `LazyColumn` используется правильный список и ключи.
- **При повороте экрана данные теряются** – используйте ViewModel, которая переживает поворот. В Compose это автоматически решается через `viewModel()`.
- **Ошибка парсинга JSON** – проверьте соответствие полей в Data class и JSON. Для отладки можно добавить логирование ответа через `Log.d()`.
- **Индикатор загрузки не исчезает** – убедитесь, что в `loadPosts()` состояние изменяется на `Success` или `Error` даже при возникновении исключения.

---

## 8. Дополнительные материалы

- [Документация Jetpack Compose по LazyColumn](https://developer.android.com/jetpack/compose/lists)
- [Руководство по сетевым запросам в Compose](https://developer.android.com/jetpack/compose/effects)
- [JSONPlaceholder](https://jsonplaceholder.typicode.com/)

**Успешной работы!**