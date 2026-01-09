# EF Core Learning: Foreign Keys Referencing Non-Primary Keys

## 📚 Concept Overview

**When a foreign key references a column OTHER than the primary key, you must explicitly configure the relationship using `HasPrincipalKey()`.**

---

## 🎯 Real-World Scenario: E-Commerce System

### Database Schema
```sql
-- Parent Table: Products
CREATE TABLE products (
    id INT PRIMARY KEY,                     -- Primary Key
    sku VARCHAR(50) NOT NULL UNIQUE,        -- Alternate Unique Key (Stock Keeping Unit)
    name VARCHAR(200),
    price DECIMAL(10,2)
);

-- Child Table: Order Items
CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT NOT NULL,
    product_sku VARCHAR(50) NOT NULL,       -- Foreign Key
    quantity INT,
    FOREIGN KEY (product_sku) REFERENCES products(sku)  -- References SKU, not ID!
);
```

**Note:** The FK references `sku`, NOT `id`!

### Why This Design?
- **Business Reason**: External systems (warehouses, suppliers) use SKU, not internal ID
- **Data Integrity**: SKU is the natural business key
- **Real-world constraint**: Can't change how other systems work

---

## ❌ Problem: Default EF Core Behavior

### What EF Core Assumes (Without Configuration)
```csharp
public class Product
{
    public int Id { get; set; }              // Primary Key
    public string Sku { get; set; }          // Just another property
    public string Name { get; set; }
    public decimal Price { get; set; }
    public List<OrderItem> OrderItems { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public string ProductSku { get; set; }   // Foreign Key
    public int Quantity { get; set; }
    public Product Product { get; set; }
}
```

**EF Core's Assumption:**
```
OrderItem.ProductSku → Product.Id  (WRONG!)
```

### Generated SQL (INCORRECT!)
```sql
SELECT oi.*, p.*
FROM order_items oi
LEFT JOIN products p ON oi.product_sku = p.id  -- ❌ Comparing string to int!
WHERE oi.order_id = 1001
```

**Result:** 
- Query returns 0 records or throws type mismatch error
- Navigation property `OrderItem.Product` is always NULL
- `.Include(oi => oi.Product)` doesn't work

---

## ✅ Solution: Explicit Configuration

### Step 1: Define the Unique Index
```csharp
modelBuilder.Entity<Product>()
    .HasIndex(p => p.Sku)
    .HasDatabaseName("ix_products_sku")
    .IsUnique();  // ⚠️ MUST be unique to use as principal key
```

**Why This is Required:**
- EF Core validates that principal keys are unique
- Prevents data integrity issues
- Matches database constraint

### Step 2: Configure the Relationship
```csharp
modelBuilder.Entity<OrderItem>()
    .HasOne(orderItem => orderItem.Product)              // Each order item has ONE product
    .WithMany(product => product.OrderItems)             // Each product has MANY order items
    .HasForeignKey(orderItem => orderItem.ProductSku)    // FK column on child table
    .HasPrincipalKey(product => product.Sku);            // 🔑 Principal key on parent table
```

### Generated SQL (CORRECT!)
```sql
SELECT oi.*, p.*
FROM order_items oi
LEFT JOIN products p ON oi.product_sku = p.sku  -- ✅ Correct join!
WHERE oi.order_id = 1001
```

---

## 🔑 Key Terminology

| Term | Definition | Real-World Example |
|------|------------|-------------------|
| **Primary Key** | Main unique identifier of an entity | `Product.Id` (database internal) |
| **Principal Key** | Any unique column used as a reference point for FK | `Product.Sku` (business identifier) |
| **Foreign Key** | Column that references the principal key | `OrderItem.ProductSku` |
| **Alternate Key** | Unique key that's not the primary key | `Sku` (unique but not PK) |
| **Natural Key** | Real-world identifier with business meaning | SKU, ISBN, SSN |
| **Surrogate Key** | Artificial identifier (usually auto-increment) | `Id` column |

---

## 🧩 Configuration Methods Explained

```csharp
HasOne<TRelatedEntity>()        // Defines the "one" side of relationship
WithMany()                       // Defines the "many" side (collection)
HasForeignKey()                  // Specifies FK column on dependent (child) entity
HasPrincipalKey()                // Specifies principal key on principal (parent) entity
IsUnique()                       // Marks index as unique constraint
```

### Relationship Flow
```
Child (Many) → HasOne → Parent (One)
    ↓
Parent (One) → WithMany → Children (Collection)
    ↓
Child → HasForeignKey → Child.ProductSku
    ↓
Parent → HasPrincipalKey → Parent.Sku
```

---

## 📊 Visual Representation

```
Product (Parent)
├── Id (PK) = 1001
├── Sku (Unique) = "LAPTOP-XPS15"  ← Principal Key
├── Name = "Dell XPS 15"
└── Price = 1499.99
        ↑
        │ (Foreign Key Relationship)
        │
OrderItem (Child)
├── Id (PK) = 5001
├── OrderId = 1001
├── ProductSku (FK) = "LAPTOP-XPS15"  ← References Parent.Sku
└── Quantity = 2
```

---

## 🚨 Common Mistakes

### Mistake 1: Forgetting `HasPrincipalKey()`
```csharp
// ❌ WRONG - EF Core will try to use Id instead of Sku
modelBuilder.Entity<OrderItem>()
    .HasOne(oi => oi.Product)
    .WithMany(p => p.OrderItems)
    .HasForeignKey(oi => oi.ProductSku);  // Missing HasPrincipalKey!

// Result: SQL tries to join ProductSku (string) with Id (int) → Error!
```

### Mistake 2: Not Marking Column as Unique
```csharp
// ❌ WRONG - Principal key must be unique
modelBuilder.Entity<Product>()
    .HasIndex(p => p.Sku);  // Missing .IsUnique()!

// Result: EF Core throws exception - principal key must be unique
```

### Mistake 3: Wrong Order of Configuration
```csharp
// ❌ WRONG - Configure on the dependent (child) entity
modelBuilder.Entity<Product>()  // Wrong starting point
    .HasMany(p => p.OrderItems)
    .WithOne(oi => oi.Product)
    .HasForeignKey(oi => oi.ProductSku)
    .HasPrincipalKey(p => p.Sku);

// ✅ CORRECT - Configure on the dependent entity
modelBuilder.Entity<OrderItem>()  // Correct starting point
    .HasOne(oi => oi.Product)
    .WithMany(p => p.OrderItems)
    .HasForeignKey(oi => oi.ProductSku)
    .HasPrincipalKey(p => p.Sku);
```

---

## 🎓 When to Use This Pattern

Use `HasPrincipalKey()` when:

| Scenario | Example |
|----------|---------|
| ✅ Legacy database schemas | Can't modify existing database structure |
| ✅ Natural business keys | ISBN for books, VIN for vehicles, SSN for people |
| ✅ External system integration | Partner APIs use their own identifiers |
| ✅ Multi-tenant systems | TenantId + Code as composite natural key |
| ✅ Performance optimization | Joining on indexed business keys |

---

## 📝 Complete Working Examples

### Example 1: E-Commerce (Products and Order Items)

```csharp
// Entity Classes
public class Product
{
    public int Id { get; set; }
    public string Sku { get; set; }  // Unique business identifier
    public string Name { get; set; }
    public decimal Price { get; set; }
    public ICollection<OrderItem> OrderItems { get; set; }
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public string ProductSku { get; set; }  // Foreign Key to Product.Sku
    public int Quantity { get; set; }
    public Product Product { get; set; }
}

// DbContext Configuration
public class ShoppingDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Define unique index on Sku
        modelBuilder.Entity<Product>()
            .HasIndex(p => p.Sku)
            .HasDatabaseName("ix_products_sku")
            .IsUnique();

        // Configure relationship
        modelBuilder.Entity<OrderItem>()
            .HasOne(oi => oi.Product)
            .WithMany(p => p.OrderItems)
            .HasForeignKey(oi => oi.ProductSku)
            .HasPrincipalKey(p => p.Sku);
    }
}

// Query Examples
public class OrderService
{
    private readonly ShoppingDbContext _context;

    // Query 1: Get order with product details
    public async Task<List<OrderItem>> GetOrderItemsAsync(int orderId)
    {
        return await _context.OrderItems
            .Include(oi => oi.Product)  // ✅ Now works correctly!
            .Where(oi => oi.OrderId == orderId)
            .ToListAsync();
    }

    // Query 2: Get all orders for a specific product SKU
    public async Task<List<OrderItem>> GetOrdersBySkuAsync(string sku)
    {
        return await _context.OrderItems
            .Where(oi => oi.ProductSku == sku)  // ✅ Proper join via SKU
            .Include(oi => oi.Product)
            .ToListAsync();
    }

    // Query 3: Get sales report
    public async Task<decimal> GetTotalSalesBySkuAsync(string sku)
    {
        return await _context.OrderItems
            .Where(oi => oi.ProductSku == sku)
            .Include(oi => oi.Product)  // ✅ Navigation works!
            .SumAsync(oi => oi.Quantity * oi.Product.Price);
    }
}
```

### Example 2: Library System (Books and Loans)

```csharp
// Entity Classes
public class Book
{
    public int Id { get; set; }
    public string Isbn { get; set; }  // International Standard Book Number (unique)
    public string Title { get; set; }
    public string Author { get; set; }
    public ICollection<BookLoan> BookLoans { get; set; }
}

public class BookLoan
{
    public int Id { get; set; }
    public string BookIsbn { get; set; }  // Foreign Key to Book.Isbn
    public int MemberId { get; set; }
    public DateTime LoanDate { get; set; }
    public DateTime? ReturnDate { get; set; }
    public Book Book { get; set; }
}

// DbContext Configuration
public class LibraryDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Define unique index on ISBN
        modelBuilder.Entity<Book>()
            .HasIndex(b => b.Isbn)
            .HasDatabaseName("ix_books_isbn")
            .IsUnique();

        // Configure relationship
        modelBuilder.Entity<BookLoan>()
            .HasOne(bl => bl.Book)
            .WithMany(b => b.BookLoans)
            .HasForeignKey(bl => bl.BookIsbn)
            .HasPrincipalKey(b => b.Isbn);
    }
}

// Query Examples
public class LibraryService
{
    private readonly LibraryDbContext _context;

    // Get all active loans with book details
    public async Task<List<BookLoan>> GetActiveLoansAsync()
    {
        return await _context.BookLoans
            .Include(bl => bl.Book)  // ✅ Joins via ISBN
            .Where(bl => bl.ReturnDate == null)
            .ToListAsync();
    }

    // Check if book is available (by ISBN)
    public async Task<bool> IsBookAvailableAsync(string isbn)
    {
        var activeLoans = await _context.BookLoans
            .Where(bl => bl.BookIsbn == isbn && bl.ReturnDate == null)
            .CountAsync();

        return activeLoans == 0;
    }
}
```

### Example 3: Employee-Department (Composite Scenario)

```csharp
// Entity Classes
public class Department
{
    public int Id { get; set; }
    public string DepartmentCode { get; set; }  // Unique code like "HR", "IT", "FIN"
    public string Name { get; set; }
    public ICollection<Employee> Employees { get; set; }
}

public class Employee
{
    public int Id { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string DepartmentCode { get; set; }  // FK to Department.DepartmentCode
    public Department Department { get; set; }
}

// DbContext Configuration
public class CompanyDbContext : DbContext
{
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Define unique index on DepartmentCode
        modelBuilder.Entity<Department>()
            .HasIndex(d => d.DepartmentCode)
            .HasDatabaseName("ix_departments_code")
            .IsUnique();

        // Configure relationship
        modelBuilder.Entity<Employee>()
            .HasOne(e => e.Department)
            .WithMany(d => d.Employees)
            .HasForeignKey(e => e.DepartmentCode)
            .HasPrincipalKey(d => d.DepartmentCode);
    }
}

// Query Examples
public class HRService
{
    private readonly CompanyDbContext _context;

    // Get all employees in a department
    public async Task<List<Employee>> GetDepartmentEmployeesAsync(string deptCode)
    {
        return await _context.Employees
            .Include(e => e.Department)  // ✅ Proper join
            .Where(e => e.DepartmentCode == deptCode)
            .OrderBy(e => e.LastName)
            .ToListAsync();
    }

    // Get department statistics
    public async Task<object> GetDepartmentStatsAsync(string deptCode)
    {
        var dept = await _context.Departments
            .Include(d => d.Employees)
            .FirstOrDefaultAsync(d => d.DepartmentCode == deptCode);

        return new
        {
            DepartmentName = dept.Name,
            EmployeeCount = dept.Employees.Count,
            Employees = dept.Employees.Select(e => $"{e.FirstName} {e.LastName}")
        };
    }
}
```

---

## 🔍 Debugging Tips

### 1. Enable SQL Logging
```csharp
// In Program.cs or Startup.cs
services.AddDbContext<YourDbContext>(options =>
{
    options.UseNpgsql(connectionString)
           .LogTo(Console.WriteLine, LogLevel.Information)  // See generated SQL
           .EnableSensitiveDataLogging();  // See parameter values
});
```

### 2. Check Generated Migrations
```bash
dotnet ef migrations add TestRelationship
```

Look for:
```csharp
migrationBuilder.CreateIndex(
    name: "ix_products_sku",
    table: "products",
    column: "sku",
    unique: true);  // ✅ Should be present

migrationBuilder.CreateIndex(
    name: "ix_order_items_product_sku",
    table: "order_items",
    column: "product_sku");  // ✅ FK index
```

### 3. Verify Database Constraints
```sql
-- PostgreSQL
SELECT conname, contype, conkey
FROM pg_constraint
WHERE conrelid = 'products'::regclass;

-- SQL Server
SELECT name, type_desc
FROM sys.indexes
WHERE object_id = OBJECT_ID('Products');
```

---

## 💡 Pro Tips

### Tip 1: Naming Conventions
```csharp
// Good: Clear business meaning
public string ProductSku { get; set; }
public string BookIsbn { get; set; }
public string DepartmentCode { get; set; }

// Bad: Generic names
public string Code { get; set; }
public string Key { get; set; }
```

### Tip 2: Add Validation
```csharp
public class Product
{
    [Required]
    [StringLength(50)]
    [RegularExpression(@"^[A-Z0-9-]+$")]
    public string Sku { get; set; }
}
```

### Tip 3: Consider Performance
```csharp
// Index the foreign key for better query performance
modelBuilder.Entity<OrderItem>()
    .HasIndex(oi => oi.ProductSku)
    .HasDatabaseName("ix_order_items_product_sku");
```

### Tip 4: Document Your Decisions
```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    // Business Reason: External systems (warehouse, suppliers) reference products by SKU
    // Technical Reason: SKU is the natural key, immutable, and already indexed
    modelBuilder.Entity<OrderItem>()
        .HasOne(oi => oi.Product)
        .WithMany(p => p.OrderItems)
        .HasForeignKey(oi => oi.ProductSku)
        .HasPrincipalKey(p => p.Sku);
}
```

---

## 📊 Performance Considerations

### Index Strategy
```csharp
// Parent table: Index on principal key
modelBuilder.Entity<Product>()
    .HasIndex(p => p.Sku)
    .IsUnique();  // ✅ Essential for principal key

// Child table: Index on foreign key
modelBuilder.Entity<OrderItem>()
    .HasIndex(oi => oi.ProductSku);  // ✅ Improves join performance
```

### Query Performance
```csharp
// ❌ Avoid: Multiple database round trips
var orderItems = await _context.OrderItems.ToListAsync();
foreach (var item in orderItems)
{
    var product = await _context.Products
        .FirstAsync(p => p.Sku == item.ProductSku);  // N+1 problem!
}

// ✅ Better: Single query with Include
var orderItems = await _context.OrderItems
    .Include(oi => oi.Product)
    .ToListAsync();
```

---

## 🔗 Related Concepts

### Composite Keys
```csharp
// When principal key is combination of columns
modelBuilder.Entity<OrderItem>()
    .HasOne(oi => oi.Product)
    .WithMany(p => p.OrderItems)
    .HasForeignKey(oi => new { oi.TenantId, oi.ProductSku })
    .HasPrincipalKey(p => new { p.TenantId, p.Sku });
```

### Shadow Properties
```csharp
// EF Core manages property not in entity class
modelBuilder.Entity<OrderItem>()
    .Property<DateTime>("CreatedAt")
    .HasDefaultValueSql("GETDATE()");
```

### Owned Entities
```csharp
// For value objects
modelBuilder.Entity<Order>()
    .OwnsOne(o => o.ShippingAddress);
```

---

## ✅ Checklist: Before Going to Production

- [ ] Unique index created on principal key column
- [ ] Relationship configured with `HasPrincipalKey()`
- [ ] Foreign key index created for performance
- [ ] Migration generated and reviewed
- [ ] SQL logging enabled to verify correct joins
- [ ] Unit tests written for navigation properties
- [ ] Integration tests verify data retrieval
- [ ] Database constraints match EF Core configuration
- [ ] Performance tested with realistic data volumes
- [ ] Code reviewed and documented

---

## 🎯 Summary

**Key Takeaway:** When your foreign key references a non-primary key column:

1. ✅ Mark the column as **unique** using `HasIndex().IsUnique()`
2. ✅ Configure relationship with `HasPrincipalKey()` on the parent entity
3. ✅ Always configure from the **dependent (child)** entity
4. ✅ Test with SQL logging enabled to verify correct joins

**Without proper configuration:**
- ❌ Queries return 0 records
- ❌ Navigation properties are NULL
- ❌ `.Include()` doesn't work
- ❌ Wrong SQL joins generated

**With proper configuration:**
- ✅ Queries return correct data
- ✅ Navigation properties populated
- ✅ `.Include()` works perfectly
- ✅ Correct SQL joins generated

---

**Date Created:** January 9, 2026  
**Technology:** EF Core 8.0, .NET 8, SQL Server / PostgreSQL  
**Document Version:** 2.0  
**Author:** AI Learning Assistant  

---

*Keep learning! 🚀*
