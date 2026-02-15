# Лабораторная работа №9
## Сохранение настроек темы. Тёмная/светлая тема в Compose

**Длительность:** 1 час 30 минут  
**Цель работы:** Изучить механизмы смены и сохранения темы приложения в Jetpack Compose, научиться использовать DataStore/SharedPreferences для хранения пользовательских настроек, реализовать переключение между тёмной и светлой темами.

---

## 1. Теоретическая справка

### 1.1. Material Theme в Compose
Jetpack Compose использует `MaterialTheme` для обеспечения единого стиля приложения. `MaterialTheme` содержит три основные компоненты:
- **ColorScheme** – цветовая палитра (primary, secondary, background, surface и т.д.)
- **Typography** – стили текста
- **Shapes** – формы компонентов

### 1.2. Тёмная и светлая темы
В Compose поддержка тёмной темы реализуется через предоставление двух наборов цветов: для светлой и для тёмной темы. Определить, какая тема активна в данный момент, можно с помощью `isSystemInDarkTheme()` – эта функция возвращает `true`, если на устройстве включена тёмная тема .

### 1.3. Кастомная тема приложения
Рекомендуется создать собственный composable-обёртку над `MaterialTheme`, которая будет принимать параметр `darkTheme` и выбирать соответствующую цветовую схему .

```kotlin
@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    content: @Composable () -> Unit
) {
    val colorScheme = if (darkTheme) DarkColors else LightColors
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography(),
        content = content
    )
}
```

### 1.4. Сохранение настроек
Для сохранения выбора пользователя можно использовать:
- **SharedPreferences** – простой способ хранения пар ключ-значение
- **DataStore** – современная замена SharedPreferences на основе Kotlin Coroutines и Flow

В этой лабораторной работе мы используем DataStore, так как он лучше интегрируется с корутинами и Compose.

### 1.5. Динамические цвета (Android 12+)
На Android 12 и выше можно использовать динамические цвета, основанные на обоях устройства, через `dynamicDarkColorScheme()` и `dynamicLightColorScheme()` .

---

## 2. Оборудование и программное обеспечение

- Персональный компьютер с ОС Windows / macOS / Linux.
- Android Studio с установленным SDK.
- Эмулятор (API 21+) или реальное устройство.

---

## 3. Порядок выполнения работы

### Этап 1. Создание нового проекта (5 мин)

Создайте новый проект с шаблоном **Empty Activity**:
- **Name:** `ThemeSwitcherApp`
- **Package name:** `com.example.themeswitcher`
- **Language:** Kotlin
- **Minimum SDK:** API 24
- **UI toolkit:** Compose

### Этап 2. Добавление зависимостей (5 мин)

Откройте файл `app/build.gradle.kts` и добавьте зависимости для DataStore:

```kotlin
dependencies {
    // ... существующие зависимости
    
    implementation("androidx.datastore:datastore-preferences:1.0.0")
    implementation("androidx.lifecycle:lifecycle-runtime-compose:2.7.0")
}
```

Выполните синхронизацию проекта.

### Этап 3. Создание цветовых схем (10 мин)

Откройте файл `ui/theme/Color.kt`. Определите две цветовые схемы: для светлой и тёмной темы. Вы можете воспользоваться [Material Theme Builder](https://m3.material.io/theme-builder) для генерации своей палитры .

```kotlin
package com.example.themeswitcher.ui.theme

import androidx.compose.ui.graphics.Color
import androidx.compose.material3.*

// Светлая тема
val LightColors = lightColorScheme(
    primary = Color(0xFF006C4C),
    onPrimary = Color(0xFFFFFFFF),
    primaryContainer = Color(0xFF89F8C7),
    onPrimaryContainer = Color(0xFF002114),
    secondary = Color(0xFF4D635A),
    onSecondary = Color(0xFFFFFFFF),
    secondaryContainer = Color(0xFFCFE9DD),
    onSecondaryContainer = Color(0xFF0A1F19),
    tertiary = Color(0xFF3A637A),
    onTertiary = Color(0xFFFFFFFF),
    tertiaryContainer = Color(0xFFC1E8FF),
    onTertiaryContainer = Color(0xFF001E2C),
    background = Color(0xFFF4FBF5),
    onBackground = Color(0xFF161D1A),
    surface = Color(0xFFF4FBF5),
    onSurface = Color(0xFF161D1A)
)

// Тёмная тема
val DarkColors = darkColorScheme(
    primary = Color(0xFF6CDBB0),
    onPrimary = Color(0xFF003825),
    primaryContainer = Color(0xFF005239),
    onPrimaryContainer = Color(0xFF89F8C7),
    secondary = Color(0xFFB3CCC1),
    onSecondary = Color(0xFF1F352D),
    secondaryContainer = Color(0xFF354B43),
    onSecondaryContainer = Color(0xFFCFE9DD),
    tertiary = Color(0xFF9DC9E5),
    onTertiary = Color(0xFF003549),
    tertiaryContainer = Color(0xFF1F4B63),
    onTertiaryContainer = Color(0xFFC1E8FF),
    background = Color(0xFF161D1A),
    onBackground = Color(0xFFE1E3DF),
    surface = Color(0xFF161D1A),
    onSurface = Color(0xFFE1E3DF)
)
```

### Этап 4. Создание DataStore для хранения настроек (15 мин)

Создайте класс `SettingsManager` для управления настройками через DataStore. Создайте файл `SettingsManager.kt` в пакете `com.example.themeswitcher.data`:

```kotlin
package com.example.themeswitcher.data

import android.content.Context
import androidx.datastore.preferences.core.booleanPreferencesKey
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map

val Context.dataStore by preferencesDataStore(name = "settings")

class SettingsManager(private val context: Context) {
    
    companion object {
        val DARK_MODE_KEY = booleanPreferencesKey("dark_mode")
    }
    
    val isDarkMode: Flow<Boolean> = context.dataStore.data
        .map { preferences ->
            preferences[DARK_MODE_KEY] ?: false // по умолчанию светлая тема
        }
    
    suspend fun saveDarkMode(enabled: Boolean) {
        context.dataStore.edit { preferences ->
            preferences[DARK_MODE_KEY] = enabled
        }
    }
}
```

### Этап 5. Создание ViewModel для темы (10 мин)

Создайте `ThemeViewModel.kt` в пакете `com.example.themeswitcher.ui.theme`:

```kotlin
package com.example.themeswitcher.ui.theme

import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import com.example.themeswitcher.data.SettingsManager
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

class ThemeViewModel(
    private val settingsManager: SettingsManager
) : ViewModel() {
    
    val isDarkTheme: StateFlow<Boolean> = settingsManager.isDarkMode
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = false
        )
    
    fun toggleTheme() {
        viewModelScope.launch {
            val currentValue = isDarkTheme.value
            settingsManager.saveDarkMode(!currentValue)
        }
    }
}
```

### Этап 6. Создание кастомной темы приложения (10 мин)

Обновите файл `Theme.kt`, чтобы он использовал нашу ViewModel и DataStore. Добавьте поддержку динамических цветов для Android 12+ .

```kotlin
package com.example.themeswitcher.ui.theme

import android.app.Activity
import android.os.Build
import androidx.compose.foundation.isSystemInDarkTheme
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.runtime.SideEffect
import androidx.compose.ui.graphics.toArgb
import androidx.compose.ui.platform.LocalContext
import androidx.compose.ui.platform.LocalView
import androidx.core.view.WindowCompat
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun ThemeSwitcherTheme(
    viewModel: ThemeViewModel = viewModel(),
    content: @Composable () -> Unit
) {
    val context = LocalContext.current
    val isDarkTheme by viewModel.isDarkTheme.collectAsState()
    
    // Динамические цвета доступны на Android 12+
    val colorScheme = when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            if (isDarkTheme) dynamicDarkColorScheme(context) else dynamicLightColorScheme(context)
        }
        isDarkTheme -> DarkColors
        else -> LightColors
    }
    
    val view = LocalView.current
    if (!view.isInEditMode) {
        SideEffect {
            val window = (view.context as Activity).window
            window.statusBarColor = colorScheme.primary.toArgb()
            WindowCompat.getInsetsController(window, view).isAppearanceLightStatusBars = !isDarkTheme
        }
    }
    
    MaterialTheme(
        colorScheme = colorScheme,
        typography = Typography(),
        content = content
    )
}
```

### Этап 7. Создание главного экрана (15 мин)

Обновите `MainActivity.kt` и создайте простой интерфейс с переключателем темы.

```kotlin
package com.example.themeswitcher

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.viewmodel.compose.viewModel
import com.example.themeswitcher.data.SettingsManager
import com.example.themeswitcher.ui.theme.ThemeSwitcherTheme
import com.example.themeswitcher.ui.theme.ThemeViewModel

class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        
        // Инициализация SettingsManager
        val settingsManager = SettingsManager(this)
        
        setContent {
            ThemeSwitcherTheme(
                viewModel = viewModel(factory = ThemeViewModelFactory(settingsManager))
            ) {
                Surface(
                    modifier = Modifier.fillMaxSize(),
                    color = MaterialTheme.colorScheme.background
                ) {
                    ThemeScreen()
                }
            }
        }
    }
}

@Composable
fun ThemeScreen(viewModel: ThemeViewModel = viewModel()) {
    val isDarkTheme by viewModel.isDarkTheme.collectAsState()
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.Center
    ) {
        Text(
            text = "Текущая тема: ${if (isDarkTheme) "Тёмная" else "Светлая"}",
            style = MaterialTheme.typography.headlineMedium
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Button(
            onClick = { viewModel.toggleTheme() }
        ) {
            Text("Переключить тему")
        }
        
        Spacer(modifier = Modifier.height(32.dp))
        
        Card(
            modifier = Modifier.fillMaxWidth(),
            elevation = CardDefaults.cardElevation(defaultElevation = 4.dp)
        ) {
            Column(
                modifier = Modifier.padding(16.dp)
            ) {
                Text(
                    text = "Пример карточки",
                    style = MaterialTheme.typography.titleLarge
                )
                Text(
                    text = "Это демонстрация того, как тема влияет на цвета компонентов. " +
                            "Primary цвет: ${MaterialTheme.colorScheme.primary}",
                    style = MaterialTheme.typography.bodyMedium
                )
            }
        }
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Row(
            horizontalArrangement = Arrangement.spacedBy(8.dp)
        ) {
            Button(
                onClick = { /* Действие 1 */ },
                colors = ButtonDefaults.buttonColors(
                    containerColor = MaterialTheme.colorScheme.secondary
                )
            ) {
                Text("Кнопка 1")
            }
            
            OutlinedButton(
                onClick = { /* Действие 2 */ }
            ) {
                Text("Кнопка 2")
            }
        }
    }
}
```

### Этап 8. Создание фабрики для ViewModel (5 мин)

Создайте `ThemeViewModelFactory.kt` для передачи зависимости в ViewModel:

```kotlin
package com.example.themeswitcher.ui.theme

import androidx.lifecycle.ViewModel
import androidx.lifecycle.ViewModelProvider
import com.example.themeswitcher.data.SettingsManager

class ThemeViewModelFactory(
    private val settingsManager: SettingsManager
) : ViewModelProvider.Factory {
    
    override fun <T : ViewModel> create(modelClass: Class<T>): T {
        if (modelClass.isAssignableFrom(ThemeViewModel::class.java)) {
            @Suppress("UNCHECKED_CAST")
            return ThemeViewModel(settingsManager) as T
        }
        throw IllegalArgumentException("Unknown ViewModel class")
    }
}
```

### Этап 9. Запуск и тестирование (10 мин)

Запустите приложение. Проверьте:
- Переключение темы работает и интерфейс мгновенно обновляется.
- После закрытия и повторного открытия приложения выбранная тема сохраняется.
- На Android 12+ должны применяться динамические цвета на основе обоев.

### Этап 10. Добавление экрана настроек (оставшееся время, 5 мин)

Для более сложного примера можно создать отдельный экран настроек с переключателем темы, используя `Switch` или `RadioButton`.

```kotlin
@Composable
fun SettingsScreen(viewModel: ThemeViewModel = viewModel()) {
    val isDarkTheme by viewModel.isDarkTheme.collectAsState()
    
    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(16.dp)
    ) {
        Text(
            text = "Настройки",
            style = MaterialTheme.typography.headlineLarge
        )
        
        Spacer(modifier = Modifier.height(24.dp))
        
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text(
                text = "Тёмная тема",
                style = MaterialTheme.typography.bodyLarge
            )
            Switch(
                checked = isDarkTheme,
                onCheckedChange = { viewModel.toggleTheme() }
            )
        }
    }
}
```

---

## 4. Индивидуальные задания (вариативно)

Выберите одно из заданий для самостоятельной реализации:

1. **Три темы**  
   Добавьте третью тему, например "Системная" (следовать за системой). Реализуйте выбор через `RadioButton` и сохраняйте настройку.

2. **Выбор акцентного цвета**  
   Добавьте возможность выбора акцентного цвета (primary) из нескольких предустановленных вариантов. Сохраняйте выбор в DataStore и применяйте соответствующую цветовую схему.

3. **Предпросмотр темы**  
   Добавьте предварительный просмотр темы в настройках, показывающий примеры компонентов (кнопки, карточки, текст) в выбранной теме.

4. **Анимация смены темы**  
   Добавьте плавную анимацию при переключении темы (можно использовать `Crossfade`).

---

## 5. Контрольные вопросы

1. Как в Compose определить, какая тема активна в данный момент (тёмная/светлая)?
2. Что такое `MaterialTheme.colorScheme` и какие основные цвета он содержит?
3. Как сохранить выбор темы пользователя между сессиями работы приложения?
4. В чём разница между `isSystemInDarkTheme()` и сохранённым пользовательским выбором?
5. Что такое динамические цвета (dynamic color) и на каких версиях Android они доступны?

---

## 6. Требования к отчёту

Отчёт должен содержать:
- Титульный лист с названием работы, ФИО, группой.
- Цель работы.
- Листинги всех созданных файлов: `Color.kt`, `Theme.kt`, `SettingsManager.kt`, `ThemeViewModel.kt`, `MainActivity.kt`.
- Скриншоты приложения в светлой и тёмной темах.
- Ответы на контрольные вопросы.
- Вывод по работе (что нового узнали, как работает сохранение настроек).

---

## 7. Возможные ошибки и их решение

- **DataStore не работает** – убедитесь, что добавлена зависимость `implementation "androidx.datastore:datastore-preferences:1.0.0"` и выполнен Sync.
- **Тема не применяется при переключении** – проверьте, что `ThemeSwitcherTheme` обёрнут вокруг всего контента и что `collectAsState()` используется правильно.
- **Ошибка "Cannot create an instance of ViewModel"** – убедитесь, что фабрика правильно зарегистрирована и передана в `viewModel()`.
- **Динамические цвета не работают** – проверьте, что `Build.VERSION.SDK_INT >= Build.VERSION_CODES.S` и что на эмуляторе установлен Android 12+.

---

## 8. Дополнительные материалы

- [Material Theme Builder](https://m3.material.io/theme-builder) – для генерации цветовых схем .
- [Официальная документация по темам в Compose](https://developer.android.com/jetpack/compose/themes) .
- [Codelab: Material Theming with Jetpack Compose](https://developer.android.com/codelabs/basic-android-kotlin-compose-material-theming) .

**Успешной работы!**