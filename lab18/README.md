# Лабораторная работа №18
## Реализация ленты с подгрузкой при скролле (бесконечная прокрутка)

**Длительность:** 1 час 30 минут  
**Цель работы:** Научиться реализовывать пагинацию (бесконечную прокрутку) при работе со списками данных из сети, используя API с поддержкой постраничной загрузки, добавлять индикатор загрузки в конец списка и обрабатывать возможные ошибки при подгрузке.

---

## 1. Теоретическая справка

### 1.1. Пагинация (постраничная загрузка)
При работе с большими объёмами данных загружать все элементы сразу неэффективно. Используется пагинация – загрузка данных порциями (страницами). Существует два основных подхода:
- **Offset-based** – используются параметры `_page` (номер страницы) и `_limit` (размер страницы). API JSONPlaceholder поддерживает именно этот метод.
- **Cursor-based** – используется маркер (например, `id` последнего элемента) для получения следующей порции данных.

В данной работе мы будем использовать offset-based пагинацию с параметрами `_page` и `_limit`.

### 1.2. Бесконечная прокрутка (Infinite scroll)
При достижении конца списка автоматически загружается следующая страница. Для этого в `LazyColumn` используется триггер – когда последний элемент становится видимым, запускается загрузка следующей страницы.

### 1.3. Управление состоянием в ViewModel
Для реализации пагинации необходимо хранить:
- Список загруженных элементов (`List<Post>`)
- Текущую страницу (`currentPage`)
- Флаг, загружаются ли сейчас данные (`isLoading`)
- Флаг, есть ли ещё данные для загрузки (`hasMorePages`)
- Состояние ошибки при загрузке следующей страницы

### 1.4. Обработка граничных случаев
- Не загружать следующую страницу, если уже идёт загрузка.
- Не загружать, если все данные уже получены.
- При ошибке показывать повторную попытку загрузить страницу.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом, созданным в лабораторной работе №17 (с Hilt, Retrofit, ViewModel и экраном на Compose).
- Эмулятор или реальное устройство с доступом в интернет.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Откройте проект из лабораторной работы №17 (приложение, загружающее список постов с картинками). Убедитесь, что проект успешно компилируется и работает: загружает первую страницу (без пагинации) и отображает карточки.

### Этап 2. Модификация API интерфейса для поддержки пагинации (5 мин)

В интерфейсе `ApiService` добавьте параметры `_page` и `_limit` к методу `getPosts()`:

```kotlin
package com.example.imagecardapp.api

import com.example.imagecardapp.model.Post
import retrofit2.http.GET
import retrofit2.http.Query

interface ApiService {
    @GET("posts")
    suspend fun getPosts(
        @Query("_page") page: Int,
        @Query("_limit") limit: Int
    ): List<Post>
}
```

### Этап 3. Обновление репозитория (5 мин)

Измените метод `getPosts()` в `PostsRepository`, чтобы он принимал параметры страницы и лимита:

```kotlin
suspend fun getPosts(page: Int, limit: Int): List<Post> = withContext(Dispatchers.IO) {
    apiService.getPosts(page, limit)
}
```

### Этап 4. Модификация ViewModel для управления пагинацией (20 мин)

Создайте новый sealed class для состояний подгрузки и обновите `PostsViewModel`. Важно: нужно различать состояние первой загрузки и последующие подгрузки.

```kotlin
package com.example.imagecardapp.viewmodel

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.imagecardapp.model.Post
import com.example.imagecardapp.repository.PostsRepository
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.update
import kotlinx.coroutines.launch
import javax.inject.Inject

sealed class PostsUiState {
    object Loading : PostsUiState()               // первая загрузка
    data class Success(
        val posts: List<Post>,
        val isLoadingMore: Boolean = false,
        val hasMore: Boolean = true,
        val error: String? = null
    ) : PostsUiState()
    data class Error(val message: String) : PostsUiState() // ошибка первой загрузки
}

@HiltViewModel
class PostsViewModel @Inject constructor(
    private val repository: PostsRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<PostsUiState>(PostsUiState.Loading)
    val uiState: StateFlow<PostsUiState> = _uiState.asStateFlow()

    private var currentPage = 1
    private var isCurrentlyLoading = false
    private val pageSize = 20

    init {
        loadFirstPage()
    }

    fun loadFirstPage() {
        viewModelScope.launch {
            _uiState.value = PostsUiState.Loading
            currentPage = 1
            isCurrentlyLoading = true
            try {
                val posts = repository.getPosts(currentPage, pageSize)
                _uiState.value = PostsUiState.Success(
                    posts = posts,
                    hasMore = posts.size == pageSize
                )
                currentPage++
            } catch (e: Exception) {
                _uiState.value = PostsUiState.Error(e.message ?: "Unknown error")
            } finally {
                isCurrentlyLoading = false
            }
        }
    }

    fun loadNextPage() {
        // Проверяем, можно ли загружать следующую страницу
        val currentState = _uiState.value
        if (currentState !is PostsUiState.Success) return
        if (isCurrentlyLoading || !currentState.hasMore) return

        viewModelScope.launch {
            isCurrentlyLoading = true
            _uiState.update { state ->
                if (state is PostsUiState.Success) {
                    state.copy(isLoadingMore = true, error = null)
                } else state
            }
            try {
                val newPosts = repository.getPosts(currentPage, pageSize)
                _uiState.update { state ->
                    if (state is PostsUiState.Success) {
                        val updatedPosts = state.posts + newPosts
                        state.copy(
                            posts = updatedPosts,
                            isLoadingMore = false,
                            hasMore = newPosts.size == pageSize,
                            error = null
                        )
                    } else state
                }
                currentPage++
            } catch (e: Exception) {
                _uiState.update { state ->
                    if (state is PostsUiState.Success) {
                        state.copy(
                            isLoadingMore = false,
                            error = e.message ?: "Ошибка загрузки"
                        )
                    } else state
                }
            } finally {
                isCurrentlyLoading = false
            }
        }
    }

    fun retryNextPage() {
        loadNextPage()
    }
}
```

### Этап 5. Обновление UI (экрана) для поддержки пагинации (20 мин)

Теперь нужно модифицировать экран, чтобы:
- При успешном состоянии отображать список постов.
- В конце списка, если есть ещё данные, показывать индикатор загрузки (или кнопку повторной попытки при ошибке).
- Следить за достижением конца списка и вызывать `loadNextPage()`.

Пример реализации с использованием `LazyColumn` и ключевого элемента `item`:

```kotlin
@Composable
fun PostsScreen(
    viewModel: PostsViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    when (uiState) {
        is PostsUiState.Loading -> {
            Box(modifier = Modifier.fillMaxSize()) {
                CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
            }
        }

        is PostsUiState.Success -> {
            val state = uiState as PostsUiState.Success
            LazyColumn {
                items(
                    items = state.posts,
                    key = { it.id }
                ) { post ->
                    PostCard(post = post)
                }

                // Элемент для отображения индикатора загрузки или кнопки повтора
                if (state.hasMore || state.isLoadingMore || state.error != null) {
                    item {
                        Box(
                            modifier = Modifier
                                .fillMaxWidth()
                                .padding(16.dp),
                            contentAlignment = Alignment.Center
                        ) {
                            when {
                                state.error != null -> {
                                    Column(horizontalAlignment = Alignment.CenterHorizontally) {
                                        Text(
                                            text = "Ошибка: ${state.error}",
                                            color = MaterialTheme.colorScheme.error
                                        )
                                        Button(
                                            onClick = { viewModel.retryNextPage() },
                                            modifier = Modifier.padding(top = 8.dp)
                                        ) {
                                            Text("Повторить")
                                        }
                                    }
                                }
                                state.isLoadingMore -> {
                                    CircularProgressIndicator()
                                }
                                state.hasMore -> {
                                    // Можно оставить пустой Box, но лучше показывать что-то невидимое как триггер
                                    // Используем LaunchedEffect для вызова loadNextPage, когда этот элемент появляется
                                    LaunchedEffect(Unit) {
                                        viewModel.loadNextPage()
                                    }
                                    Box(Modifier.fillMaxWidth()) // невидимый триггер
                                }
                            }
                        }
                    }
                }
            }
        }

        is PostsUiState.Error -> {
            Column(
                modifier = Modifier.fillMaxSize(),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text("Ошибка: ${(uiState as PostsUiState.Error).message}")
                Button(
                    onClick = { viewModel.loadFirstPage() },
                    modifier = Modifier.padding(top = 8.dp)
                ) {
                    Text("Повторить")
                }
            }
        }
    }
}
```

**Важно:** В коде выше используется `LaunchedEffect` для автоматического вызова `loadNextPage()` при появлении элемента-триггера. Это корректно, но нужно быть внимательным, чтобы не вызвать множество раз. Можно также использовать модификатор `Modifier.onGloballyPositioned` и проверять видимость, но `LaunchedEffect` проще.

### Этап 6. Запуск и тестирование (10 мин)

Запустите приложение. Убедитесь, что:
- Первая страница загружается (первые 20 постов).
- При прокрутке вниз автоматически подгружается следующая страница.
- Индикатор загрузки появляется в конце списка.
- Когда данные закончатся (на странице меньше `pageSize` элементов), индикатор не показывается и новые запросы не отправляются.
- При ошибке (например, отключить интернет во время подгрузки) отображается кнопка "Повторить".

### Этап 7. Доработка обработки ошибок (5 мин)

Убедитесь, что при ошибке подгрузки пользователь может повторить попытку, и что после повторной попытки загрузка продолжается с той же страницы.

### Этап 8. Оптимизация (оставшееся время)

- Добавьте debounce или throttle для предотвращения множественных вызовов при быстрой прокрутке (хотя в `LaunchedEffect` это не нужно, так как он срабатывает только один раз при появлении).
- Можно добавить возможность обновления списка (pull-to-refresh).

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Pull-to-refresh**  
   Добавьте `PullRefreshIndicator` из Compose Material для обновления списка (сброс всех страниц и загрузка первой).

2. **Разные размеры страниц**  
   Сделайте размер страницы настраиваемым через параметр (например, 10, 20, 50) с помощью переключателя в интерфейсе.

3. **Кэширование страниц**  
   Реализуйте простое кэширование загруженных страниц в памяти (например, в `ViewModel` хранить `Map<Int, List<Post>>`), чтобы при повторной загрузке той же страницы не делать лишних запросов.

4. **Сохранение состояния при повороте**  
   Убедитесь, что после поворота экрана текущий список и номер страницы сохраняются (с ViewModel это должно работать автоматически). Добавьте сохранение прокрутки с помощью `rememberLazyListState`.

---

## 5. Контрольные вопросы

1. Какие параметры обычно используются для offset-based пагинации?
2. Как определить, что все данные загружены?
3. Зачем нужен флаг `isCurrentlyLoading` в ViewModel?
4. Как в Compose отследить момент, когда пользователь достиг конца списка?
5. Какие проблемы могут возникнуть при реализации бесконечной прокрутки и как их избежать?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги изменённых файлов: `ApiService.kt`, `PostsRepository.kt`, `PostsViewModel.kt`, `MainActivity.kt` (или экрана).
- Скриншоты приложения с демонстрацией подгрузки (можно сделать несколько скриншотов с разными страницами).
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Запросы дублируются** – проверьте флаг `isCurrentlyLoading`, возможно, он не сбрасывается или `LaunchedEffect` срабатывает несколько раз. Используйте уникальный ключ в `LaunchedEffect`.
- **Не загружается вторая страница** – убедитесь, что параметры `_page` и `_limit` корректно передаются. Проверьте URL в логах.
- **После ошибки кнопка "Повторить" не работает** – проверьте, что `retryNextPage` вызывает `loadNextPage` без сброса состояния.
- **Индикатор загрузки виден постоянно** – проверьте логику `hasMore`: если страница пуста или размер меньше лимита, `hasMore` должно стать `false`.

---

**Успешной работы!**