# Лабораторная работа 14: Применение принципов SOLID в мобильной разработке

## Анализ нарушений SOLID в исходном коде

| Принцип | Нарушение в вашем коде | Проблема |
|---------|------------------------|----------|
| **SRP** | `MainActivity` и `NotesListScreen` напрямую работают с `NoteDao` | Экран отвечает и за UI, и за доступ к БД. При смене источника данных придётся править UI. |
| **OCP** | Жёсткая привязка к Room DAO | Добавление кэша или синхронизации с сервером потребует изменения экранов. |
| **LSP** | Нет интерфейса для подстановки моков | Нельзя протестировать экран без запуска реальной БД. |
| **ISP** | Экран зависит от всего `NoteDao`, хотя использует только `getAll()`, `insert()`, `delete()` | Изменение неиспользуемых методов (например, `update`) может сломать сборку. |
| **DIP** | UI получает DAO через `(context as NotesApp).database.noteDao()` | Зависимость от конкретной реализации Room, а не от абстракции. |

## Рефакторинг с применением SRP и DIP

### Добавленные файлы

| Файл | Назначение | Принцип |
|------|------------|---------|
| `data/repository/NoteRepository.kt` | Интерфейс репозитория (абстракция) | DIP |
| `data/repository/NoteRepositoryImpl.kt` | Реализация доступа к данным | SRP |
| `NotesApp.kt` (обновлён) | Единая точка создания зависимостей | SRP, DIP |
| `MainActivity.kt` (обновлён) | Использование репозитория вместо DAO | DIP |

## Реализация SRP

**`data/repository/NoteRepositoryImpl.kt`**

```kotlin
class NoteRepositoryImpl(
    private val noteDao: NoteDao
) : NoteRepository {
    override fun getAll(): Flow<List<NoteEntity>> =
        noteDao.getAll()
    override suspend fun insert(note: NoteEntity) =
        noteDao.insert(note)
    override suspend fun delete(note: NoteEntity) =
        noteDao.delete(note)
}
```
## Реализация DIP
### `data/repository/NoteRepository.kt`

```kotlin
interface NoteRepository {
    fun getAll(): Flow<List<NoteEntity>>
    suspend fun insert(note: NoteEntity): Long
    suspend fun delete(note: NoteEntity)
}
```
### MainActivity.kt

```kotlin
val repository = (application as NotesApp).noteRepository

LaunchedEffect(repository) {
    repository.getAll().collectLatest { entities ->
        notes.clear()
        notes.addAll(entities)
    }
}

scope.launch {
    repository.insert(NoteEntity(title = title, content = content))
}
```
### Обновление `NotesApp.kt`

```kotlin
class NotesApp : Application() {

    val database: AppDatabase by lazy {
        Room.databaseBuilder(this, AppDatabase::class.java, "notes_db")
            .fallbackToDestructiveMigration()
            .build()
    }

    // SRP + DIP: репозиторий создаётся здесь один раз
    val noteRepository: NoteRepository by lazy {
        NoteRepositoryImpl(database.noteDao())
    }
}
```
## Влияние рефакторинга на структуру кода

| Аспект | До рефакторинга | После рефакторинга |
|--------|----------------|-------------------|
| **Тестируемость** | Нельзя протестировать без БД | Можно подменить `NoteRepository` на мок |
| **Расширяемость** | Правка экранов для новых источников | Новые классы без изменения существующих |
| **Читаемость** | Логика БД размазана по экранам | Чёткое разделение: UI — Repository — DAO |
| **Сопровождение** | Изменение API БД ломает экраны | Экраны изолированы от изменений БД |

## Выводы

1. **SRP** был применён путём создания отдельного класса `NoteRepositoryImpl`, который отвечает только за доступ к данным. Экраны `MainActivity` и `NotesListScreen` теперь отвечают только за отображение UI и пользовательское взаимодействие.

2. **DIP** был применён путём введения интерфейса `NoteRepository`. Экраны теперь зависят от абстракции, а не от конкретной реализации Room DAO.

3. Структура проекта стала более модульной.

4. Функциональность приложения не изменилась.

5. **Перспективы**:
   - Кэширование
   - Синхронизация с сервером
   - Юнит-тесты с мок-репозиторием
