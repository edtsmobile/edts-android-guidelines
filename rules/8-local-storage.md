# Local Storage Rules (Room Database)

This guideline defines the standard architecture and implementation details for local data caching using Room Database and Kotlin Symbol Processing (KSP).

---

## 1. Technology Stack (Room + KSP)

You must use the following stack for local database implementation:
- **Database Engine**: Room Database.
- **Symbol Processor**: KSP (Kotlin Symbol Processing). *KAPT is strictly forbidden.*
- **Reactive Queries**: Kotlin `Flow` for reactive UI state updates.
- **Dependency Injection**: Hilt/Koin based on the UI stack (Compose vs View-based).

---

## 2. Directory & Architecture Boundaries

Local storage implementations must reside strictly in the data module (typically `:core-data` or `:data` in multi-module, or `data/source/local` in single-module architectures).

- **FORBIDDEN**: Room `@Entity` data classes are exposed to the presentation layer or used directly in UI components.
- **FORBIDDEN**: Domain logic, UseCases, or ViewModels reference Room classes.
- **REQUIRED**: Database entities must be mapped to domain models in the repository before being exposed to outer layers.

---

## 3. Entity Setup & Domain Mapping

All entity classes must be annotated with `@Entity` and define mapping methods to transition data to the clean domain layer.

```kotlin
package com.edts.mobile.core_data.source.local.entity

import androidx.room.Entity
import androidx.room.PrimaryKey
import com.edts.mobile.core_domain.model.Product

@Entity(tableName = "products")
data class ProductEntity(
    @PrimaryKey
    val id: String,
    val name: String,
    val price: Double,
    val stock: Int
) {
    // Map database entity to domain model
    fun toDomain(): Product {
        return Product(
            id = id,
            name = name,
            price = price,
            stock = stock
        )
    }
}
```

---

## 4. DAO Setup (Reactive Queries)

DAO interfaces must specify local queries. To enable reactive state observation, all query/read operations **must** return a Kotlin `Flow`. Write/mutation operations must be declared as `suspend` functions.

```kotlin
package com.edts.mobile.core_data.source.local.dao

import androidx.room.Dao
import androidx.room.Insert
import androidx.room.OnConflictStrategy
import androidx.room.Query
import kotlinx.coroutines.flow.Flow
import com.edts.mobile.core_data.source.local.entity.ProductEntity

@Dao
interface ProductDao {

    // Reactive read query - returns Flow
    @Query("SELECT * FROM products")
    fun getAllProducts(): Flow<List<ProductEntity>>

    @Query("SELECT * FROM products WHERE id = :id")
    fun getProductById(id: String): Flow<ProductEntity?>

    // Suspend functions for write operations
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertProducts(products: List<ProductEntity>)

    @Query("DELETE FROM products WHERE id = :id")
    suspend fun deleteProduct(id: String)

    @Query("DELETE FROM products")
    suspend fun clearProducts()
}
```

---

## 5. Database Setup

The database class must extend `RoomDatabase` and be annotated with `@Database`.

```kotlin
package com.edts.mobile.core_data.source.local

import androidx.room.Database
import androidx.room.RoomDatabase
import com.edts.mobile.core_data.source.local.dao.ProductDao
import com.edts.mobile.core_data.source.local.entity.ProductEntity

@Database(
    entities = [ProductEntity::class],
    version = 1,
    exportSchema = false
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun productDao(): ProductDao
}
```

---

## 6. Repository Mapping Example

Repositories retrieve entity data from the DAO and use flow operations to map database entities to domain models before emitting them.

```kotlin
package com.edts.mobile.core_data.repository

import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import com.edts.mobile.core_data.source.local.dao.ProductDao
import com.edts.mobile.core_domain.model.Product
import com.edts.mobile.core_domain.repository.ProductRepository
import javax.inject.Inject
import javax.inject.Singleton

@Singleton
class ProductRepositoryImpl @Inject constructor(
    private val productDao: ProductDao
) : ProductRepository {

    override fun observeProducts(): Flow<List<Product>> {
        return productDao.getAllProducts().map { entities ->
            entities.map { it.toDomain() }
        }
    }
}
```

---

## Rules & Anti-Patterns

1. **Strict KSP Usage**: Ensure Room processor is declared as `ksp(libs.room.compiler)` instead of `kapt(...)` inside `build.gradle.kts`.
2. **Reactive Reads**: Never return raw lists (`List<Entity>`) for read queries. Use `Flow<List<Entity>>` so updates to the database are pushed reactively to observers.
3. **No Direct UI Usage**: Room annotations (`@Entity`, `@ColumnInfo`, `@Relation`) must never leak to ViewModels or Composables. Map them to domain models first.
4. **No UI Threads**: Ensure database writes and heavy read queries (non-flow queries) run on `Dispatchers.IO`.
5. **No Schema Hardcoding**: Keep schema definitions version-controlled. If schema export is enabled, specify the output folder in build scripts.
