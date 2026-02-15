# Лабораторная работа №19
## Интеграция выбранного API

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться самостоятельно интегрировать реальный внешний API в Android-приложение, анализировать документацию API, создавать соответствующие модели данных и обрабатывать ответы сервера.

---

## 1. Теоретическая справка

### 1.1. Выбор API для интеграции
При разработке приложений часто возникает необходимость использовать данные из внешних источников. Существует множество публичных API, предоставляющих бесплатный доступ к данным . При выборе API следует учитывать:
- **Документация** – насколько подробно описаны эндпоинты, форматы запросов и ответов
- **Аутентификация** – требуется ли API ключ, токен или OAuth
- **Ограничения** – лимиты на количество запросов (rate limits)
- **Формат данных** – обычно JSON, но бывает XML или другие форматы

### 1.2. Анализ JSON-ответа
Перед созданием моделей данных необходимо изучить структуру JSON-ответа от API. В документации обычно есть раздел "Example Response", который показывает, как выглядят возвращаемые данные .

Пример анализа JSON:
```json
{
  "id": 1,
  "title": "Example Title",
  "description": "Example description",
  "metadata": {
    "created": "2024-01-01",
    "author": "John Doe"
  }
}
```

На основе такой структуры создаются data class'ы:
```kotlin
data class Item(
    val id: Int,
    val title: String,
    val description: String,
    val metadata: Metadata
)

data class Metadata(
    val created: String,
    val author: String
)
```

### 1.3. Работа с API ключами
Многие API требуют аутентификации через API ключ. Ключ можно передавать разными способами :
- **Как query параметр**: `https://api.example.com/data?api_key=YOUR_KEY`
- **В заголовке запроса**: `Authorization: Bearer YOUR_KEY` или `X-API-Key: YOUR_KEY`

Важно: API ключи не должны храниться в коде в открытом виде. Для учебных проектов можно использовать `BuildConfig` или локальные свойства.

### 1.4. Обработка сложных ответов
Некоторые API возвращают вложенные структуры, массивы в объектах, или требуют специальной обработки полей (например, имена полей в JSON могут отличаться от принятых в Kotlin). Для этого используется аннотация `@SerializedName` .

```kotlin
data class Movie(
    @SerializedName("imdb_id")
    val imdbId: String,
    
    @SerializedName("Title")
    val title: String
)
```

---

## 2. Варианты API для интеграции

Студенты могут выбрать один из предложенных API или предложить свой (по согласованию с преподавателем).

### Вариант 1: National Park Service API
Бесплатный API для получения информации о национальных парках США .
- **Документация**: https://www.nps.gov/subjects/developer/api-documentation.htm
- **Эндпоинт**: `https://developer.nps.gov/api/v1/parks`
- **Требуется**: API ключ (бесплатный после регистрации)
- **Данные**: парки, описание, изображения, штаты

### Вариант 2: GitHub API
Публичный API для получения информации о репозиториях .
- **Документация**: https://docs.github.com/en/rest
- **Эндпоинт**: `https://api.github.com/search/repositories?q=language:kotlin&sort=stars`
- **Требуется**: без ключа (ограничение 60 запросов/час)
- **Данные**: репозитории, звёзды, описание, владелец

### Вариант 3: IMDb API (через RapidAPI)
API для получения информации о фильмах .
- **Платформа**: https://rapidapi.com/ (требуется регистрация)
- **Эндпоинты**: автодополнение поиска, информация о фильме, актёрский состав
- **Требуется**: RapidAPI ключ (бесплатный тариф)
- **Данные**: фильмы, актёры, рейтинги, постеры

### Вариант 4: JSONPlaceholder (упрощённый)
Тестовое API, не требует ключа .
- **Документация**: https://jsonplaceholder.typicode.com/
- **Эндпоинт**: `https://jsonplaceholder.typicode.com/posts`
- **Требуется**: без ключа
- **Данные**: посты, комментарии, пользователи

---

## 3. Порядок выполнения работы

### Этап 1. Выбор API и изучение документации (10 мин)

1. Выберите один из предложенных API (или предложите свой).
2. Изучите документацию выбранного API:
   - Найдите базовый URL (base URL)
   - Найдите интересующий вас эндпоинт (например, список парков, поиск репозиториев)
   - Изучите пример ответа (JSON)
   - Выясните, требуется ли аутентификация
3. Запишите в отчёт:
   - Название API
   - Базовый URL
   - Выбранный эндпоинт
   - Пример структуры ответа

### Этап 2. Создание проекта и добавление зависимостей (5 мин)

Создайте новый проект с шаблоном **Empty Activity** и Compose, либо используйте существующий проект из предыдущих работ. Добавьте необходимые зависимости:

```kotlin
dependencies {
    // Retrofit
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    
    // Корутины
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    
    // ViewModel для Compose
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
    
    // Coil для загрузки изображений (если API возвращает URL картинок)
    implementation("io.coil-kt:coil-compose:2.5.0")
}
```

Не забудьте добавить разрешение на интернет в `AndroidManifest.xml` :
```xml
<uses-permission android:name="android.permission.INTERNET" />
```

### Этап 3. Создание моделей данных на основе анализа JSON (15 мин)

На основе изученного примера ответа создайте data class'ы в пакете `model`. Обратите внимание на вложенные структуры и возможные расхождения в именовании полей.

**Пример для National Park Service API :**
```kotlin
package com.example.nationalparkapp.model

import com.google.gson.annotations.SerializedName

data class ParksResponse(
    val total: Int,
    val data: List<Park>,
    val limit: Int,
    val start: Int
)

data class Park(
    val id: String,
    val name: String,
    val description: String,
    
    @SerializedName("fullName")
    val fullName: String,
    
    val states: String,
    
    @SerializedName("images")
    val images: List<ParkImage>
)

data class ParkImage(
    val url: String,
    val altText: String,
    val title: String
)
```

**Пример для GitHub API :**
```kotlin
package com.example.githubapp.model

data class SearchResponse(
    val total_count: Int,
    val items: List<Repository>
)

data class Repository(
    val id: Int,
    val name: String,
    val description: String?,
    
    @SerializedName("stargazers_count")
    val stars: Int,
    
    @SerializedName("html_url")
    val htmlUrl: String,
    
    val owner: Owner
)

data class Owner(
    val login: String,
    
    @SerializedName("avatar_url")
    val avatarUrl: String
)
```

### Этап 4. Создание API интерфейса (10 мин)

Создайте интерфейс, описывающий эндпоинты API. Используйте аннотации `@GET`, `@POST`, `@Query`, `@Path` и другие в зависимости от документации .

**Пример для National Park Service API :**
```kotlin
package com.example.nationalparkapp.api

import com.example.nationalparkapp.model.ParksResponse
import retrofit2.http.GET
import retrofit2.http.Query

interface NationalParkServiceApi {
    @GET("parks")
    suspend fun getParks(
        @Query("api_key") apiKey: String,
        @Query("limit") limit: Int = 20,
        @Query("start") start: Int = 0
    ): ParksResponse
}
```

**Пример для GitHub API :**
```kotlin
package com.example.githubapp.api

import com.example.githubapp.model.SearchResponse
import retrofit2.http.GET
import retrofit2.http.Query

interface GitHubApi {
    @GET("search/repositories")
    suspend fun searchRepositories(
        @Query("q") query: String,
        @Query("sort") sort: String = "stars",
        @Query("order") order: String = "desc"
    ): SearchResponse
}
```

### Этап 5. Создание Retrofit клиента (10 мин)

Создайте объект для предоставления экземпляра Retrofit и API сервиса .

**Базовый вариант (без ключа):**
```kotlin
package com.example.githubapp.api

import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {
    private const val BASE_URL = "https://api.github.com/"
    
    val instance: ApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(ApiService::class.java)
    }
}
```

**Вариант с API ключом (через интерцептор) :**
```kotlin
package com.example.nationalparkapp.api

import okhttp3.OkHttpClient
import okhttp3.logging.HttpLoggingInterceptor
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {
    private const val BASE_URL = "https://developer.nps.gov/api/v1/"
    private const val API_KEY = "YOUR_API_KEY" // В реальном проекте используйте BuildConfig
    
    private val loggingInterceptor = HttpLoggingInterceptor().apply {
        level = HttpLoggingInterceptor.Level.BODY
    }
    
    private val client = OkHttpClient.Builder()
        .addInterceptor { chain ->
            val original = chain.request()
            val originalUrl = original.url
            val url = originalUrl.newBuilder()
                .addQueryParameter("api_key", API_KEY)
                .build()
            val request = original.newBuilder()
                .url(url)
                .build()
            chain.proceed(request)
        }
        .addInterceptor(loggingInterceptor)
        .build()
    
    val instance: NationalParkServiceApi by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
            .create(NationalParkServiceApi::class.java)
    }
}
```

### Этап 6. Создание репозитория (10 мин)

Создайте репозиторий, который будет использовать API сервис .

```kotlin
package com.example.nationalparkapp.repository

import com.example.nationalparkapp.api.RetrofitClient
import com.example.nationalparkapp.model.Park
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.withContext

class ParksRepository {
    private val api = RetrofitClient.instance
    
    suspend fun getParks(): List<Park> = withContext(Dispatchers.IO) {
        try {
            val response = api.getParks(limit = 20)
            response.data
        } catch (e: Exception) {
            emptyList()
        }
    }
    
    fun getParksFlow(): Flow<List<Park>> = flow {
        val parks = getParks()
        emit(parks)
    }
}
```

### Этап 7. Создание ViewModel (10 мин)

Создайте ViewModel для управления состоянием .

```kotlin
package com.example.nationalparkapp.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.nationalparkapp.model.Park
import com.example.nationalparkapp.repository.ParksRepository
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.launch

sealed class ParksUiState {
    object Loading : ParksUiState()
    data class Success(val parks: List<Park>) : ParksUiState()
    data class Error(val message: String) : ParksUiState()
}

class ParksViewModel : ViewModel() {
    private val repository = ParksRepository()
    
    private val _uiState = MutableStateFlow<ParksUiState>(ParksUiState.Loading)
    val uiState: StateFlow<ParksUiState> = _uiState.asStateFlow()
    
    init {
        loadParks()
    }
    
    fun loadParks() {
        viewModelScope.launch {
            _uiState.value = ParksUiState.Loading
            try {
                val parks = repository.getParks()
                _uiState.value = ParksUiState.Success(parks)
            } catch (e: Exception) {
                _uiState.value = ParksUiState.Error(e.message ?: "Unknown error")
            }
        }
    }
    
    fun refresh() {
        loadParks()
    }
}
```

### Этап 8. Создание UI с использованием Compose (15 мин)

Создайте экран для отображения полученных данных .

```kotlin
package com.example.nationalparkapp.ui

import androidx.compose.foundation.layout.Box
import androidx.compose.foundation.layout.Column
import androidx.compose.foundation.layout.fillMaxSize
import androidx.compose.foundation.layout.fillMaxWidth
import androidx.compose.foundation.layout.padding
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.Button
import androidx.compose.material3.Card
import androidx.compose.material3.CircularProgressIndicator
import androidx.compose.material3.MaterialTheme
import androidx.compose.material3.Text
import androidx.compose.runtime.Composable
import androidx.compose.runtime.getValue
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import coil.compose.AsyncImage
import com.example.nationalparkapp.model.Park
import com.example.nationalparkapp.viewmodel.ParksUiState
import com.example.nationalparkapp.viewmodel.ParksViewModel

@Composable
fun ParksScreen(viewModel: ParksViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    Box(modifier = Modifier.fillMaxSize()) {
        when (uiState) {
            is ParksUiState.Loading -> {
                CircularProgressIndicator(
                    modifier = Modifier.align(Alignment.Center)
                )
            }
            
            is ParksUiState.Success -> {
                val parks = (uiState as ParksUiState.Success).parks
                LazyColumn {
                    items(parks) { park ->
                        ParkCard(park = park)
                    }
                }
            }
            
            is ParksUiState.Error -> {
                Column(
                    modifier = Modifier.align(Alignment.Center),
                    horizontalAlignment = Alignment.CenterHorizontally
                ) {
                    Text(
                        text = "Ошибка: ${(uiState as ParksUiState.Error).message}",
                        color = MaterialTheme.colorScheme.error
                    )
                    Button(
                        onClick = { viewModel.refresh() },
                        modifier = Modifier.padding(top = 8.dp)
                    ) {
                        Text("Повторить")
                    }
                }
            }
        }
    }
}

@Composable
fun ParkCard(park: Park) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 8.dp, vertical = 4.dp)
    ) {
        Column(
            modifier = Modifier.padding(16.dp)
        ) {
            if (park.images.isNotEmpty()) {
                AsyncImage(
                    model = park.images.first().url,
                    contentDescription = park.images.first().altText,
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(bottom = 8.dp)
                )
            }
            
            Text(
                text = park.name,
                style = MaterialTheme.typography.titleLarge
            )
            
            Text(
                text = park.states,
                style = MaterialTheme.typography.labelMedium
            )
            
            Text(
                text = park.description,
                style = MaterialTheme.typography.bodyMedium,
                maxLines = 3,
                modifier = Modifier.padding(top = 4.dp)
            )
        }
    }
}
```

### Этап 9. Запуск и тестирование (5 мин)

Запустите приложение и проверьте:
- Данные успешно загружаются и отображаются
- При отсутствии интернета показывается ошибка и кнопка повтора
- Изображения (если есть) загружаются корректно

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Детальный экран**  
   Добавьте второй экран с подробной информацией об элементе (парке, репозитории, фильме). При клике на карточку открывается экран деталей.

2. **Поиск**  
   Реализуйте поиск по данным. Для GitHub API – поиск репозиториев по названию, для National Park API – фильтрация по штату.

3. **Пагинация**  
   Добавьте бесконечную прокрутку с подгрузкой следующих страниц (используйте параметры `start`/`limit` или `page`/`per_page`).

4. **Кэширование с Room**  
   Сохраняйте загруженные данные в локальную базу данных. При запуске сначала показывайте кэш, затем обновляйте с сервера.

5. **Избранное**  
   Добавьте возможность отмечать элементы как избранные и сохранять это состояние .

---

## 5. Контрольные вопросы

1. Какие способы передачи API ключа в запросе существуют? Какой из них наиболее безопасный? 
2. Для чего нужна аннотация `@SerializedName`? В каких случаях она необходима?
3. Как проанализировать структуру JSON-ответа и создать соответствующие модели данных?
4. Какие сложности могут возникнуть при интеграции реального API и как их можно решить?
5. Что такое rate limiting и как его учитывать при разработке?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Название выбранного API, его базовый URL, выбранный эндпоинт, пример JSON-ответа.
- Листинги созданных файлов (модели данных, API интерфейс, Retrofit клиент, репозиторий, ViewModel, UI).
- Скриншоты работающего приложения (список данных, состояние загрузки, состояние ошибки).
- Ответы на контрольные вопросы.
- Вывод по работе (что нового узнали, с какими трудностями столкнулись при интеграции API).

---

## 7. Возможные ошибки и их решение

- **Ошибка аутентификации (401/403)** – проверьте правильность API ключа и способ его передачи (query parameter или header) .
- **Ошибка парсинга JSON** – распечатайте сырой ответ в лог и сверьте структуру с вашими моделями. Используйте `@SerializedName` для несоответствующих полей.
- **Пустой список при успешном ответе** – проверьте путь к данным в JSON (возможно, данные находятся в поле `data` или `items`) .
- **Превышение лимита запросов** – для GitHub API без ключа лимит 60 запросов в час. Добавьте кэширование или используйте ключ.
- **Изображения не загружаются** – проверьте URL изображений (относительные или абсолютные). Для относительных URL может потребоваться формировать полный URL.

---

## 8. Ресурсы для выбора API

- [Public APIs](https://github.com/public-apis/public-apis) – огромный список публичных API
- [RapidAPI Hub](https://rapidapi.com/hub) – платформа с множеством API
- [JSONPlaceholder](https://jsonplaceholder.typicode.com/) – тестовое API для практики
- [The Movie Database (TMDB)](https://developers.themoviedb.org/3) – популярное API о фильмах
- [PokéAPI](https://pokeapi.co/) – API о покемонах (без ключа)

**Успешной работы!**