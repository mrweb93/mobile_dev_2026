# Лабораторная работа №15
## Экран с карточками товаров/фильмов. Загрузка картинок по URL

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться загружать и отображать изображения из сети в приложении Jetpack Compose, используя библиотеку Coil, интегрировать полученные навыки с предыдущими лабораторными работами для создания экрана с карточками объектов (товаров, фильмов, постов с картинками).

---

## 1. Теоретическая справка

### 1.1. Загрузка изображений в Android
Для эффективной загрузки изображений из сети в Android используются специализированные библиотеки, которые решают следующие задачи:
- Кэширование (память и диск)
- Управление памятью (избегание OutOfMemoryError)
- Асинхронная загрузка
- Обработка placeholder и ошибок

Наиболее популярные библиотеки:
- **Glide** – мощная и гибкая библиотека, хорошо интегрируется с XML и Compose
- **Coil** – современная библиотека, написанная на Kotlin, с отличной поддержкой корутин и Compose (официально рекомендована для Compose)
- **Picasso** – простая и лёгкая

В этой лабораторной работе мы будем использовать **Coil**, так как он оптимально подходит для Compose и корутин.

### 1.2. Coil в Compose
Coil предоставляет composable функцию `AsyncImage`, которая загружает изображение по URL и отображает его. Основные параметры:
- `model` – источник изображения (URL, Uri, File и т.д.)
- `contentDescription` – описание для accessibility
- `placeholder` – composable, отображаемый во время загрузки
- `error` – composable, отображаемый при ошибке
- `modifier` – модификаторы размера, формы и т.д.

Пример:
```kotlin
AsyncImage(
    model = "https://example.com/image.jpg",
    contentDescription = "Product image",
    modifier = Modifier.size(100.dp),
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error)
)
```

### 1.3. Получение данных из сети
Как и в предыдущих работах, мы будем использовать Retrofit для получения списка объектов. Каждый объект будет содержать идентификатор или прямую ссылку на изображение. Для демонстрации можно использовать публичное API, например:
- [JSONPlaceholder](https://jsonplaceholder.typicode.com/posts) – возвращает посты (id, title, body)
- Для изображений используем сервис [Picsum](https://picsum.photos/), который генерирует случайные изображения по ID: `https://picsum.photos/id/{id}/200/200`

Таким образом, для каждого поста мы можем сформировать URL картинки: `https://picsum.photos/id/${post.id}/200/200`

### 1.4. Отображение в LazyColumn
Для отображения списка карточек используем `LazyColumn`. Каждый элемент будет представлять собой карточку (`Card`), содержащую изображение и текстовую информацию.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с установленным SDK.
- Эмулятор или реальное устройство с доступом в интернет.

---

## 3. Порядок выполнения работы

### Этап 1. Создание нового проекта (5 мин)

Создайте новый проект с шаблоном **Empty Activity** и выберите **Jetpack Compose**:
- **Name:** `ImageCardApp`
- **Package name:** `com.example.imagecardapp`
- **Language:** Kotlin
- **Minimum SDK:** API 24

### Этап 2. Добавление зависимостей (5 мин)

Откройте файл `app/build.gradle.kts` (Module) и добавьте зависимости для сетевых запросов и загрузки изображений:

```kotlin
dependencies {
    // ... существующие зависимости Compose

    // Retrofit
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")

    // Coil для Compose
    implementation("io.coil-kt:coil-compose:2.5.0")

    // ViewModel для Compose
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
}
```

Выполните синхронизацию проекта.

### Этап 3. Создание модели данных (5 мин)

Создайте пакет `model` и внутри него data class `Post`. Обратите внимание, что изображение мы будем формировать динамически, поэтому в модели храним только `id`, `title`, `body`.

```kotlin
package com.example.imagecardapp.model

data class Post(
    val id: Int,
    val title: String,
    val body: String
)
```

### Этап 4. Создание API интерфейса (5 мин)

Создайте пакет `api` и интерфейс `ApiService`:

```kotlin
package com.example.imagecardapp.api

import com.example.imagecardapp.model.Post
import retrofit2.http.GET

interface ApiService {
    @GET("posts")
    suspend fun getPosts(): List<Post>
}
```

### Этап 5. Создание Retrofit клиента (5 мин)

В том же пакете создайте объект `RetrofitClient`:

```kotlin
package com.example.imagecardapp.api

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
package com.example.imagecardapp.repository

import com.example.imagecardapp.api.RetrofitClient
import com.example.imagecardapp.model.Post
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
package com.example.imagecardapp.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.imagecardapp.model.Post
import com.example.imagecardapp.repository.PostsRepository
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

### Этап 8. Создание компонента карточки (10 мин)

Создайте файл `PostCard.kt` (или добавьте в основной файл) с функцией `PostCard`:

```kotlin
package com.example.imagecardapp.ui

import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.Row
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.layout.size
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material3.Card
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.ui.Modifier
import androidx.compose.ui.draw.clip
import androidx.compose.ui.layout.ContentScale
import androidx.compose.ui.unit.dp
import coil.compose.AsyncImage
import com.example.imagecardapp.model.Post

@Composable
fun PostCard(post: Post) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 8.dp, vertical = 4.dp)
    ) {
        Row(
            modifier = Modifier.padding(8.dp)
        ) {
            // Изображение
            AsyncImage(
                model = "https://picsum.photos/id/${post.id}/200/200",
                contentDescription = "Post image",
                modifier = Modifier
                    .size(100.dp)
                    .clip(RoundedCornerShape(8.dp)),
                contentScale = ContentScale.Crop
            )
            
            // Текстовая информация
            Column(
                modifier = Modifier
                    .padding(start = 8.dp)
                    .weight(1f)
            ) {
                Text(
                    text = post.title,
                    style = MaterialTheme.typography.titleMedium
                )
                Text(
                    text = post.body,
                    style = MaterialTheme.typography.bodySmall,
                    maxLines = 3
                )
            }
        }
    }
}
```

### Этап 9. Создание главного экрана (15 мин)

Обновите `MainActivity.kt`:

```kotlin
package com.example.imagecardapp

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
import androidx.compose.material3.TopAppBar
import androidx.compose.material3.IconButton
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Refresh
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import com.example.imagecardapp.ui.PostCard
import com.example.imagecardapp.viewmodel.PostsUiState
import com.example.imagecardapp.viewmodel.PostsViewModel

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
            TopAppBar(
                title = { Text("Посты с картинками") },
                actions = {
                    IconButton(onClick = { viewModel.loadPosts() }) {
                        Icon(Icons.Default.Refresh, contentDescription = "Обновить")
                    }
                }
            )
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
                    LazyColumn {
                        items(
                            items = (uiState as PostsUiState.Success).posts,
                            key = { post -> post.id }
                        ) { post ->
                            PostCard(post = post)
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

Запустите приложение. Убедитесь, что:
- При запуске отображается индикатор загрузки.
- Через несколько секунд появляется список карточек с изображениями и текстом.
- Изображения загружаются корректно (можно проверить разные ID – у Picsum есть изображения не для всех ID, но для большинства они есть).
- При отключении интернета отображается ошибка и кнопка "Повторить".
- При нажатии на кнопку обновления данные перезагружаются.

### Этап 11. Дополнительные улучшения (оставшееся время, 10 мин)

- Добавьте placeholder и error изображения в `AsyncImage`. Для этого можно использовать `painterResource` с локальными drawable.
- Добавьте обработку клика на карточку (например, открытие детального экрана с большим изображением).
- Измените размеры и форму изображений (например, круглые аватары).

Пример улучшенного `AsyncImage`:
```kotlin
AsyncImage(
    model = "https://picsum.photos/id/${post.id}/200/200",
    contentDescription = "Post image",
    modifier = Modifier
        .size(100.dp)
        .clip(RoundedCornerShape(8.dp)),
    contentScale = ContentScale.Crop,
    placeholder = painterResource(R.drawable.placeholder),
    error = painterResource(R.drawable.error)
)
```

Не забудьте добавить файлы `placeholder.png` и `error.png` в `res/drawable`.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Экран товаров интернет-магазина**  
   Создайте модель `Product` (id, name, price, imageUrl). Используйте MockAPI или сгенерируйте тестовые данные. Отобразите товары в виде карточек с ценой и изображением. Добавьте кнопку "Купить" (показывает Toast).

2. **Галерея фильмов**  
   Используйте API The Movie Database (TMDB) для получения списка популярных фильмов. Для этого потребуется зарегистрироваться и получить API ключ. Отобразите постеры, названия и рейтинг. (Сложный вариант, требует работы с API ключом.)

3. **Кэширование изображений**  
   Изучите, как Coil кэширует изображения. Добавьте индикатор прогресса загрузки (можно использовать `CircularProgressIndicator` как placeholder).

4. **Бесконечная прокрутка с пагинацией**  
   Реализуйте подгрузку новых постов при прокрутке вниз. JSONPlaceholder поддерживает параметры `_page` и `_limit`. Добавьте состояние загрузки следующей страницы.

---

## 5. Контрольные вопросы

1. Для чего нужны библиотеки загрузки изображений (Coil, Glide, Picasso)? Какие задачи они решают?
2. Как работает `AsyncImage` из Coil? Какие параметры можно настроить?
3. Как сформировать URL изображения на основе данных из API?
4. Каким образом Coil взаимодействует с корутинами?
5. Как обработать ситуацию, когда изображение не загрузилось (ошибка сети, неверный URL)?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги всех созданных файлов: `Post.kt`, `ApiService.kt`, `RetrofitClient.kt`, `PostsRepository.kt`, `PostsViewModel.kt`, `PostCard.kt`, `MainActivity.kt`.
- Скриншоты работающего приложения (список карточек, состояние загрузки, состояние ошибки).
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Изображения не загружаются** – проверьте доступ к интернету. Убедитесь, что URL корректен (можно открыть в браузере). Добавьте `Log.d` для проверки URL.
- **Ошибка парсинга JSON** – убедитесь, что поля в модели соответствуют JSON. Для отладки можно вывести ответ в лог.
- **Приложение падает с OutOfMemoryError** – используйте библиотеки загрузки изображений (они автоматически оптимизируют размер). Убедитесь, что изображения не слишком большие.
- **Placeholder не отображается** – проверьте, что файлы ресурсов добавлены и указаны правильные имена.
- **Coil не работает в Compose** – убедитесь, что добавлена зависимость `io.coil-kt:coil-compose` и выполнена синхронизация.

---

## 8. Дополнительные материалы

- [Документация Coil для Compose](https://coil-kt.github.io/coil/compose/)
- [Picsum Photos](https://picsum.photos/) – сервис случайных изображений
- [JSONPlaceholder](https://jsonplaceholder.typicode.com/) – тестовое API

**Успешной работы!**