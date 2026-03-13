## Что такое DI - объяснение от Koin official docs

https://insert-koin.io/docs/intro/what-is-dependency-injection/

DI - dependency injection - подход, когда объекты получают свои зависимости извне, а не создают их внутри себя.

DI решает проблему сильных связей, облегчает тестирование, делает код чище.

### Что такое зависимость (dependency)
Зависимость - объект, без которого другой объект не может работать.

Например, класс Car не сможет работать без класса Engine у себя внутри.

Поэтому Engine - зависимость для Car

Car - зависимый класс.

И этот самый Car может создавать объект Engine внутри себя, что плохо. А может получать объект Engine извне - это и есть DI.

### Пример с классами Car и Engine

До внедрения DI:

- Car жестко зависит от Engine - не хорошо

- Сложно тестировать Car

- Сложно поменять тип движка на другой (ElectricEngine, DieselEngine и тд)

- Car жестко определяет как создавать Engine-объект

```kotlin
class Engine {
    fun start() {
        println("Engine is starting")
    }
}

class Car {
    //not good, we need DI here!
    private val engine = Engine()

    fun drive() {
        engine.start()
        println("Car is driving")
    }
}
```

После внедрения DI - топ

```kotlin
class Car(
    private val engine: Engine
) {
    fun drive() {
        engine.start()
        println("Car is driving")
    }
}

val car1 = Car(ElectricEngine())

val car2 = Car(GasEngine())
```

Преимущества кода с DI:

- Car не знает, как был создан Engine-объект
- Легко тестировать, мокая Engine
- Можно менять типы Engine
- Зависимости класса видны в его конструкторе - самодокументируемый код.

## Три способа предоставлять зависимости

### 1. constructor injection

Constructor injection - наиболее рекомендованный способ DI в Koin

```kotlin
Class UserRepository(
    private val db: Database,
    private val apiClient: ApiClient
) {
    //...
}

//и в Koin делаем модуль
val appModule = module {
    single<Database>()
    single<ApiClient>()
    single<UserRepository>()
}
```

### 2. field injection
field injection применяется, когда мы не можем изменить конструктор класса. Например, нам нельзя трогать конструкторы Activity, Fragment, Service - мы делаем в них Field injection.

ЗЫ. можно делать ленивую field injection через делегат by

```kotlin
class UserActivity : AppCompatActivity() {
    //lazy field injection
    //то есть field injection случится при первом обращении к ViewModel,
    private val viewModel: UserViewModel by viewModel()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        viewModel.loadUser()
        // ViewModel instance created here
    }
}
```

В koin lazy field injection делается через делегат

А eager field injection, то есть инъекция моментальная, при создании объекта класса - она через простое равно.

Например, вот как сделать lazy или eager field injection для некоего класса Presenter
```kotlin
// Lazy injection
val presenter: Presenter by inject()

// Eager injection
val presenter: Presenter = get()
```

### 3. Method injection
Это наименее распространенный способ.

Юзается в колбэках, в необязательных для работы всего класса зависимостях и тд

```kotlin
class ReportGenerator {
    fun generateReport(data: DataSource) {
        //...
    }
}
```

## Ручной DI, без фреймворков
Это можно, но это долго и бойлерплейт

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        //делаем DI вручную
        val db = Database()
        val apiClient = ApiClient()
        val userRepository = UserRepository(db, apiClient)
        val viewModel = UserViewModel(userRepository)

        //и только после этого можно юзать viewModel и тд
    }
}
```

Минусы:
- код Activity или Fragment засоряется
- легко наделать ошибок в ручном создании такого dependency graph
- сложно поддерживать с ростом кодовой базы
- сложно управлять ЖЦ зависимостей (синглтоны и другие скоупы)

### Ручной DI через Container-паттерн проектирования DI

```kotlin
object AppContainer {
    //эти зависимости singleton-scoped
    private val db by lazy {
        Database()
    }
    private val apiClient by lazy {
        ApiClient()
    }
    val userRepository by lazy {
        UserRepository(db, apiClient)
    }

    //viewModel без scope, создается при каждом вызове метода
    fun createUserViewModel(): UserViewModel {
        return UserViewModel(userRepository)
    }
}

// Usage
class MainActivity : AppCompatActivity() {
    private val viewModel = AppContainer.createUserViewModel()
}
```

Уже лучше, но все еще есть проблемы
- сложно управлять ЖЦ зависимостей (кроме singleton и no-scoped)
- сложно поддерживать с ростом кодовой базы
- леко наделать ошибок

## Как Koin внедряет DI автоматически

```kotlin
//один раз определяем зависимости в модуле
val appModule = module {
    single<Database>()
    single<ApiClient>()
    single<UserRepository>
    viewModel<UserViewModel>
}


//потом надо в application-классе
//запустить koin
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()
        startKoin {
            modules(appModule)
        }
    }
}


//и потом можно юзать DI везде
class MainActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModel()

    //...
}
```

## Сравнение Koin, Dagger/Hilt и ручного DI

Dagger/Hilt юзают кодогенерацию на аннотациях - это compile-time-DI. Сложно подключать в проект, но быстро работает.

Koin использует рефлексию в рантайме. Легко подключать в проект, но медленнее работает.

Koin может делать DI и через Kotlin DSL, и через аннотации. Мы просто выбираем, что удобнее.

## ServiceLocator - аналог DI
У нас есть класс с зависимости.

Мы наследуем этот класс от KoinComponent и создаем зависимости внутри класса через inject() функцию

```kotlin
//ServiceLocator
class UserService : KoinComponent {
    private val repository: UserRepository by inject()
}
```

Если бы мы делали DI, то было бы так:
```kotlin
//DI constructor injection
class UserService(
    private val repository: UserRepository
)
```

## KOIN BEST PRACTICES

### 1. предпочитаем constructor injection в большинстве случаев

Ибо constructor injection можно тестить без koin

```kotlin
// Good - testable without Koin
class UserViewModel(private val userService: UserService) : ViewModel()

// Koin resolves dependencies
val appModule = module {
    viewModel<UserViewModel>()

}
```

### 2. Избегаем KoinComponent в бизнес-логике
Ибо иначе будет сложно тестировать такой код

```kotlin
// Bad - hard to test
class UserService : KoinComponent {
    private val repository: UserRepository = get()
}

// Good - explicit dependencies
class UserService(private val repository: UserRepository)
```

### 3. ServiceLocator паттерн DI использовать только по крайней необходимости
