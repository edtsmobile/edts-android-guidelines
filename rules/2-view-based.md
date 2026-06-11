# View-based Project — Structure & Rules

## Folder Structure (Multi-Module)

View-based projects follow a clean multi-module architecture:

```
[ProjectRoot]/
├── settings.gradle.kts
├── app/                                    # Thin application launcher
│   └── src/main/java/[package]/
│       └── App.kt                          # Koin initialization
├── core-domain/                            # Business Logic & Entities (Pure Kotlin)
│   └── src/main/java/[package]/core_domain/
│       ├── model/                          # Domain models
│       ├── repository/                     # Repository interfaces
│       └── use_case/                       # UseCase / Interactor classes
├── core-data/                              # Data Sources, Local DB, API (Android/Kotlin)
│   └── src/main/java/[package]/core_data/
│       ├── source/
│       │   ├── local/                      # Room databases/SharedPreferences
│       │   └── remote/                     # Retrofit API Services
│       ├── repository/                     # Repository implementations
│       └── mapper/                         # MapStruct Mapper interfaces
├── core-navigation/                        # Navigation Contracts & Helpers
│   └── src/main/java/[package]/navigation/
│       └── ModuleNavigator.kt              # Navigation interface
└── feature-[FeatureName]/                  # UI modules (XML + ViewBinding + ViewModels)
    └── src/main/
        ├── java/[package]/[feature]/
        │   ├── activity/                   # Activities
        │   ├── fragment/                   # Fragments (if any)
        │   └── viewmodel/                  # ViewModels
        └── res/
            └── layout/                     # ViewBinding layout XMLs
```

---

## 1. BaseActivity<VB> Template

Every Activity **must** extend `BaseActivity<VB>` (or a project-specific subclass such as `BaseLpiActivity<VB>`). Direct `AppCompatActivity` extension is forbidden.

```kotlin
package com.edts.mobile.core.ui.base

import android.os.Bundle
import android.view.LayoutInflater
import androidx.appcompat.app.AppCompatActivity
import androidx.viewbinding.ViewBinding

abstract class BaseActivity<VB : ViewBinding> : AppCompatActivity() {

    private var _binding: VB? = null
    protected val binding: VB get() = _binding!!

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        _binding = inflateBinding(layoutInflater)
        setContentView(binding.root)

        setupView()
        setupObserver()
        setupListener()
        initData()
    }

    abstract fun inflateBinding(inflater: LayoutInflater): VB
    abstract fun setupView()
    abstract fun setupObserver()
    abstract fun setupListener()
    abstract fun initData()

    override fun onDestroy() {
        super.onDestroy()
        _binding = null
    }
}
```

---

## 2. BaseViewModel Template

View-based ViewModels **must** use `LiveData` (or `MutableLiveData`) for outputs observed by Activities/Fragments.

```kotlin
package com.edts.mobile.core.ui.base

import androidx.lifecycle.LiveData
import androidx.lifecycle.MutableLiveData
import androidx.lifecycle.ViewModel

abstract class BaseViewModel : ViewModel() {
    // Common properties or helper states (e.g. tracking API requests)
}

// Feature ViewModel implementation:
class GetInfoViewModel(
    private val getInfoUseCase: GetInfoUseCase
) : BaseViewModel() {

    private val _infoState = MutableLiveData<Resource<InfoModel>>()
    val infoState: LiveData<Resource<InfoModel>> get() = _infoState

    fun fetchInfo(id: String) {
        _infoState.value = Resource.Loading()
        // Coroutine/RxJava flow
    }
}
```

---

## 3. Repository Interface & UseCase Template

- Repository interfaces (`I<Feature>Repository`) **must** live in `core-domain`.
- Repository implementations (`<Feature>Repository`) **must** live in `core-data`.
- Use cases (e.g., `GetInfoUseCase` interface and `GetInfoInteractor` implementation) **must** both live in `core-domain`.

### Domain Repository Interface (`core-domain`)

```kotlin
package com.edts.mobile.core_domain.repository

import kotlinx.coroutines.flow.Flow
import com.edts.mobile.core_domain.model.InfoModel

interface IGetInfoRepository {
    fun getInfo(id: String): Flow<Resource<InfoModel>>
}
```

### Domain UseCase & Interactor (`core-domain`)

```kotlin
package com.edts.mobile.core_domain.use_case.info

import kotlinx.coroutines.flow.Flow
import com.edts.mobile.core_domain.model.InfoModel
import com.edts.mobile.core_domain.repository.IGetInfoRepository

interface GetInfoUseCase {
    fun execute(id: String): Flow<Resource<InfoModel>>
}

class GetInfoInteractor(
    private val repository: IGetInfoRepository
) : GetInfoUseCase {
    override fun execute(id: String): Flow<Resource<InfoModel>> {
        return repository.getInfo(id)
    }
}
```

### Data Repository Implementation (`core-data`)

```kotlin
package com.edts.mobile.core_data.repository

import kotlinx.coroutines.flow.Flow
import com.edts.mobile.core_data.source.remote.InfoRemoteDataSource
import com.edts.mobile.core_domain.model.InfoModel
import com.edts.mobile.core_domain.repository.IGetInfoRepository

class GetInfoRepository(
    private val remoteDataSource: InfoRemoteDataSource,
    private val mapper: InfoMapper
) : IGetInfoRepository {
    override fun getInfo(id: String): Flow<Resource<InfoModel>> {
        // Implementation logic
    }
}
```

---

## 4. MapStruct @Mapper Template

All response-to-domain model mapping **must** use MapStruct `@Mapper` interfaces located in `core-data/mapper/`. Do not write manual extension mapping functions.

```kotlin
package com.edts.mobile.core_data.mapper

import org.mapstruct.Mapper
import org.mapstruct.Mapping
import org.mapstruct.factory.Mappers
import com.edts.mobile.core_data.model.InfoResponse
import com.edts.mobile.core_domain.model.InfoModel

@Mapper
interface InfoMapper {
    companion object {
        val INSTANCE: InfoMapper = Mappers.getMapper(InfoMapper::class.java)
    }

    @Mapping(target = "domainField", source = "apiField")
    fun mapResponseToModel(response: InfoResponse): InfoModel
}
```

---

## 5. Koin DI Configuration (Koin 4.1.0 DSL)

Koin registration **must** separate components strictly by scope/type:

```kotlin
import org.koin.core.module.dsl.viewModel
import org.koin.core.module.dsl.factoryOf
import org.koin.core.module.dsl.singleOf
import org.koin.dsl.bind
import org.koin.dsl.module

// Data Module (Data sources & Network)
val dataModule = module {
    single { InfoRemoteDataSource(get()) }
    single { InfoMapper.INSTANCE }
}

// Repository Module
val repositoryModule = module {
    singleOf(::GetInfoRepository) bind IGetInfoRepository::class
}

// UseCase Module
val useCaseModule = module {
    factoryOf(::GetInfoInteractor) bind GetInfoUseCase::class
}

// Feature UI Module
val infoFeatureModule = module {
    viewModel { GetInfoViewModel(get()) }
}
```

---

## 6. ModuleNavigator for Cross-Feature Navigation

Activities **must not** start other feature Activities directly using explicit Class references (avoid `Intent(this, DetailActivity::class.java)`). They must use `ModuleNavigator` declared in `core-navigation`.

```kotlin
package com.edts.mobile.navigation

import android.content.Context

interface ModuleNavigator {
    fun navigateToDetail(context: Context, itemId: String)
    fun navigateToProfile(context: Context)
}
```

---

## 7. NetworkBoundProcessResource Template

Use the standard `NetworkBoundProcessResource` implementation for resource loading and database caching coordination:

```kotlin
package com.edts.mobile.core_data.util

import kotlinx.coroutines.flow.*

abstract class NetworkBoundProcessResource<ResultType, RequestType> {

    fun asFlow(): Flow<Resource<ResultType>> = flow {
        emit(Resource.Loading())
        val dbSource = loadFromDb().firstOrNull()
        
        if (shouldFetch(dbSource)) {
            emit(Resource.Loading(dbSource))
            try {
                val apiResponse = fetchFromNetwork()
                saveNetworkResult(processResponse(apiResponse))
                emitAll(loadFromDb().map { Resource.Success(it) })
            } catch (throwable: Throwable) {
                onFetchFailed(throwable)
                emitAll(loadFromDb().map { Resource.Error(throwable.message ?: "Unknown Network Error", it) })
            }
        } else {
            emitAll(loadFromDb().map { Resource.Success(it) })
        }
    }

    protected abstract fun saveNetworkResult(item: RequestType)
    protected abstract fun shouldFetch(data: ResultType?): Boolean
    protected abstract fun loadFromDb(): Flow<ResultType>
    protected abstract suspend fun fetchFromNetwork(): RequestType
    protected open fun processResponse(response: RequestType): RequestType = response
    protected open fun onFetchFailed(throwable: Throwable) {}
}
```

---

## Rules

1. **ViewBinding only**: Direct `findViewById` is strictly forbidden. Always use `binding.viewId`.
2. **Base classes**: All Activities must extend `BaseActivity<VB>`. All Fragments must extend `BaseFragment<VB>`.
3. **LiveData observation**: Observe LiveData only in `setupObserver()`.
4. **Clean Architecture boundaries**: Repositories must use interfaces in `core-domain` and classes in `core-data`.
5. **No direct ViewModel execution**: Activities must not perform database or networking logic directly; all business logic must go through ViewModels and UseCases.
6. **No manual mapping**: Response to domain model mapping must use MapStruct.
7. **Cross-module separation**: Features must depend on navigation interfaces (`ModuleNavigator`) instead of referencing code from other features directly.
8. **Resource check**: Before creating a new service, datasource, or repository file, verify whether one exists. If it exists, ask: *"I found an existing `<FileName>` — should I add to that file or create a new one?"* Wait for developer confirmation.
