- 👋 Hi, I’m Asamaning Redolf
- 📫 Email redolfkendrick@gmail.com
- 📫 linkedin.com/in/redolf250

![](https://komarev.com/ghpvc/?username=yourusername&color=green)


# Locking Mechanisms in ORMs: EF Core, TypeORM, Hibernate (JPA)

## Table of Contents

1. Introduction to Locking
2. Locking in EF Core
   - Pessimistic Locking
   - Optimistic Locking
3. Locking in TypeORM
   - Pessimistic Locking
   - Optimistic Locking
4. Locking in Hibernate (JPA with Spring Boot)
   - Pessimistic Locking
   - Optimistic Locking
5. Summary

---

## 1. Introduction to Locking

Locking is a technique to ensure data consistency and avoid race conditions in concurrent environments. There are two main types:

- **Pessimistic Locking**: Locks the data until a transaction completes.
- **Optimistic Locking**: Assumes low contention and uses version checking to prevent conflicts.

---

## 2. Locking in EF Core

### Pessimistic Locking

EF Core uses raw SQL or TransactionScope for explicit locks:

```csharp
using var context = new AppDbContext();
using var transaction = context.Database.BeginTransaction();

var product = context.Products
    .FromSqlRaw("SELECT * FROM Products WITH (UPDLOCK, ROWLOCK) WHERE Id = {0}", productId)
    .First();

product.Stock -= 10;
context.SaveChanges();
transaction.Commit();
```

### Optimistic Locking

Add a concurrency token to the model:

```csharp
public class Product {
    public int Id { get; set; }
    public int Stock { get; set; }

    [Timestamp]
    public byte[] RowVersion { get; set; }
}
```

EF Core will automatically throw `DbUpdateConcurrencyException` if a conflict is detected.

---

## 3. Locking in TypeORM

### Pessimistic Locking

```typescript
const product = await dataSource.getRepository(Product)
    .createQueryBuilder("product")
    .setLock("pessimistic_write")
    .where("product.id = :id", { id: 1 })
    .getOne();
```

Available lock modes:

- `pessimistic_read`
- `pessimistic_write`
- `dirty_read`
- `for_no_key_update`
- `for_key_share`

### Optimistic Locking

Add a `@VersionColumn`:

```typescript
@Entity()
export class Product {
  @PrimaryGeneratedColumn()
  id: number;

  @Column()
  stock: number;

  @VersionColumn()
  version: number;
}
```

Then use standard repository methods. TypeORM throws an error if versions mismatch.

---

## 4. Locking in Hibernate (JPA with Spring Boot)

### Pessimistic Locking

Use `@Lock` and JPQL:

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT p FROM Product p WHERE p.id = :id")
    Optional<Product> findByIdForUpdate(@Param("id") Long id);
}
```

Service usage:

```java
@Transactional
public void updateStock(Long id, int qty) {
    Product product = productRepository.findByIdForUpdate(id)
        .orElseThrow();
    product.setStock(product.getStock() - qty);
}
```

Set timeout (optional):

```java
@QueryHints({
    @QueryHint(name = "javax.persistence.lock.timeout", value = "5000")
})
```

### Optimistic Locking

Add `@Version` field:

```java
@Entity
public class Product {
    @Id
    private Long id;

    private int stock;

    @Version
    private int version;
}
```

On conflict, JPA throws `OptimisticLockException`.

---

## 5. Summary

| ORM       | Pessimistic Locking            | Optimistic Locking    |
| --------- | ------------------------------ | --------------------- |
| EF Core   | FromSqlRaw + UPDLOCK           | [Timestamp] attribute |
| TypeORM   | .setLock("pessimistic\_write") | @VersionColumn        |
| Hibernate | @Lock + LockModeType           | @Version              |

Each ORM provides both locking strategies. Choose **pessimistic** for critical sections and **optimistic** for performance-oriented scenarios with lower contention.


