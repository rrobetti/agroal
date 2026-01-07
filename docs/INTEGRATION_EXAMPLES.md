# Practical Integration Examples for Agroal XA Pooling

This document provides complete, working examples for integrating Agroal XA connection pooling into your application.

## Table of Contents

1. [Maven/Gradle Setup](#mavengradle-setup)
2. [Example 1: Standalone XA Pool](#example-1-standalone-xa-pool)
3. [Example 2: Database Proxy with Dual Pools](#example-2-database-proxy-with-dual-pools)
4. [Example 3: Integration with Narayana](#example-3-integration-with-narayana)
5. [Example 4: Manual XA Transaction Management](#example-4-manual-xa-transaction-management)
6. [Example 5: Spring Boot Integration](#example-5-spring-boot-integration)
7. [Configuration Examples](#configuration-examples)
8. [Testing Your Integration](#testing-your-integration)

---

## Maven/Gradle Setup

### Maven (pom.xml)

```xml
<properties>
    <agroal.version>3.0</agroal.version>
    <narayana.version>7.1.0.Final</narayana.version>
</properties>

<dependencies>
    <!-- Core Agroal dependencies -->
    <dependency>
        <groupId>io.agroal</groupId>
        <artifactId>agroal-api</artifactId>
        <version>${agroal.version}</version>
    </dependency>
    <dependency>
        <groupId>io.agroal</groupId>
        <artifactId>agroal-pool</artifactId>
        <version>${agroal.version}</version>
    </dependency>
    
    <!-- Optional: Narayana integration -->
    <dependency>
        <groupId>io.agroal</groupId>
        <artifactId>agroal-narayana</artifactId>
        <version>${agroal.version}</version>
    </dependency>
    
    <!-- Your JDBC driver with XA support -->
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <version>42.7.1</version>
    </dependency>
</dependencies>
```

### Gradle (build.gradle)

```gradle
dependencies {
    implementation 'io.agroal:agroal-api:3.0'
    implementation 'io.agroal:agroal-pool:3.0'
    
    // Optional: Narayana integration
    implementation 'io.agroal:agroal-narayana:3.0'
    
    // Your JDBC driver
    implementation 'org.postgresql:postgresql:42.7.1'
}
```

---

## Example 1: Standalone XA Pool

### Simple XA Pool Configuration

```java
package com.example.pool;

import io.agroal.api.AgroalDataSource;
import io.agroal.api.AgroalDataSourceListener;
import io.agroal.api.configuration.AgroalConnectionPoolConfiguration.ConnectionValidator;
import io.agroal.api.configuration.supplier.AgroalDataSourceConfigurationSupplier;
import io.agroal.api.security.NamePrincipal;
import io.agroal.api.security.SimplePassword;
import org.postgresql.xa.PGXADataSource;

import java.sql.Connection;
import java.sql.SQLException;
import java.time.Duration;

public class SimpleXAPoolExample {
    
    private final AgroalDataSource dataSource;
    
    public SimpleXAPoolExample() throws SQLException {
        AgroalDataSourceConfigurationSupplier configuration = 
            new AgroalDataSourceConfigurationSupplier()
                .metricsEnabled(true)
                .connectionPoolConfiguration(cp -> cp
                    // Pool sizing
                    .initialSize(10)
                    .minSize(10)
                    .maxSize(50)
                    
                    // Timeouts
                    .acquisitionTimeout(Duration.ofSeconds(30))
                    .leakTimeout(Duration.ofMinutes(5))
                    .validationTimeout(Duration.ofMinutes(1))
                    .reapTimeout(Duration.ofMinutes(10))
                    .idleValidationTimeout(Duration.ofMinutes(5))
                    
                    // Validation
                    .validateOnBorrow(false)  // Validate on idle instead
                    .connectionValidator(ConnectionValidator.defaultValidator())
                    
                    // Connection factory - XA DataSource
                    .connectionFactoryConfiguration(cf -> cf
                        .connectionProviderClass(PGXADataSource.class)
                        .jdbcUrl("jdbc:postgresql://localhost:5432/mydb")
                        .principal(new NamePrincipal("myuser"))
                        .credential(new SimplePassword("mypassword"))
                        .autoCommit(false)  // Important for XA
                        .jdbcTransactionIsolation(
                            AgroalConnectionFactoryConfiguration.TransactionIsolation.READ_COMMITTED
                        )
                    )
                );
        
        this.dataSource = AgroalDataSource.from(configuration);
    }
    
    public Connection getConnection() throws SQLException {
        return dataSource.getConnection();
    }
    
    public void close() {
        if (dataSource != null) {
            dataSource.close();
        }
    }
    
    public void printMetrics() {
        var metrics = dataSource.getMetrics();
        System.out.println("Active connections: " + metrics.activeCount());
        System.out.println("Available connections: " + metrics.availableCount());
        System.out.println("Awaiting threads: " + metrics.awaitingCount());
        System.out.println("Max used connections: " + metrics.maxUsedCount());
    }
    
    public static void main(String[] args) throws SQLException {
        SimpleXAPoolExample pool = new SimpleXAPoolExample();
        
        try (Connection conn = pool.getConnection()) {
            var stmt = conn.createStatement();
            var rs = stmt.executeQuery("SELECT version()");
            if (rs.next()) {
                System.out.println("Database: " + rs.getString(1));
            }
        }
        
        pool.printMetrics();
        pool.close();
    }
}
```

---

## Example 2: Database Proxy with Dual Pools

### Hybrid Approach (HikariCP + Agroal)

```java
package com.example.proxy;

import com.zaxxer.hikari.HikariConfig;
import com.zaxxer.hikari.HikariDataSource;
import io.agroal.api.AgroalDataSource;
import io.agroal.api.configuration.supplier.AgroalDataSourceConfigurationSupplier;
import io.agroal.api.security.NamePrincipal;
import io.agroal.api.security.SimplePassword;
import org.postgresql.xa.PGXADataSource;

import javax.sql.XAConnection;
import java.sql.Connection;
import java.sql.SQLException;
import java.time.Duration;

/**
 * Database proxy that uses HikariCP for non-XA connections
 * and Agroal for XA connections
 */
public class DatabaseProxy {
    
    private final HikariDataSource nonXaPool;
    private final AgroalDataSource xaPool;
    
    public DatabaseProxy(String jdbcUrl, String username, String password) throws SQLException {
        this.nonXaPool = createNonXAPool(jdbcUrl, username, password);
        this.xaPool = createXAPool(jdbcUrl, username, password);
    }
    
    private HikariDataSource createNonXAPool(String jdbcUrl, String username, String password) {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(jdbcUrl);
        config.setUsername(username);
        config.setPassword(password);
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        
        return new HikariDataSource(config);
    }
    
    private AgroalDataSource createXAPool(String jdbcUrl, String username, String password) 
            throws SQLException {
        AgroalDataSourceConfigurationSupplier config = 
            new AgroalDataSourceConfigurationSupplier()
                .metricsEnabled(true)
                .connectionPoolConfiguration(cp -> cp
                    .initialSize(5)
                    .minSize(5)
                    .maxSize(20)
                    .acquisitionTimeout(Duration.ofSeconds(30))
                    .leakTimeout(Duration.ofMinutes(5))
                    .validationTimeout(Duration.ofMinutes(1))
                    .reapTimeout(Duration.ofMinutes(10))
                    .idleValidationTimeout(Duration.ofMinutes(5))
                    .connectionFactoryConfiguration(cf -> cf
                        .connectionProviderClass(PGXADataSource.class)
                        .jdbcUrl(jdbcUrl)
                        .principal(new NamePrincipal(username))
                        .credential(new SimplePassword(password))
                        .autoCommit(false)
                    )
                );
        
        return AgroalDataSource.from(config);
    }
    
    /**
     * Get a connection based on whether XA is required
     */
    public Connection getConnection(boolean requiresXA) throws SQLException {
        if (requiresXA) {
            return xaPool.getConnection();
        } else {
            return nonXaPool.getConnection();
        }
    }
    
    /**
     * Get a regular non-XA connection
     */
    public Connection getConnection() throws SQLException {
        return nonXaPool.getConnection();
    }
    
    /**
     * Get an XA connection for direct XA operations
     */
    public XAConnection getXAConnection() throws SQLException {
        // Access the underlying pool for XA connection
        // This requires casting to implementation class
        io.agroal.pool.DataSource ds = (io.agroal.pool.DataSource) xaPool;
        io.agroal.pool.Pool pool = ds.getConnectionPool();
        return pool.getRecoveryConnection();
    }
    
    /**
     * Get metrics from both pools
     */
    public PoolMetrics getMetrics() {
        return new PoolMetrics(
            nonXaPool.getHikariPoolMXBean().getActiveConnections(),
            nonXaPool.getHikariPoolMXBean().getIdleConnections(),
            xaPool.getMetrics().activeCount(),
            xaPool.getMetrics().availableCount()
        );
    }
    
    public void close() {
        if (nonXaPool != null) {
            nonXaPool.close();
        }
        if (xaPool != null) {
            xaPool.close();
        }
    }
    
    public static class PoolMetrics {
        public final long nonXaActive;
        public final long nonXaIdle;
        public final long xaActive;
        public final long xaAvailable;
        
        public PoolMetrics(long nonXaActive, long nonXaIdle, long xaActive, long xaAvailable) {
            this.nonXaActive = nonXaActive;
            this.nonXaIdle = nonXaIdle;
            this.xaActive = xaActive;
            this.xaAvailable = xaAvailable;
        }
        
        @Override
        public String toString() {
            return String.format(
                "Non-XA Pool: %d active, %d idle | XA Pool: %d active, %d available",
                nonXaActive, nonXaIdle, xaActive, xaAvailable
            );
        }
    }
    
    // Example usage
    public static void main(String[] args) throws SQLException {
        DatabaseProxy proxy = new DatabaseProxy(
            "jdbc:postgresql://localhost:5432/mydb",
            "myuser",
            "mypassword"
        );
        
        try {
            // Regular non-XA connection
            try (Connection conn = proxy.getConnection()) {
                System.out.println("Got non-XA connection");
                // Use connection
            }
            
            // XA connection when needed
            try (Connection conn = proxy.getConnection(true)) {
                System.out.println("Got XA connection");
                // Use in distributed transaction
            }
            
            // Print metrics
            System.out.println(proxy.getMetrics());
            
        } finally {
            proxy.close();
        }
    }
}
```

---

## Example 3: Integration with Narayana

### Complete Narayana + Agroal Setup

```java
package com.example.narayana;

import com.arjuna.ats.jta.TransactionManager;
import io.agroal.api.AgroalDataSource;
import io.agroal.api.configuration.supplier.AgroalDataSourceConfigurationSupplier;
import io.agroal.api.security.NamePrincipal;
import io.agroal.api.security.SimplePassword;
import io.agroal.narayana.NarayanaTransactionIntegration;
import org.postgresql.xa.PGXADataSource;

import javax.transaction.HeuristicMixedException;
import javax.transaction.HeuristicRollbackException;
import javax.transaction.NotSupportedException;
import javax.transaction.RollbackException;
import javax.transaction.SystemException;
import java.sql.Connection;
import java.sql.SQLException;
import java.time.Duration;

public class NarayanaXAExample {
    
    private final javax.transaction.TransactionManager tm;
    private final AgroalDataSource dataSource1;
    private final AgroalDataSource dataSource2;
    
    public NarayanaXAExample() throws SQLException {
        // Get Narayana transaction manager
        this.tm = TransactionManager.transactionManager();
        
        // Create XA data sources with Narayana integration
        this.dataSource1 = createDataSource(
            "jdbc:postgresql://localhost:5432/db1",
            "user1",
            "pass1"
        );
        
        this.dataSource2 = createDataSource(
            "jdbc:postgresql://localhost:5432/db2",
            "user2",
            "pass2"
        );
    }
    
    private AgroalDataSource createDataSource(String url, String user, String pass) 
            throws SQLException {
        AgroalDataSourceConfigurationSupplier config = 
            new AgroalDataSourceConfigurationSupplier()
                .connectionPoolConfiguration(cp -> cp
                    .maxSize(20)
                    .acquisitionTimeout(Duration.ofSeconds(30))
                    // Integrate with Narayana
                    .transactionIntegration(new NarayanaTransactionIntegration(tm))
                    .connectionFactoryConfiguration(cf -> cf
                        .connectionProviderClass(PGXADataSource.class)
                        .jdbcUrl(url)
                        .principal(new NamePrincipal(user))
                        .credential(new SimplePassword(pass))
                        .autoCommit(false)
                    )
                );
        
        return AgroalDataSource.from(config);
    }
    
    /**
     * Perform a distributed transaction across two databases
     */
    public void performDistributedTransaction() 
            throws SQLException, SystemException, NotSupportedException, 
                   HeuristicRollbackException, HeuristicMixedException, RollbackException {
        
        tm.begin();
        
        try {
            // Connection to first database
            try (Connection conn1 = dataSource1.getConnection()) {
                var stmt1 = conn1.createStatement();
                stmt1.executeUpdate(
                    "UPDATE accounts SET balance = balance - 100 WHERE id = 1"
                );
            }
            
            // Connection to second database
            try (Connection conn2 = dataSource2.getConnection()) {
                var stmt2 = conn2.createStatement();
                stmt2.executeUpdate(
                    "UPDATE accounts SET balance = balance + 100 WHERE id = 2"
                );
            }
            
            // Both updates succeed - commit the distributed transaction
            tm.commit();
            System.out.println("Distributed transaction committed successfully");
            
        } catch (Exception e) {
            // Something failed - rollback both databases
            tm.rollback();
            System.err.println("Distributed transaction rolled back: " + e.getMessage());
            throw e;
        }
    }
    
    /**
     * Demonstrate connection reuse within a transaction
     */
    public void demonstrateConnectionReuse() 
            throws SQLException, SystemException, NotSupportedException, 
                   HeuristicRollbackException, HeuristicMixedException, RollbackException {
        
        tm.begin();
        
        try {
            // First call to getConnection()
            try (Connection conn1 = dataSource1.getConnection()) {
                System.out.println("First connection: " + conn1);
                conn1.createStatement().executeUpdate("UPDATE table1 SET col1 = 'val1'");
            }
            
            // Second call to getConnection() - REUSES the same underlying connection
            try (Connection conn2 = dataSource1.getConnection()) {
                System.out.println("Second connection: " + conn2);
                conn2.createStatement().executeUpdate("UPDATE table2 SET col2 = 'val2'");
            }
            
            // Both operations in same transaction
            tm.commit();
            
        } catch (Exception e) {
            tm.rollback();
            throw e;
        }
    }
    
    public void close() {
        if (dataSource1 != null) {
            dataSource1.close();
        }
        if (dataSource2 != null) {
            dataSource2.close();
        }
    }
    
    public static void main(String[] args) throws Exception {
        NarayanaXAExample example = new NarayanaXAExample();
        
        try {
            example.performDistributedTransaction();
            example.demonstrateConnectionReuse();
        } finally {
            example.close();
        }
    }
}
```

---

## Example 4: Manual XA Transaction Management

### Low-Level XA Operations (Without Transaction Manager)

```java
package com.example.manual;

import io.agroal.api.AgroalDataSource;
import io.agroal.api.configuration.supplier.AgroalDataSourceConfigurationSupplier;
import io.agroal.api.security.NamePrincipal;
import io.agroal.api.security.SimplePassword;
import org.postgresql.xa.PGXADataSource;

import javax.sql.XAConnection;
import javax.transaction.xa.XAException;
import javax.transaction.xa.XAResource;
import javax.transaction.xa.Xid;
import java.sql.Connection;
import java.sql.SQLException;
import java.time.Duration;
import java.util.UUID;

public class ManualXAExample {
    
    private final AgroalDataSource dataSource;
    
    public ManualXAExample() throws SQLException {
        AgroalDataSourceConfigurationSupplier config = 
            new AgroalDataSourceConfigurationSupplier()
                .connectionPoolConfiguration(cp -> cp
                    .maxSize(10)
                    .acquisitionTimeout(Duration.ofSeconds(30))
                    .connectionFactoryConfiguration(cf -> cf
                        .connectionProviderClass(PGXADataSource.class)
                        .jdbcUrl("jdbc:postgresql://localhost:5432/mydb")
                        .principal(new NamePrincipal("myuser"))
                        .credential(new SimplePassword("mypassword"))
                        .autoCommit(false)
                    )
                );
        
        this.dataSource = AgroalDataSource.from(config);
    }
    
    /**
     * Create a transaction ID (XID)
     */
    private Xid createXid() {
        return new Xid() {
            private final byte[] gtrid = UUID.randomUUID().toString().getBytes();
            private final byte[] bqual = UUID.randomUUID().toString().getBytes();
            
            @Override
            public int getFormatId() {
                return 1;
            }
            
            @Override
            public byte[] getGlobalTransactionId() {
                return gtrid;
            }
            
            @Override
            public byte[] getBranchQualifier() {
                return bqual;
            }
        };
    }
    
    /**
     * Perform manual XA transaction
     */
    public void performManualXATransaction() throws SQLException, XAException {
        // Get XA connection from pool
        io.agroal.pool.DataSource ds = (io.agroal.pool.DataSource) dataSource;
        io.agroal.pool.Pool pool = ds.getConnectionPool();
        XAConnection xaConn = pool.getRecoveryConnection();
        
        try {
            XAResource xaRes = xaConn.getXAResource();
            Connection conn = xaConn.getConnection();
            
            // Create XID
            Xid xid = createXid();
            
            // Start XA transaction
            xaRes.start(xid, XAResource.TMNOFLAGS);
            System.out.println("XA transaction started");
            
            // Perform database operations
            var stmt = conn.createStatement();
            int updated = stmt.executeUpdate(
                "UPDATE accounts SET balance = balance - 100 WHERE id = 1"
            );
            System.out.println("Updated " + updated + " rows");
            
            // End the transaction
            xaRes.end(xid, XAResource.TMSUCCESS);
            System.out.println("XA transaction ended");
            
            // Prepare phase
            int prepareResult = xaRes.prepare(xid);
            System.out.println("Prepare result: " + prepareResult);
            
            // Commit phase
            if (prepareResult == XAResource.XA_OK) {
                xaRes.commit(xid, false);
                System.out.println("XA transaction committed");
            } else if (prepareResult == XAResource.XA_RDONLY) {
                // Read-only transaction, no commit needed
                System.out.println("XA transaction was read-only");
            }
            
        } catch (Exception e) {
            System.err.println("Error in XA transaction: " + e.getMessage());
            throw e;
        } finally {
            // Return connection to pool
            xaConn.close();
        }
    }
    
    /**
     * Perform XA transaction with rollback
     */
    public void performXATransactionWithRollback() throws SQLException, XAException {
        io.agroal.pool.DataSource ds = (io.agroal.pool.DataSource) dataSource;
        io.agroal.pool.Pool pool = ds.getConnectionPool();
        XAConnection xaConn = pool.getRecoveryConnection();
        
        try {
            XAResource xaRes = xaConn.getXAResource();
            Connection conn = xaConn.getConnection();
            Xid xid = createXid();
            
            // Start transaction
            xaRes.start(xid, XAResource.TMNOFLAGS);
            
            try {
                var stmt = conn.createStatement();
                stmt.executeUpdate("UPDATE accounts SET balance = balance - 1000 WHERE id = 1");
                
                // Simulate error
                throw new RuntimeException("Business logic error!");
                
            } catch (Exception e) {
                // Mark transaction for rollback
                xaRes.end(xid, XAResource.TMFAIL);
                xaRes.rollback(xid);
                System.out.println("XA transaction rolled back");
                throw e;
            }
            
        } finally {
            xaConn.close();
        }
    }
    
    public void close() {
        if (dataSource != null) {
            dataSource.close();
        }
    }
    
    public static void main(String[] args) throws SQLException, XAException {
        ManualXAExample example = new ManualXAExample();
        
        try {
            example.performManualXATransaction();
            
            try {
                example.performXATransactionWithRollback();
            } catch (RuntimeException e) {
                System.out.println("Expected error: " + e.getMessage());
            }
            
        } finally {
            example.close();
        }
    }
}
```

---

## Example 5: Spring Boot Integration

### Spring Boot Configuration

```java
package com.example.springboot;

import io.agroal.api.AgroalDataSource;
import io.agroal.api.configuration.supplier.AgroalDataSourceConfigurationSupplier;
import io.agroal.api.security.NamePrincipal;
import io.agroal.api.security.SimplePassword;
import org.postgresql.xa.PGXADataSource;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.transaction.annotation.EnableTransactionManagement;

import java.sql.SQLException;
import java.time.Duration;

@Configuration
@EnableTransactionManagement
public class DataSourceConfig {
    
    @Value("${spring.datasource.url}")
    private String jdbcUrl;
    
    @Value("${spring.datasource.username}")
    private String username;
    
    @Value("${spring.datasource.password}")
    private String password;
    
    @Bean
    public AgroalDataSource agroalDataSource() throws SQLException {
        AgroalDataSourceConfigurationSupplier config = 
            new AgroalDataSourceConfigurationSupplier()
                .metricsEnabled(true)
                .connectionPoolConfiguration(cp -> cp
                    .initialSize(10)
                    .minSize(10)
                    .maxSize(50)
                    .acquisitionTimeout(Duration.ofSeconds(30))
                    .leakTimeout(Duration.ofMinutes(5))
                    .validationTimeout(Duration.ofMinutes(1))
                    .reapTimeout(Duration.ofMinutes(10))
                    .idleValidationTimeout(Duration.ofMinutes(5))
                    .connectionFactoryConfiguration(cf -> cf
                        .connectionProviderClass(PGXADataSource.class)
                        .jdbcUrl(jdbcUrl)
                        .principal(new NamePrincipal(username))
                        .credential(new SimplePassword(password))
                        .autoCommit(false)
                    )
                );
        
        return AgroalDataSource.from(config);
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate(AgroalDataSource dataSource) {
        return new JdbcTemplate(dataSource);
    }
}
```

### application.properties

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=myuser
spring.datasource.password=mypassword
```

---

## Configuration Examples

### Minimal Configuration

```java
AgroalDataSourceConfigurationSupplier config = 
    new AgroalDataSourceConfigurationSupplier()
        .connectionPoolConfiguration(cp -> cp
            .maxSize(20)
            .connectionFactoryConfiguration(cf -> cf
                .connectionProviderClass(PGXADataSource.class)
                .jdbcUrl("jdbc:postgresql://localhost/db")
            )
        );
```

### Production-Ready Configuration

```java
AgroalDataSourceConfigurationSupplier config = 
    new AgroalDataSourceConfigurationSupplier()
        .metricsEnabled(true)
        .connectionPoolConfiguration(cp -> cp
            // Sizing
            .initialSize(10)
            .minSize(10)
            .maxSize(50)
            .maxLifetime(Duration.ofMinutes(30))
            
            // Timeouts
            .acquisitionTimeout(Duration.ofSeconds(30))
            .leakTimeout(Duration.ofMinutes(5))
            .validationTimeout(Duration.ofMinutes(1))
            .reapTimeout(Duration.ofMinutes(10))
            .idleValidationTimeout(Duration.ofMinutes(5))
            
            // Validation
            .validateOnBorrow(false)
            .enhancedLeakReport(true)
            
            // Connection factory
            .connectionFactoryConfiguration(cf -> cf
                .connectionProviderClass(PGXADataSource.class)
                .jdbcUrl("jdbc:postgresql://localhost/db")
                .principal(new NamePrincipal("user"))
                .credential(new SimplePassword("pass"))
                .autoCommit(false)
                .loginTimeout(Duration.ofSeconds(10))
                .jdbcTransactionIsolation(TransactionIsolation.READ_COMMITTED)
                .initialSql("SET application_name = 'MyApp'")
            )
        );
```

---

## Testing Your Integration

### Unit Test Example

```java
package com.example.test;

import io.agroal.api.AgroalDataSource;
import io.agroal.api.configuration.supplier.AgroalDataSourceConfigurationSupplier;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.postgresql.xa.PGXADataSource;

import java.sql.Connection;
import java.sql.SQLException;
import java.time.Duration;

import static org.junit.jupiter.api.Assertions.*;

class AgroalXAPoolTest {
    
    private AgroalDataSource dataSource;
    
    @BeforeEach
    void setUp() throws SQLException {
        AgroalDataSourceConfigurationSupplier config = 
            new AgroalDataSourceConfigurationSupplier()
                .metricsEnabled(true)
                .connectionPoolConfiguration(cp -> cp
                    .initialSize(2)
                    .minSize(2)
                    .maxSize(10)
                    .acquisitionTimeout(Duration.ofSeconds(5))
                    .connectionFactoryConfiguration(cf -> cf
                        .connectionProviderClass(PGXADataSource.class)
                        .jdbcUrl("jdbc:postgresql://localhost/testdb")
                    )
                );
        
        dataSource = AgroalDataSource.from(config);
    }
    
    @AfterEach
    void tearDown() {
        if (dataSource != null) {
            dataSource.close();
        }
    }
    
    @Test
    void testGetConnection() throws SQLException {
        try (Connection conn = dataSource.getConnection()) {
            assertNotNull(conn);
            assertFalse(conn.isClosed());
        }
    }
    
    @Test
    void testConnectionReturnsToPool() throws SQLException {
        // Get and close connection
        try (Connection conn = dataSource.getConnection()) {
            // Use connection
        }
        
        // Pool should have all connections available
        var metrics = dataSource.getMetrics();
        assertEquals(0, metrics.activeCount());
        assertEquals(2, metrics.availableCount());  // minSize = 2
    }
    
    @Test
    void testMaxPoolSize() throws SQLException {
        Connection[] connections = new Connection[10];
        
        // Acquire max connections
        for (int i = 0; i < 10; i++) {
            connections[i] = dataSource.getConnection();
        }
        
        // Verify all active
        var metrics = dataSource.getMetrics();
        assertEquals(10, metrics.activeCount());
        
        // Clean up
        for (Connection conn : connections) {
            conn.close();
        }
    }
}
```

---

**These examples provide practical, working code for integrating Agroal XA connection pooling into various types of applications.**

**Next Steps:**
1. Choose the example that fits your architecture
2. Adjust configurations for your specific database
3. Test thoroughly with your workload
4. Monitor metrics in production

**For more details, see:** `XA_POOLING_ANALYSIS.md` and `ARCHITECTURE_DIAGRAMS.md`
