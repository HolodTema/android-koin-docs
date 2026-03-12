## Введение в Koin

Koin - легковесный DI framework для Kotlin. По идее его можно применять в Android, KMP, Ktor, Spring и тд.

### Koin можно подключать через аннотации и через Kotlin DSL

Мы можем выбрать, что нам нравится - аннотации или kotlin dsl подход.

### Пример koin на аннотациях

```kotlin
@Singleton
class UserRepository(private val api: ApiService) {
    //...
}

@KoinViewModel
class UserViewModel(
    private val userRepository: UserRepository
) : ViewModel {
    //...
}

@Module
@ComponentScan("com.myapp")
class AppModule
```

### Пример koin на Kotlin DSL

```kotlin
class UserRepository(private val api: ApiService)

class UserViewModel(
    private val userRepository: UserRepository
) : ViewModel()


//создаем модуль через DSL-плагин kotlin
val appModule = module {
    single<ApiService>()

    single<UserRepository>()

    viewModel<UserViewModel>()
}

//Добавляем модуль при старте Koin
startKoin {
    modules(appModule)
}


//Делаем DI в активити
```kotlin
class MainActivity : AppCompatActivity() {

    private val viewModel : UserViewModel by viewModel()

    //...
}