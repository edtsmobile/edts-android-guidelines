# Data Mapping Guidelines

This guide defines the standardized conventions for data mapping between layers in EDTS Android projects, ensuring proper isolation of Remote DTOs and Room Entities from Domain models.

---

## 1. Directory & Package Conventions

Mappers MUST be located under the `data/mapper/` package of their respective modules and partitioned by mapping domain:

- **Remote Mappers (`data/mapper/remote/`)**: Used for converting Remote DTO responses/requests into Domain models.
- **Local Mappers (`data/mapper/local/`)**: Used for converting database Room Entities into Domain models (and vice versa).

```
app/src/main/java/[package]/core/data/
├── mapper/
│   ├── remote/
│   │   ├── AuthMapper.kt
│   │   └── ProductMapper.kt
│   └── local/
│       └── UserLocalMapper.kt
```

---

## 2. Standard MapStruct Configuration

All complex data mapping classes SHOULD use MapStruct interfaces to perform compile-time data translation.

### Interface Template

Mappers must be declared as interfaces, annotated with `@Mapper`, and reference the companion `INSTANCE` accessor.

```kotlin
package id.co.edts.mobile.data.mapper.remote

import org.mapstruct.Mapper
import org.mapstruct.factory.Mappers
import id.co.edts.mobile.data.source.remote.response.ProductResponse
import id.co.edts.mobile.domain.model.Product

@Mapper
interface ProductMapper {
    companion object {
        val INSTANCE: ProductMapper = Mappers.getMapper(ProductMapper::class.java)
    }

    fun toDomain(response: ProductResponse): Product
    fun toDomainList(responseList: List<ProductResponse>): List<Product>
}
```

---

## 3. Handling Property Mismatch (`@Mapping`)

When source DTOs/Entities and target Domain model fields do not match in name or format, use the `@Mapping` annotation explicitly to guide the compiler.

```kotlin
@Mapper
interface UserLocalMapper {
    companion object {
        val INSTANCE: UserLocalMapper = Mappers.getMapper(UserLocalMapper::class.java)
    }

    @Mapping(source = "dbUserId", target = "id")
    @Mapping(source = "fullName", target = "name")
    fun toDomain(entity: UserEntity): User
}
```

---

## 4. Manual Mapping Fallback (Kotlin Extensions)

While MapStruct is required for complex nested objects and lists to prevent runtime errors, manual Kotlin extension functions are permitted and recommended for **simple, single-level primitive models** to keep compilation overhead low.

### Trivial Mapping Example

```kotlin
package id.co.edts.mobile.data.mapper.remote

import id.co.edts.mobile.data.source.remote.response.StatusResponse
import id.co.edts.mobile.domain.model.StatusInfo

// Trivial extension function mapping is allowed for simple models
fun StatusResponse.toDomain(): StatusInfo {
    return StatusInfo(
        code = this.code.orEmpty(),
        message = this.message.orEmpty()
    )
}
```
