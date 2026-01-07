# Agroal XA Connection Pooling - Quick Reference

## TL;DR

✅ **YES, you can use Agroal for XA connection pooling in isolation**

Just add the dependencies and configure it:

```xml
<dependency>
    <groupId>io.agroal</groupId>
    <artifactId>agroal-pool</artifactId>
    <version>3.0</version>
</dependency>
```

## Quick Start

### Basic XA Pool Setup

```java
import io.agroal.api.AgroalDataSource;
import io.agroal.api.configuration.supplier.AgroalDataSourceConfigurationSupplier;

// Configure
AgroalDataSourceConfigurationSupplier config = 
    new AgroalDataSourceConfigurationSupplier()
        .connectionPoolConfiguration(cp -> cp
            .maxSize(20)
            .connectionFactoryConfiguration(cf -> cf
                .connectionProviderClass(MyXADataSource.class)
                .jdbcUrl("jdbc:postgresql://localhost/db")
            )
        );

// Create and use
try (AgroalDataSource dataSource = AgroalDataSource.from(config)) {
    try (Connection conn = dataSource.getConnection()) {
        // Use connection
    }
}
```

## Architecture at a Glance

```
Application
    ↓
DataSource (Entry point)
    ↓
ConnectionPool (Pool management)
    ↓
ConnectionHandler (State management)
    ↓
XAConnection (From XADataSource)
    ↓
XAResource (For transaction manager)
```

## Key Components

| Component | Purpose | Lines |
|-----------|---------|-------|
| `ConnectionPool` | Main pooling logic | ~900 |
| `ConnectionHandler` | Connection wrapper with state | ~430 |
| `ConnectionFactory` | Creates connections | ~320 |
| `DataSource` | Public API | ~140 |

**Total Core**: ~4,300 lines (including config and utilities)

## For Your Database Proxy

### Hybrid Approach (Recommended)

```java
public class DatabaseProxy {
    private HikariDataSource nonXaPool;  // Keep for non-XA
    private AgroalDataSource xaPool;      // Add for XA
    
    public Connection getConnection(boolean needsXA) {
        return needsXA ? xaPool.getConnection() 
                      : nonXaPool.getConnection();
    }
    
    public XAConnection getXAConnection() throws SQLException {
        // For direct XA access
        return ((ConnectionPool) xaPool).getRecoveryConnection();
    }
}
```

## Key Differences from HikariCP

| Feature | Agroal | HikariCP |
|---------|--------|----------|
| XA Support | ✅ Native | ❌ None |
| Use Case | XA + Non-XA | Non-XA only |
| TM Integration | ✅ Pluggable | ❌ None |

## Connection States

```
NEW → CHECKED_IN ⇄ CHECKED_OUT → FLUSH → DESTROYED
            ↓           ↓
        VALIDATION  (for health checks)
```

## Important Concepts

### 1. Unified Connection Handling
- XA and non-XA connections handled uniformly
- Non-XA wrapped in `XAConnectionAdaptor` (null XAResource)

### 2. State Management
- Atomic state transitions using `AtomicReferenceFieldUpdater`
- Thread-safe acquisition/return

### 3. Transaction Integration
- Optional `TransactionIntegration` interface
- Default no-op implementation available
- Narayana integration in separate module

## Common Configurations

### Production Settings
```java
.connectionPoolConfiguration(cp -> cp
    .initialSize(10)
    .minSize(10)
    .maxSize(50)
    .acquisitionTimeout(Duration.ofSeconds(30))
    .leakTimeout(Duration.ofMinutes(5))
    .validationTimeout(Duration.ofMinutes(1))
    .reapTimeout(Duration.ofMinutes(10))
    .validateOnBorrow(false)  // Validate on idle instead
    .idleValidationTimeout(Duration.ofMinutes(5))
)
```

### With Transaction Manager
```java
import io.agroal.narayana.NarayanaTransactionIntegration;

.connectionPoolConfiguration(cp -> cp
    .maxSize(20)
    .transactionIntegration(
        new NarayanaTransactionIntegration(transactionManager)
    )
)
```

## Copying Classes (Not Recommended)

If you must extract the code, minimum required:

### Essential Classes
- `io.agroal.pool.ConnectionPool`
- `io.agroal.pool.ConnectionHandler`
- `io.agroal.pool.ConnectionFactory`
- `io.agroal.pool.DataSource`
- `io.agroal.pool.Pool`

### Wrappers
- `io.agroal.pool.wrapper.*` (all)
- `io.agroal.pool.util.XAConnectionAdaptor`

### Utilities  
- `io.agroal.pool.util.PriorityScheduledExecutor`
- `io.agroal.pool.util.StampedCopyOnWriteArrayList`
- `io.agroal.pool.util.PropertyInjector`

### Configuration
- All classes in `io.agroal.api.configuration.*`

**Why not recommended:**
- ~4,300 lines with intricate interactions
- Complex concurrency handling
- Better to use as library

## Decision Matrix

### Use Agroal When:
- ✅ Need XA/distributed transactions
- ✅ Need transaction recovery
- ✅ Integrating with JTA/Jakarta EE
- ✅ Want unified XA + non-XA pool

### Keep HikariCP When:
- ✅ Only need non-XA connections
- ✅ Existing setup works well
- ✅ Simpler requirements

### Use Both When:
- ✅ Have mixed XA and non-XA workloads
- ✅ Want optimal performance for each type
- ✅ Building a database proxy (your case!)

## Resources

- **GitHub**: https://github.com/agroal/agroal
- **Full Analysis**: See `XA_POOLING_ANALYSIS.md` in this directory
- **Examples**: See `agroal-test` module in repository

## Quick Debugging

### Check Pool Status
```java
AgroalDataSourceMetrics metrics = dataSource.getMetrics();
System.out.println("Active: " + metrics.activeCount());
System.out.println("Available: " + metrics.availableCount());
System.out.println("Awaiting: " + metrics.awaitingCount());
```

### Enable Leak Detection
```java
.connectionPoolConfiguration(cp -> cp
    .leakTimeout(Duration.ofMinutes(5))
    .enhancedLeakReport(true)  // Detailed stack traces
)
```

### Validate Connections
```java
// On borrow (slight overhead)
.validateOnBorrow(true)

// Or periodically when idle (preferred)
.idleValidationTimeout(Duration.ofMinutes(5))
```

## Common Pitfalls

❌ **Don't**: Close the XAConnection directly when using with a pool  
✅ **Do**: Close the wrapper Connection returned by getConnection()

❌ **Don't**: Mix pool-managed and non-pooled connections  
✅ **Do**: Always get connections from the DataSource

❌ **Don't**: Try to use XA operations without a transaction manager  
✅ **Do**: Either use manual XA or integrate a TM

❌ **Don't**: Forget to close connections  
✅ **Do**: Use try-with-resources

---

**For detailed explanations, architecture diagrams, and examples, see**: `XA_POOLING_ANALYSIS.md`
