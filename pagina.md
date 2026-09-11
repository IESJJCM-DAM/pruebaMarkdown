# Arquitectura MVVM + Clean Architecture en Android

## Caso práctico: app de recetas *SaboreaAsturias*

> **Idea clave:** MVVM organiza la relación entre la interfaz y su estado; Clean Architecture organiza el conjunto de la aplicación en capas con responsabilidades y dependencias bien definidas.

---

## Índice

1. [Objetivos de aprendizaje](#1-objetivos-de-aprendizaje)
2. [Visión general de la arquitectura](#2-visión-general-de-la-arquitectura)
3. [Regla de dependencias](#3-regla-de-dependencias)
4. [Capa `data`](#4-capa-data)
5. [Capa `domain`](#5-capa-domain)
6. [Capa `presentation` o `feature`](#6-capa-presentation-o-feature)
7. [`core` y entrada de la aplicación](#7-core-y-entrada-de-la-aplicación)
8. [Estructura completa de paquetes](#8-estructura-completa-de-paquetes)
9. [Flujo de una operación](#9-flujo-de-una-operación)
10. [Conceptos que no debemos confundir](#10-conceptos-que-no-debemos-confundir)
11. [Resumen final](#11-resumen-final)

---

## 1. Objetivos de aprendizaje

Al finalizar este tema, el alumnado debería ser capaz de:

- identificar las capas `data`, `domain` y `presentation`;
- explicar la responsabilidad de cada capa;
- situar Retrofit, Room, Hilt y Jetpack Compose en la arquitectura;
- distinguir entre DTO, entidad de Room y modelo de dominio;
- diferenciar un repositorio de un DAO y de un *data source*;
- describir el recorrido de los datos desde una pantalla hasta la API o la base de datos;
- justificar por qué las dependencias apuntan hacia el dominio.

---

## 2. Visión general de la arquitectura

La aplicación utiliza dos ideas complementarias:

- **MVVM** dentro de la capa de presentación: `View` + `ViewModel` + estado de UI.
- **Clean Architecture** para separar la aplicación completa: `presentation`, `domain` y `data`.

Además, utiliza `core` para los recursos compartidos y varios archivos raíz para arrancar la aplicación.

### 2.1. Capas y responsabilidades

| Capa o bloque | Responsabilidad principal | Contenido habitual | Ejemplo en la app |
|---|---|---|---|
| `data` | Obtener, guardar y transformar datos. | DTO, API, DAO, entidades de Room, *data sources*, *mappers* e implementación de repositorios. | Descargar recetas y guardar favoritas. |
| `domain` | Representar los conceptos y operaciones centrales de la aplicación. | Modelos de dominio, contratos de repositorio y casos de uso. | `Recipe`, `RecipeRepository` y `ToggleFavoriteUseCase`. |
| `presentation` o `feature` | Mostrar información, recoger acciones y gestionar el estado de las pantallas. | Pantallas Compose, componentes, `ViewModel`, `UiState` y eventos. | `HomeScreen`, `HomeViewModel`, `HomeUiState` y `HomeEvent`. |
| `core` o `common` | Proporcionar elementos compartidos por distintas funcionalidades. | Hilt, navegación, tema, utilidades y componentes comunes. | `NetworkModule`, `AppNavigation` y `Theme`. |
| Entrada de la app | Iniciar Android y montar la interfaz general. | Clase `Application`, actividad y composable raíz. | `RecipesApplication.kt`, `MainActivity.kt` y `RecipesApp.kt`. |

### 2.2. Tecnologías utilizadas

| Tecnología | Papel en la aplicación | Zona habitual |
|---|---|---|
| Jetpack Compose | Construir la interfaz declarativa. | `presentation` / `feature` |
| ViewModel | Mantener el estado y procesar acciones de la UI. | `presentation` / `feature` |
| Retrofit | Realizar peticiones HTTP a la API. | `data/remote` |
| Room | Guardar y consultar datos locales. | `data/local` |
| Hilt | Crear e inyectar dependencias. | Configuración en `core/di`; uso transversal |

> **Importante:** Retrofit, Room y Hilt son herramientas, no capas arquitectónicas.

---

## 3. Regla de dependencias

La regla fundamental de Clean Architecture es que las capas externas pueden depender de las internas, pero el dominio no debe conocer los detalles externos.

```text
┌──────────────────────────────────────┐
│ PRESENTATION                         │
│ Compose · ViewModel · UiState        │
└─────────────────┬────────────────────┘
                  │ utiliza
                  ▼
┌──────────────────────────────────────┐
│ DOMAIN                               │
│ Modelos · contratos · casos de uso   │
└─────────────────▲────────────────────┘
                  │ implementa contratos
┌─────────────────┴────────────────────┐
│ DATA                                 │
│ Retrofit · Room · DTO · Entity       │
└──────────────────────────────────────┘
```

Por tanto:

- `presentation` utiliza los casos de uso y modelos de `domain`;
- `data` implementa los contratos declarados en `domain`;
- `domain` no conoce Retrofit, Room, Compose ni Android;
- `core` aloja infraestructura y recursos realmente compartidos.

Una forma breve de recordarlo es:

> **La presentación solicita una operación; el dominio define qué se necesita; la capa de datos resuelve cómo conseguirlo.**

---

## 4. Capa `data`

La capa `data` contiene los detalles técnicos necesarios para acceder a los datos. Puede trabajar con una fuente remota, como una API, y con una fuente local, como una base de datos Room.

### 4.1. Datos remotos: `data/remote`

#### `data/remote/dto/RecipeDto.kt`

- **Función:** representar una receta tal como llega en el JSON de la API.
- **Suele contener:** una `data class`, campos serializados y anotaciones como `@SerializedName` o `@SerialName`.
- **Detalle importante:** su estructura depende del contrato de la API, no de las necesidades de la interfaz.

#### `data/remote/dto/RecipesResponseDto.kt`

- **Función:** representar una respuesta que agrupa varias recetas.
- **Suele contener:** una lista de `RecipeDto` y, si existe, información de paginación o número total de resultados.

#### `data/remote/api/TheMealDbApi.kt`

- **Función:** declarar las operaciones HTTP que Retrofit debe implementar.
- **Suele contener:** funciones `suspend`, anotaciones `@GET`, `@POST`, `@Path` y `@Query`, y DTO como tipos de respuesta.

Ejemplo simplificado:

```kotlin
interface TheMealDbApi {
    @GET("recipes")
    suspend fun getRecipes(): RecipesResponseDto

    @GET("recipes/{id}")
    suspend fun getRecipeById(@Path("id") id: String): RecipeDto
}
```

#### `data/remote/datasource/RemoteRecipeDataSource.kt`

- **Función:** definir operaciones de alto nivel sobre la fuente remota sin exponer Retrofit al repositorio.
- **Suele contener:** métodos como `getRecipes()`, `getRecipeById()` o `searchRecipes()`.
- **Forma posible:** puede ser una interfaz o una clase, según la complejidad y las necesidades de prueba.

#### `data/remote/datasource/RemoteRecipeDataSourceImpl.kt`

- **Función:** implementar el acceso remoto mediante `TheMealDbApi`.
- **Suele contener:** una dependencia de la API, funciones `suspend` y tratamiento de respuestas o errores HTTP.
- **Resultado habitual:** devuelve DTO; la conversión al dominio se realiza posteriormente.

### 4.2. Datos locales: `data/local`

#### `data/local/database/AppDatabase.kt`

- **Función:** definir la base de datos Room principal.
- **Suele contener:** `@Database`, la lista de entidades, la versión de la base de datos y métodos abstractos que proporcionan los DAO.

```kotlin
@Database(
    entities = [FavoriteRecipeEntity::class],
    version = 1
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun favoriteRecipeDao(): FavoriteRecipeDao
}
```

#### `data/local/entity/FavoriteRecipeEntity.kt`

- **Función:** definir cómo se guarda una receta favorita en una tabla de Room.
- **Suele contener:** `@Entity`, `@PrimaryKey`, columnas y tipos compatibles con Room.
- **Detalle importante:** es un modelo de persistencia, no el modelo que debería utilizar la interfaz.

#### `data/local/dao/FavoriteRecipeDao.kt`

- **Función:** declarar las consultas y operaciones sobre la tabla de favoritos.
- **Suele contener:** `@Dao`, `@Query`, `@Insert`, `@Delete`, funciones `suspend` y resultados reactivos con `Flow`.

```kotlin
@Dao
interface FavoriteRecipeDao {
    @Query("SELECT * FROM favorite_recipes")
    fun observeFavorites(): Flow<List<FavoriteRecipeEntity>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(recipe: FavoriteRecipeEntity)

    @Delete
    suspend fun delete(recipe: FavoriteRecipeEntity)
}
```

#### `data/local/datasource/LocalRecipeDataSource.kt`

- **Función:** ocultar los detalles de Room al repositorio.
- **Suele contener:** operaciones como `observeFavorites()`, `isFavorite(id)`, `insertFavorite()` y `deleteFavorite()`.
- **Ventaja:** el repositorio no necesita conocer cada consulta concreta del DAO.

### 4.3. Conversión de modelos: `data/mapper`

#### `data/mapper/RecipeMapper.kt`

- **Función:** convertir modelos entre las distintas capas.
- **Conversiones habituales:**
  - `RecipeDto` → `Recipe`;
  - `FavoriteRecipeEntity` → `Recipe`;
  - `Recipe` → `FavoriteRecipeEntity`.

```kotlin
fun RecipeDto.toDomain(): Recipe = Recipe(
    id = id,
    name = name,
    imageUrl = imageUrl
)

fun Recipe.toEntity(): FavoriteRecipeEntity = FavoriteRecipeEntity(
    id = id,
    name = name,
    imageUrl = imageUrl
)
```

Los *mappers* evitan que los cambios en el JSON o en la base de datos se propaguen directamente al resto de la aplicación.

### 4.4. Implementación del repositorio: `data/repository`

#### `data/repository/RecipeRepositoryImpl.kt`

- **Función:** implementar el contrato `RecipeRepository` definido en `domain`.
- **Suele contener:** los *data sources* remoto y local, los *mappers*, estrategias de caché y conversión de errores.
- **Responsabilidad clave:** decidir de qué fuente obtener los datos y cómo combinarlos.

```kotlin
class RecipeRepositoryImpl(
    private val remoteDataSource: RemoteRecipeDataSource,
    private val localDataSource: LocalRecipeDataSource
) : RecipeRepository {

    override suspend fun getRecipes(): List<Recipe> =
        remoteDataSource.getRecipes().map { it.toDomain() }
}
```

---

## 5. Capa `domain`

La capa `domain` representa el núcleo funcional de la aplicación. Debe ser independiente de Android y de herramientas concretas como Retrofit o Room.

### 5.1. Modelos: `domain/model`

#### `domain/model/Recipe.kt`

- **Función:** representar el concepto de receta que necesita la aplicación.
- **Suele contener:** una `data class` limpia con propiedades como `id`, `name`, `imageUrl`, `ingredients`, `instructions` e `isFavorite`.
- **No debería contener:** anotaciones de Retrofit, serialización o Room.

#### `domain/model/RecipeIngredient.kt`

- **Función:** representar un ingrediente de una receta.
- **Suele contener:** propiedades como `name` y `measure`.

### 5.2. Contrato del repositorio: `domain/repository`

#### `domain/repository/RecipeRepository.kt`

- **Función:** definir las operaciones de datos que necesita el dominio.
- **Suele contener:** funciones como `getRecipes()`, `getRecipeById()`, `observeFavorites()` y `toggleFavorite()`.
- **No contiene:** Retrofit, Room ni detalles sobre la procedencia de los datos.

```kotlin
interface RecipeRepository {
    suspend fun getRecipes(): List<Recipe>
    suspend fun getRecipeById(id: String): Recipe
    fun observeFavorites(): Flow<List<Recipe>>
    suspend fun toggleFavorite(recipe: Recipe)
}
```

La interfaz se sitúa en `domain` y su implementación, `RecipeRepositoryImpl`, se sitúa en `data`. Esta inversión permite que el dominio no dependa de la infraestructura.

### 5.3. Casos de uso: `domain/usecase`

Un caso de uso representa **una intención del usuario o una operación del sistema**. Normalmente depende de un repositorio y expone una única operación mediante `operator fun invoke(...)`.

| Archivo | Operación representada | Contenido habitual |
|---|---|---|
| `GetRecipesUseCase.kt` | Obtener el listado principal. | Llamada al repositorio, validación, filtrado u ordenación. |
| `GetGlutenFreeRecipesUseCase.kt` | Obtener o filtrar recetas sin gluten. | Reglas de filtrado y acceso a `RecipeRepository`. |
| `GetRecipeByIdUseCase.kt` | Recuperar una receta por su identificador. | Recepción del `id` y llamada al repositorio. |
| `GetFavoriteRecipesUseCase.kt` | Observar las recetas favoritas. | Normalmente devuelve `Flow<List<Recipe>>`. |
| `ToggleFavoriteUseCase.kt` | Añadir o retirar una receta de favoritos. | Reglas de la operación y llamada al repositorio. |

Ejemplo:

```kotlin
class GetRecipesUseCase(
    private val repository: RecipeRepository
) {
    suspend operator fun invoke(): List<Recipe> =
        repository.getRecipes()
}
```

> No es necesario crear un caso de uso para cada función trivial. Es especialmente útil cuando la operación contiene reglas, se reutiliza o representa claramente una acción del sistema.

---

## 6. Capa `presentation` o `feature`

La presentación se organiza **por funcionalidad** en lugar de separar globalmente todas las pantallas, todos los estados y todos los `ViewModel`.

```text
feature/
├── home/
├── detail/
├── favorites/
└── today/
```

Esta organización mantiene juntos todos los archivos relacionados con una pantalla.

### 6.1. Patrón común de una funcionalidad

Cada funcionalidad puede seguir esta estructura:

```text
feature/home/
├── components/
│   └── RecipeCard.kt
├── HomeScreen.kt
├── HomeContent.kt
├── HomeUiState.kt
├── HomeEvent.kt
└── HomeViewModel.kt
```

| Elemento | Responsabilidad |
|---|---|
| `Screen` | Conectar Compose con el `ViewModel`, observar el estado y gestionar navegación o efectos. |
| `Content` | Dibujar la interfaz únicamente a partir del estado y de funciones de callback. |
| `UiState` | Describir toda la información necesaria para representar la pantalla. |
| `Event` | Representar las acciones que pueden producirse en la interfaz. |
| `ViewModel` | Procesar eventos, ejecutar casos de uso y actualizar el estado. |
| `components/` | Contener composables propios de esa funcionalidad. |

### 6.2. Funcionalidad Home

#### `HomeScreen.kt`

- **Función:** ser el punto de entrada Compose de Home.
- **Suele contener:** obtención del `ViewModel`, observación con `collectAsStateWithLifecycle()`, efectos y llamada a `HomeContent`.

#### `HomeContent.kt`

- **Función:** dibujar la pantalla a partir de un estado recibido.
- **Suele contener:** composables sin acceso directo al repositorio ni a los casos de uso.
- **Parámetros habituales:** `state` y `onEvent`.

#### `components/RecipeCard.kt`

- **Función:** mostrar el resumen visual de una receta.
- **Suele contener:** imagen, título, categorías, botón de favorito y callbacks de pulsación.

#### `HomeUiState.kt`

- **Función:** representar todo el estado visible de Home.
- **Suele contener:** `isLoading`, `recipes`, `errorMessage` y filtros seleccionados.

```kotlin
data class HomeUiState(
    val isLoading: Boolean = false,
    val recipes: List<Recipe> = emptyList(),
    val errorMessage: String? = null
)
```

#### `HomeEvent.kt`

- **Función:** enumerar las acciones que puede procesar el `ViewModel`.
- **Suele contener:** una `sealed interface` o `sealed class`.

```kotlin
sealed interface HomeEvent {
    data class RecipeClicked(val id: String) : HomeEvent
    data class FavoriteClicked(val recipe: Recipe) : HomeEvent
    data object RetryClicked : HomeEvent
}
```

#### `HomeViewModel.kt`

- **Función:** ejecutar casos de uso y convertir sus resultados en `HomeUiState`.
- **Suele contener:** `@HiltViewModel`, `StateFlow`, `viewModelScope`, casos de uso inyectados y una función `onEvent()`.

### 6.3. Separación entre `Screen` y `Content`

```kotlin
@Composable
fun HomeScreen(
    viewModel: HomeViewModel = hiltViewModel()
) {
    val state by viewModel.uiState.collectAsStateWithLifecycle()

    HomeContent(
        state = state,
        onEvent = viewModel::onEvent
    )
}
```

- `HomeScreen` conecta Compose con el `ViewModel`.
- `HomeContent` solo dibuja.
- `HomeUiState` contiene los datos necesarios para dibujar.
- `HomeEvent` representa lo que hace el usuario.
- `HomeViewModel` procesa los eventos y modifica el estado.

Esta separación facilita las *previews*, las pruebas y la reutilización de la interfaz.

### 6.4. Resto de funcionalidades

| Funcionalidad | Archivos principales | Responsabilidad |
|---|---|---|
| Detalle | `DetailScreen.kt`, `DetailUiState.kt`, `DetailEvent.kt`, `DetailViewModel.kt` | Cargar una receta por su `id`, mostrar ingredientes e instrucciones y cambiar el favorito. |
| Favoritos | `FavoritesScreen.kt`, `FavoritesUiState.kt`, `FavoritesEvent.kt`, `FavoritesViewModel.kt` | Observar y mostrar las recetas guardadas en Room. |
| Hoy | `TodayScreen.kt`, `TodayUiState.kt`, `TodayEvent.kt`, `TodayViewModel.kt` | Elegir o cargar la propuesta diaria y representar sus estados. |

Archivos complementarios:

- `feature/detail/components/IngredientItem.kt`: muestra un ingrediente y su cantidad.
- `DetailViewModel.kt`: puede utilizar `SavedStateHandle`, `GetRecipeByIdUseCase`, `ToggleFavoriteUseCase` y `StateFlow`.
- `FavoritesViewModel.kt`: utiliza `GetFavoriteRecipesUseCase` y observa un flujo de favoritos.
- `TodayViewModel.kt`: coordina los casos de uso necesarios, corrutinas y tratamiento de errores.

---

## 7. `core` y entrada de la aplicación

`core` reúne recursos utilizados por varias funcionalidades. No debe convertirse en un lugar donde guardar cualquier archivo que no sepamos clasificar.

### 7.1. Inyección de dependencias: `core/di`

| Archivo | Función | Contenido habitual |
|---|---|---|
| `AppModule.kt` | Proporcionar dependencias generales. | Métodos `@Provides`, ámbitos como `@Singleton` y dependencias compartidas. |
| `NetworkModule.kt` | Crear los objetos de red. | `OkHttpClient`, `Retrofit`, URL base, conversor JSON y `TheMealDbApi`. |
| `DatabaseModule.kt` | Crear la base de datos y los DAO. | `Room.databaseBuilder(...)` y métodos `@Provides`. |
| `RepositoryModule.kt` | Relacionar contratos con implementaciones. | Un método `@Binds` entre `RecipeRepository` y `RecipeRepositoryImpl`. |

Ejemplo de enlace con Hilt:

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    abstract fun bindRecipeRepository(
        implementation: RecipeRepositoryImpl
    ): RecipeRepository
}
```

Hilt se encarga de crear y conectar los objetos, pero no sustituye a ninguna capa.

### 7.2. Navegación: `core/navigation`

| Archivo | Función | Contenido habitual |
|---|---|---|
| `AppDestination.kt` | Definir los destinos de la aplicación. | Rutas para Home, detalle, favoritos y hoy, con sus argumentos. |
| `AppNavigation.kt` | Construir el grafo de navegación. | `NavHost`, destinos `composable`, argumentos y callbacks. |
| `BottomNavigationBar.kt` | Mostrar la navegación inferior. | Iconos, rutas y estado del destino seleccionado. |

### 7.3. Interfaz compartida: `core/ui`

| Paquete o archivo | Función |
|---|---|
| `core/ui/theme/Color.kt` | Declarar la paleta de colores. |
| `core/ui/theme/Theme.kt` | Aplicar colores, tipografía y formas mediante `MaterialTheme`. |
| `core/ui/theme/Type.kt` | Centralizar `Typography` y los estilos de texto. |
| `core/ui/components/LoadingIndicator.kt` | Proporcionar un indicador de carga reutilizable. |

Un componente se sitúa en `core/ui/components` solo si lo utilizan varias funcionalidades. Si únicamente pertenece a Home, debe permanecer en `feature/home/components`.

### 7.4. Utilidades: `core/util`

#### `Constants.kt`

- **Función:** centralizar constantes globales justificadas, como una URL base o determinadas claves de configuración.
- **Precaución:** no debe convertirse en un “cajón de sastre” de valores sin relación.

### 7.5. Entrada de la aplicación

| Archivo | Función | Contenido habitual |
|---|---|---|
| `RecipesApplication.kt` | Inicializar la aplicación y activar Hilt. | Clase `Application` anotada con `@HiltAndroidApp`. |
| `MainActivity.kt` | Iniciar la interfaz Compose. | `@AndroidEntryPoint`, `setContent`, tema y llamada a `RecipesApp`. |
| `RecipesApp.kt` | Componer la estructura visual global. | `NavController`, `Scaffold`, barra inferior y `AppNavigation`. |

`RecipesApp.kt` es opcional, pero permite que `MainActivity.kt` permanezca pequeña y centrada en iniciar la aplicación.

---

## 8. Estructura completa de paquetes

La siguiente estructura integra los paquetes y archivos del ejemplo:

```text
com.example.recipes/
├── data/
│   ├── remote/
│   │   ├── dto/
│   │   │   ├── RecipeDto.kt
│   │   │   └── RecipesResponseDto.kt
│   │   ├── api/
│   │   │   └── TheMealDbApi.kt
│   │   └── datasource/
│   │       ├── RemoteRecipeDataSource.kt
│   │       └── RemoteRecipeDataSourceImpl.kt
│   ├── local/
│   │   ├── database/
│   │   │   └── AppDatabase.kt
│   │   ├── dao/
│   │   │   └── FavoriteRecipeDao.kt
│   │   ├── entity/
│   │   │   └── FavoriteRecipeEntity.kt
│   │   └── datasource/
│   │       └── LocalRecipeDataSource.kt
│   ├── mapper/
│   │   └── RecipeMapper.kt
│   └── repository/
│       └── RecipeRepositoryImpl.kt
│
├── domain/
│   ├── model/
│   │   ├── Recipe.kt
│   │   └── RecipeIngredient.kt
│   ├── repository/
│   │   └── RecipeRepository.kt
│   └── usecase/
│       ├── GetRecipesUseCase.kt
│       ├── GetGlutenFreeRecipesUseCase.kt
│       ├── GetRecipeByIdUseCase.kt
│       ├── GetFavoriteRecipesUseCase.kt
│       └── ToggleFavoriteUseCase.kt
│
├── feature/
│   ├── home/
│   │   ├── components/
│   │   │   └── RecipeCard.kt
│   │   ├── HomeScreen.kt
│   │   ├── HomeContent.kt
│   │   ├── HomeUiState.kt
│   │   ├── HomeEvent.kt
│   │   └── HomeViewModel.kt
│   ├── detail/
│   │   ├── components/
│   │   │   └── IngredientItem.kt
│   │   ├── DetailScreen.kt
│   │   ├── DetailUiState.kt
│   │   ├── DetailEvent.kt
│   │   └── DetailViewModel.kt
│   ├── favorites/
│   │   ├── FavoritesScreen.kt
│   │   ├── FavoritesUiState.kt
│   │   ├── FavoritesEvent.kt
│   │   └── FavoritesViewModel.kt
│   └── today/
│       ├── TodayScreen.kt
│       ├── TodayUiState.kt
│       ├── TodayEvent.kt
│       └── TodayViewModel.kt
│
├── core/
│   ├── di/
│   │   ├── AppModule.kt
│   │   ├── NetworkModule.kt
│   │   ├── DatabaseModule.kt
│   │   └── RepositoryModule.kt
│   ├── navigation/
│   │   ├── AppDestination.kt
│   │   ├── AppNavigation.kt
│   │   └── BottomNavigationBar.kt
│   ├── ui/
│   │   ├── components/
│   │   │   └── LoadingIndicator.kt
│   │   └── theme/
│   │       ├── Color.kt
│   │       ├── Theme.kt
│   │       └── Type.kt
│   └── util/
│       └── Constants.kt
│
├── RecipesApplication.kt
├── MainActivity.kt
└── RecipesApp.kt
```

---

## 9. Flujo de una operación

### 9.1. Cargar las recetas en Home

| Paso | Elemento | Responsabilidad |
|---:|---|---|
| 1 | `HomeScreen` | Observa `HomeUiState` y muestra `HomeContent`. |
| 2 | `HomeViewModel` | Recibe la petición de carga y ejecuta `GetRecipesUseCase`. |
| 3 | `GetRecipesUseCase` | Aplica las reglas necesarias y llama a `RecipeRepository`. |
| 4 | `RecipeRepositoryImpl` | Decide obtener los datos desde la fuente remota. |
| 5 | `RemoteRecipeDataSource` | Solicita los datos a `TheMealDbApi`. |
| 6 | `TheMealDbApi` | Retrofit realiza la petición HTTP y devuelve DTO. |
| 7 | `RecipeMapper` | Convierte `RecipeDto` en el modelo de dominio `Recipe`. |
| 8 | `RecipeRepositoryImpl` | Devuelve los modelos de dominio al caso de uso. |
| 9 | `HomeViewModel` | Actualiza el `StateFlow<HomeUiState>`. |
| 10 | Compose | Detecta el nuevo estado y recompone `HomeContent`. |

Flujo resumido:

```text
HomeScreen
    → HomeViewModel
    → GetRecipesUseCase
    → RecipeRepository
    → RecipeRepositoryImpl
    → RemoteRecipeDataSource
    → TheMealDbApi
    → Retrofit / Internet
```

El resultado recorre después el camino inverso hasta que Compose recibe un nuevo `HomeUiState`.

### 9.2. Marcar una receta como favorita

```text
RecipeCard
    → HomeEvent.FavoriteClicked
    → HomeViewModel
    → ToggleFavoriteUseCase
    → RecipeRepository
    → RecipeRepositoryImpl
    → LocalRecipeDataSource
    → FavoriteRecipeDao
    → Room
```

En este recorrido:

1. la interfaz genera un evento;
2. el `ViewModel` delega la operación en un caso de uso;
3. el caso de uso utiliza el contrato del repositorio;
4. la implementación del repositorio selecciona la fuente local;
5. el DAO ejecuta la operación sobre Room.

---

## 10. Conceptos que no debemos confundir

| Conceptos | Diferencia |
|---|---|
| `RecipeDto` y `Recipe` | El DTO representa el formato de Internet; `Recipe` representa el concepto que necesita la aplicación. |
| `FavoriteRecipeEntity` y `Recipe` | La entidad representa una tabla de Room; `Recipe` es independiente del almacenamiento. |
| `RecipeRepository` y `RecipeRepositoryImpl` | El primero es el contrato definido en `domain`; el segundo lo implementa en `data`. |
| DAO y repositorio | El DAO opera solamente sobre Room; el repositorio coordina una o varias fuentes de datos. |
| *Data source* y repositorio | El *data source* encapsula una fuente concreta; el repositorio decide cuáles utilizar y cómo combinarlas. |
| `Screen` y `Content` | `Screen` conecta estado, `ViewModel` y navegación; `Content` dibuja la interfaz. |
| `UiState` y `Event` | El estado describe cómo está la pantalla; el evento describe algo que ha ocurrido. |
| `ViewModel` y caso de uso | El `ViewModel` gestiona la UI; el caso de uso representa una operación o regla de la aplicación. |
| `core/ui/components` y `feature/.../components` | `core` aloja componentes compartidos; la *feature* conserva los componentes exclusivos de su pantalla. |
| Hilt y las capas | Hilt no es una capa: es el mecanismo que crea y conecta dependencias. |

---

## 11. Resumen final

### ¿Dónde colocar cada elemento?

| Si el elemento... | Debe situarse normalmente en... |
|---|---|
| representa el JSON de una API | `data/remote/dto` |
| declara endpoints de Retrofit | `data/remote/api` |
| representa una tabla de Room | `data/local/entity` |
| contiene consultas de Room | `data/local/dao` |
| convierte DTO, entidades y modelos | `data/mapper` |
| implementa un repositorio | `data/repository` |
| representa un concepto del negocio | `domain/model` |
| define las operaciones del repositorio | `domain/repository` |
| representa una acción de la aplicación | `domain/usecase` |
| gestiona el estado de una pantalla | `feature/<pantalla>` |
| es un composable usado por varias pantallas | `core/ui/components` |
| configura Retrofit, Room o Hilt | `core/di` |
| define rutas y grafo de navegación | `core/navigation` |

### Frases para recordar

> **MVVM organiza la pantalla; Clean Architecture organiza la aplicación completa.**

> **`domain` expresa qué necesita la aplicación; `data` resuelve cómo obtenerlo; `presentation` decide cómo mostrarlo.**

> **Los DTO pertenecen a la API, las entidades pertenecen a Room y los modelos de dominio pertenecen a la aplicación.**

