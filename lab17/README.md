# Лабораторная работа №17
## Подключение Hilt. Внедрение репозитория и Retrofit в ViewModel

**Длительность:** 1 час 30 минут  
**Цель работы:** Изучить основы внедрения зависимостей (Dependency Injection) с помощью фреймворка Hilt, научиться организовывать модули для предоставления зависимостей, внедрять репозиторий и сетевой клиент в ViewModel, а также настроить проект для использования Hilt.

---

## 1. Теоретическая справка

### 1.1. Dependency Injection (DI)
**Внедрение зависимостей** — это паттерн проектирования, при котором объект получает свои зависимости извне, а не создаёт их сам. Это улучшает тестируемость, переиспользуемость и разделение ответственности.

### 1.2. Dagger Hilt
**Hilt** — это библиотека для DI, построенная на основе Dagger, но значительно упрощающая его использование в Android-приложениях. Hilt предоставляет стандартные компоненты, привязанные к жизненному циклу Android (Application, Activity, ViewModel и т.д.), и автоматически генерирует код для внедрения зависиностей.

Основные понятия Hilt:
- **@HiltAndroidApp** — аннотирует класс Application, инициализирует Hilt.
- **@AndroidEntryPoint** — аннотирует Activity, Fragment, Service и т.д., чтобы в них можно было внедрять зависимости.
- **@Module** — класс, в котором определяются, как создавать зависимости.
- **@Provides** — метод модуля, возвращающий экземпляр зависимости.
- **@Singleton** — указывает, что зависимость будет создаваться один раз на всё приложение.
- **@Inject** — аннотирует конструктор или поле для запроса зависимости.
- **@HiltViewModel** — аннотирует ViewModel, чтобы Hilt мог внедрять зависимости в её конструктор.

### 1.3. Преимущества использования Hilt
- Уменьшение шаблонного кода.
- Автоматическое управление жизненным циклом компонентов.
- Интеграция с Jetpack-библиотеками (ViewModel, WorkManager и т.д.).
- Улучшенная тестируемость.

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с проектом, который выполняет сетевые запросы (например, из лабораторной работы №15 или №16 — приложение, загружающее посты с картинками).
- Эмулятор или реальное устройство.

---

## 3. Порядок выполнения работы

### Этап 1. Подготовка проекта (5 мин)

Возьмите проект из лабораторной работы №15 или №16 (экран с карточками постов). Убедитесь, что проект успешно компилируется и работает: загружает данные из сети, отображает список, обрабатывает ошибки.

### Этап 2. Добавление зависимостей Hilt (5 мин)

Откройте файл `build.gradle.kts` (Project-level) и добавьте плагин Hilt в список `plugins` в секции `buildscript` или `plugins` (в зависимости от структуры). Обычно это делается в корневом `build.gradle.kts`:

```kotlin
plugins {
    // ...
    id("com.google.dagger.hilt.android") version "2.48" apply false
}
```

Затем в файле `app/build.gradle.kts` (Module-level) добавьте плагин и зависимости:

```kotlin
plugins {
    id("com.android.application")
    id("org.jetbrains.kotlin.android")
    id("kotlin-kapt")
    id("com.google.dagger.hilt.android")
}

android {
    // ... остальные настройки
}

dependencies {
    // ... существующие зависимости

    implementation("com.google.dagger:hilt-android:2.48")
    kapt("com.google.dagger:hilt-compiler:2.48")

    // Для ViewModel (если ещё не добавлено)
    implementation("androidx.hilt:hilt-navigation-compose:1.1.0")
    // или для XML-проекта:
    implementation("androidx.hilt:hilt-lifecycle-viewmodel:1.0.0-alpha03")
    kapt("androidx.hilt:hilt-compiler:1.1.0")
}
```

Выполните синхронизацию проекта.

### Этап 3. Создание Application класса и аннотация Hilt (5 мин)

Создайте класс, наследующий от `Application`, и аннотируйте его `@HiltAndroidApp`:

```kotlin
package com.example.imagecardapp

import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp
class App : Application()
```

Не забудьте указать этот класс в манифесте (`AndroidManifest.xml`) в атрибуте `android:name` тега `<application>`:

```xml
<application
    android:name=".App"
    ... >
    ...
</application>
```

### Этап 4. Создание модулей для зависимостей (15 мин)

Создайте пакет `di` (dependency injection). В нём создайте модули для Retrofit, ApiService и репозитория.

**Модуль для Network (Retrofit и ApiService):**

```kotlin
package com.example.imagecardapp.di

import com.example.imagecardapp.api.ApiService
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"

    @Provides
    @Singleton
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BASE_URL)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

**Модуль для репозитория:**

```kotlin
package com.example.imagecardapp.di

import com.example.imagecardapp.repository.PostsRepository
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {

    @Provides
    @Singleton
    fun providePostsRepository(apiService: ApiService): PostsRepository {
        return PostsRepository(apiService)
    }
}
```

### Этап 5. Модификация репозитория (5 мин)

Измените класс `PostsRepository`, чтобы он принимал `ApiService` через конструктор (внедрение зависимости), а не создавал его сам:

```kotlin
package com.example.imagecardapp.repository

import com.example.imagecardapp.api.ApiService
import com.example.imagecardapp.model.Post
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.withContext
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class PostsRepository @Inject constructor(
    private val apiService: ApiService
) {
    suspend fun getPosts(): List<Post> = withContext(Dispatchers.IO) {
        apiService.getPosts()
    }
}
```

Обратите внимание:
- `@Inject` на конструкторе указывает Hilt, как создавать экземпляр этого класса.
- `@Singleton` гарантирует, что репозиторий будет создан один раз на всё приложение.

### Этап 6. Модификация ViewModel (10 мин)

Теперь измените `PostsViewModel`, чтобы он получал репозиторий через конструктор. Для этого используйте аннотацию `@HiltViewModel` и `@Inject` в конструкторе:

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
import kotlinx.coroutines.launch
import javax.inject.Inject

sealed class PostsUiState {
    object Loading : PostsUiState()
    data class Success(val posts: List<Post>) : PostsUiState()
    data class Error(val message: String) : PostsUiState()
}

@HiltViewModel
class PostsViewModel @Inject constructor(
    private val repository: PostsRepository
) : ViewModel() {

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

### Этап 7. Обновление Activity (5 мин)

В Activity, которая использует эту ViewModel (например, `MainActivity`), добавьте аннотацию `@AndroidEntryPoint`. Уберите ручное создание репозитория и ViewModel — теперь Hilt будет внедрять ViewModel автоматически.

```kotlin
package com.example.imagecardapp

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.runtime.getValue
import androidx.lifecycle.compose.collectAsStateWithLifecycle
import androidx.lifecycle.viewmodel.compose.viewModel
import dagger.hilt.android.AndroidEntryPoint

@AndroidEntryPoint
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                // Теперь ViewModel создаётся Hilt
                val viewModel: PostsViewModel = viewModel()
                val uiState by viewModel.uiState.collectAsStateWithLifecycle()
                PostsScreen(uiState, viewModel::loadPosts)
            }
        }
    }
}
```

Обратите внимание: мы передаём в `PostsScreen` только состояние и колбэк для обновления, чтобы не зависеть от ViewModel напрямую (опционально). Можно оставить и прежнюю структуру, где `PostsScreen` получает ViewModel через `viewModel()`.

### Этап 8. Исправление остальных зависимостей (если есть)

Если в проекте использовались какие-то ещё зависимости (например, контекст), их тоже можно внедрить через Hilt. Для данной работы это не требуется.

### Этап 9. Запуск и тестирование (5 мин)

Запустите приложение. Убедитесь, что оно работает так же, как и раньше: загружает список постов, отображает карточки, обрабатывает ошибки.

### Этап 10. Проверка сборки и логов (5 мин)

При успешной сборке в логах не должно быть ошибок, связанных с Hilt. Если возникают проблемы, проверьте:
- Аннотации `@HiltAndroidApp`, `@AndroidEntryPoint`, `@HiltViewModel`.
- Наличие всех зависимостей и плагинов.
- Правильность импортов.

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Добавление модуля для Context**  
   Создайте модуль, который предоставляет `Application` или `Context`. Внедрите его в репозиторий или ViewModel (например, для доступа к ресурсам).

2. **Внедрение OkHttpClient с интерцептором**  
   Модифицируйте `NetworkModule`, чтобы добавить `OkHttpClient` с логирующим интерцептором. Внедрите этот клиент в Retrofit.

3. **Многомодульность**  
   Если проект разбит на модули, настройте Hilt для работы с несколькими модулями.

4. **Тестирование с Hilt**  
   Напишите простой тест для ViewModel с использованием `HiltViewModel` и `TestInstallIn`.

---

## 5. Контрольные вопросы

1. Что такое Dependency Injection? Какие проблемы он решает?
2. Какие преимущества даёт использование Hilt по сравнению с ручным созданием зависимостей?
3. Для чего нужны аннотации `@Module`, `@Provides`, `@Singleton`?
4. Как Hilt интегрируется с ViewModel?
5. Что произойдёт, если забыть аннотировать Activity `@AndroidEntryPoint`?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги всех созданных/изменённых файлов: `App.kt`, `NetworkModule.kt`, `RepositoryModule.kt`, `PostsRepository.kt`, `PostsViewModel.kt`, `MainActivity.kt`.
- Скриншоты работающего приложения (можно те же, что и в предыдущих работах, чтобы показать сохранение функциональности).
- Ответы на контрольные вопросы.
- Вывод по работе.

---

## 7. Возможные ошибки и их решение

- **Ошибка: "Dagger does not know how to provide ApiService"** – убедитесь, что модуль `NetworkModule` правильно предоставляет `ApiService` и что модуль установлен в правильный компонент (`SingletonComponent`).
- **Ошибка: "Cannot create instance of ViewModel"** – проверьте, что ViewModel аннотирована `@HiltViewModel` и имеет конструктор с `@Inject`.
- **Ошибка: "No injector factory bound for Class"** – убедитесь, что Activity аннотирована `@AndroidEntryPoint` и что Application аннотирован `@HiltAndroidApp`.
- **Ошибка компиляции kapt** – проверьте версии Hilt и плагинов, выполните `Build -> Clean Project` и `Rebuild`.
- **Приложение падает при запуске с `IllegalStateException: Hilt Activity must be attached to an @HiltAndroidApp Application`** – значит, не указан `android:name=".App"` в манифесте.

---

## 8. Дополнительные материалы

- [Официальная документация Hilt](https://developer.android.com/training/dependency-injection/hilt-android)
- [Codelab: Dependency Injection with Hilt](https://developer.android.com/codelabs/android-hilt)

**Успешной работы!**