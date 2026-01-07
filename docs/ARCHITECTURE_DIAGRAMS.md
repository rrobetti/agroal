# Agroal Architecture Diagrams

This document contains detailed architectural diagrams for Agroal's XA connection pooling system.

## 1. Overall System Architecture

```mermaid
graph TB
    subgraph "Client Application Layer"
        APP[Application Code]
        JDBC_API[JDBC API Calls]
    end
    
    subgraph "Agroal Public API - agroal-api module"
        IFACE_DS[AgroalDataSource Interface]
        IFACE_CONFIG[Configuration Interfaces]
        IFACE_TI[TransactionIntegration Interface]
    end
    
    subgraph "Agroal Pool Implementation - agroal-pool module"
        DS[DataSource Implementation]
        POOL[ConnectionPool]
        HANDLER[ConnectionHandler]
        FACTORY[ConnectionFactory]
    end
    
    subgraph "Connection Wrappers"
        CONN_WRAP[ConnectionWrapper]
        XA_WRAP[XAConnectionWrapper]
        XARES_WRAP[XAResourceWrapper]
    end
    
    subgraph "Utilities"
        ADAPTOR[XAConnectionAdaptor]
        EXECUTOR[PriorityScheduledExecutor]
        CACHE[ConnectionCache]
    end
    
    subgraph "JDBC Providers"
        XADS[XADataSource]
        DS_JDBC[DataSource]
        DRIVER[JDBC Driver]
    end
    
    subgraph "Transaction Manager (Optional)"
        TM[Transaction Manager<br/>Narayana/Atomikos]
        TI_IMPL[TransactionIntegration Impl]
    end
    
    APP -->|uses| JDBC_API
    JDBC_API -->|via| IFACE_DS
    IFACE_DS -->|implemented by| DS
    DS -->|delegates to| POOL
    POOL -->|manages| HANDLER
    POOL -->|uses| FACTORY
    POOL -->|integrates| IFACE_TI
    HANDLER -->|wraps| XA_WRAP
    HANDLER -->|wraps| CONN_WRAP
    XA_WRAP -->|tracks| XARES_WRAP
    FACTORY -->|creates from| XADS
    FACTORY -->|creates from| DS_JDBC
    FACTORY -->|creates from| DRIVER
    FACTORY -->|may wrap with| ADAPTOR
    POOL -->|schedules with| EXECUTOR
    POOL -->|thread-local| CACHE
    TM -.->|implements| TI_IMPL
    TI_IMPL -.->|plugs into| POOL
    
    style APP fill:#e1f5ff
    style POOL fill:#fff5e1
    style HANDLER fill:#fff5e1
    style TM fill:#e8f5e9
```

## 2. Connection Creation Flow

```mermaid
sequenceDiagram
    participant App as Application
    participant DS as DataSource
    participant Pool as ConnectionPool
    participant Factory as ConnectionFactory
    participant Provider as XADataSource/Driver
    participant Handler as ConnectionHandler
    
    App->>DS: getConnection()
    DS->>Pool: getConnection()
    
    alt Connection Available in Pool
        Pool->>Pool: Find CHECKED_IN handler
        Pool->>Handler: acquire()
        Handler->>Handler: setState(CHECKED_IN, CHECKED_OUT)
        Handler-->>Pool: success
    else Need to Create New Connection
        Pool->>Pool: Check capacity < maxSize
        Pool->>Factory: createConnection()
        Factory->>Provider: getXAConnection() or connect()
        Provider-->>Factory: XAConnection / Connection
        
        alt XA Mode
            Factory-->>Pool: XAConnection
        else Non-XA Mode
            Factory->>Factory: new XAConnectionAdaptor(connection)
            Factory-->>Pool: XAConnectionAdaptor
        end
        
        Pool->>Handler: new ConnectionHandler(xaConnection, pool)
        Handler->>Handler: setState(NEW)
        Pool->>Pool: allConnections.add(handler)
        Handler->>Handler: setState(CHECKED_IN)
        Pool->>Handler: acquire()
        Handler->>Handler: setState(CHECKED_IN, CHECKED_OUT)
    end
    
    Pool->>Handler: connectionWrapper()
    Handler-->>Pool: ConnectionWrapper
    Pool-->>DS: Connection
    DS-->>App: Connection
    
    App->>App: Use connection
    
    App->>DS: connection.close()
    DS->>Pool: returnConnectionHandler(handler)
    Pool->>Handler: resetConnection()
    Pool->>Handler: setState(CHECKED_OUT, CHECKED_IN)
    Handler-->>Pool: Back in pool
```

## 3. State Machine for ConnectionHandler

```mermaid
stateDiagram-v2
    [*] --> NEW: Connection created
    
    NEW --> CHECKED_IN: Added to pool<br/>setState(CHECKED_IN)
    
    CHECKED_IN --> CHECKED_OUT: Acquired by thread<br/>acquire()
    CHECKED_IN --> VALIDATION: Periodic validation<br/>ValidationTask
    CHECKED_IN --> FLUSH: Max lifetime/reap<br/>FlushTask
    
    CHECKED_OUT --> VALIDATION: Borrow validation<br/>if enabled
    CHECKED_OUT --> CHECKED_IN: Returned to pool<br/>returnConnectionHandler()
    CHECKED_OUT --> FLUSH: Fatal error<br/>setFlushOnly()
    
    VALIDATION --> CHECKED_OUT: Valid<br/>borrow path
    VALIDATION --> CHECKED_IN: Valid<br/>validation path
    VALIDATION --> FLUSH: Invalid<br/>performValidation()
    
    FLUSH --> DESTROYED: Close connection<br/>DestroyConnectionTask
    
    DESTROYED --> [*]
    
    note right of CHECKED_IN
        Available for use
        In pool queue
    end note
    
    note right of CHECKED_OUT
        In use by application
        Tracked by thread
    end note
    
    note right of VALIDATION
        Connection health check
        Temporary state
    end note
    
    note right of FLUSH
        Marked for removal
        Will be destroyed
    end note
```

## 4. Thread Interaction Patterns

```mermaid
graph TB
    subgraph "Application Threads"
        T1[Thread 1]
        T2[Thread 2]
        T3[Thread 3]
    end
    
    subgraph "Connection Pool"
        QUEUE[TransferQueue<br/>LinkedTransferQueue]
        ALL[allConnections<br/>StampedCopyOnWriteArrayList]
        CACHE[Local Cache<br/>ThreadLocal]
    end
    
    subgraph "Housekeeping Thread"
        EXEC[PriorityScheduledExecutor]
        CREATE[CreateConnectionTask]
        VALIDATE[ValidationTask]
        LEAK[LeakTask]
        REAP[ReapTask]
        DESTROY[DestroyConnectionTask]
    end
    
    T1 -->|1. Check cache| CACHE
    T1 -->|2. Poll if empty| QUEUE
    T1 -->|3. Scan if needed| ALL
    
    T2 -->|Return connection| QUEUE
    QUEUE -->|tryTransfer| T3
    
    T2 -->|Put in cache| CACHE
    
    EXEC -->|Schedule| CREATE
    EXEC -->|Schedule| VALIDATE
    EXEC -->|Schedule| LEAK
    EXEC -->|Schedule| REAP
    
    CREATE -->|Add to| ALL
    CREATE -->|Offer to| QUEUE
    
    VALIDATE -->|Check| ALL
    LEAK -->|Detect in| ALL
    REAP -->|Remove from| ALL
    REAP -->|Execute| DESTROY
    
    style QUEUE fill:#e3f2fd
    style ALL fill:#fff3e0
    style CACHE fill:#f3e5f5
    style EXEC fill:#e8f5e9
```

## 5. XA Transaction Flow with Agroal

```mermaid
sequenceDiagram
    participant App as Application
    participant TM as Transaction Manager
    participant TI as TransactionIntegration
    participant Pool as ConnectionPool
    participant Handler as ConnectionHandler
    participant XA as XAResource
    participant DB as Database
    
    App->>TM: tm.begin()
    activate TM
    TM->>TM: Create Transaction<br/>Generate XID
    
    App->>Pool: getConnection()
    Pool->>TI: getTransactionAware()
    TI-->>Pool: null (first connection)
    
    Pool->>Pool: Get available handler
    Pool->>TI: associate(handler, xaResource)
    TI->>TM: Register XAResource
    TM->>XA: start(xid, TMNOFLAGS)
    activate XA
    XA->>DB: BEGIN XA 'xid'
    DB-->>XA: OK
    XA-->>TM: Started
    TI->>Handler: Mark as enlisted
    
    Pool-->>App: Connection
    
    App->>DB: executeUpdate(...)
    DB-->>App: Result
    
    Note over App,DB: Multiple SQL operations...
    
    App->>TM: tm.commit()
    TM->>XA: end(xid, TMSUCCESS)
    deactivate XA
    XA->>DB: END XA 'xid'
    
    TM->>XA: prepare(xid)
    activate XA
    XA->>DB: PREPARE XA 'xid'
    DB-->>XA: OK / XA_RDONLY
    XA-->>TM: Vote: OK
    
    alt All resources vote OK
        TM->>XA: commit(xid, false)
        XA->>DB: COMMIT XA 'xid'
        DB-->>XA: Committed
        XA-->>TM: Committed
    else Any resource fails
        TM->>XA: rollback(xid)
        XA->>DB: ROLLBACK XA 'xid'
        DB-->>XA: Rolled back
        XA-->>TM: Rolled back
    end
    deactivate XA
    
    TM->>TI: Transaction complete
    TI->>Handler: transactionEnd()
    Handler->>Pool: returnConnectionHandler(this)
    Pool->>Handler: setState(CHECKED_OUT, CHECKED_IN)
    
    deactivate TM
```

## 6. Connection Pool Sizing Logic

```mermaid
graph TD
    START[Connection Request] --> CHECK_LOCAL{Check<br/>Local Cache}
    
    CHECK_LOCAL -->|Found| ACQUIRE[Try Acquire]
    CHECK_LOCAL -->|Empty| CHECK_QUEUE
    
    ACQUIRE -->|Success| RETURN[Return Connection]
    ACQUIRE -->|Failed| CHECK_QUEUE
    
    CHECK_QUEUE{Check<br/>Transfer Queue} -->|Available| ACQUIRE
    CHECK_QUEUE -->|Empty| SCAN_ALL
    
    SCAN_ALL[Scan allConnections] -->|Found CHECKED_IN| ACQUIRE
    SCAN_ALL -->|None Available| CHECK_SIZE
    
    CHECK_SIZE{size < maxSize?}
    CHECK_SIZE -->|Yes| CREATE[CreateConnectionTask]
    CHECK_SIZE -->|No| WAIT[Wait on Transfer Queue]
    
    CREATE -->|Success| NEW_HANDLER[New ConnectionHandler]
    CREATE -->|Failed| ERROR[SQLException]
    
    NEW_HANDLER --> ACQUIRE
    
    WAIT -->|Timeout| TIMEOUT[Acquisition Timeout]
    WAIT -->|Available| ACQUIRE
    
    TIMEOUT -->|Last effort| SCAN_ALL_FINAL[Final Scan]
    SCAN_ALL_FINAL -->|Found| ACQUIRE
    SCAN_ALL_FINAL -->|None| ERROR
    
    RETURN --> END[Use Connection]
    ERROR --> END_ERROR[Throw Exception]
    
    style RETURN fill:#c8e6c9
    style ERROR fill:#ffcdd2
    style CREATE fill:#fff9c4
    style WAIT fill:#e1bee7
```

## 7. Housekeeping Tasks Timeline

```mermaid
gantt
    title Agroal Housekeeping Tasks
    dateFormat X
    axisFormat %S
    
    section Leak Detection
    LeakTask runs every N seconds :active, leak1, 0, 30
    Check each connection for leak :leak2, 30, 31
    LeakTask runs again :leak3, 60, 90
    
    section Validation
    ValidationTask runs every M seconds :active, val1, 0, 45
    Validate idle connections :val2, 45, 47
    ValidationTask runs again :val3, 90, 135
    
    section Reaping
    ReapTask runs every R seconds :active, reap1, 0, 60
    Reap idle connections > minSize :reap2, 60, 62
    ReapTask runs again :reap3, 120, 180
    
    section Creation
    CreateConnectionTask (on-demand) :crit, create1, 5, 6
    CreateConnectionTask (fill to min) :crit, create2, 65, 66
    
    section Destruction
    DestroyConnectionTask :destroy1, 62, 63
    DestroyConnectionTask :destroy2, 125, 126
```

## 8. Memory Layout and Data Structures

```mermaid
graph TB
    subgraph "ConnectionPool Instance"
        CONFIG[AgroalConnectionPoolConfiguration]
        LISTENERS[AgroalDataSourceListener array]
        ALL_CONN[StampedCopyOnWriteArrayList<br/>allConnections]
        TRANSFER_Q[LinkedTransferQueue<br/>handlerTransferQueue]
        FACTORY_REF[ConnectionFactory reference]
        EXECUTOR_REF[PriorityScheduledExecutor]
        TX_INTEGRATION[TransactionIntegration]
        LOCAL_CACHE_REF[ConnectionCache ThreadLocal]
        METRICS[MetricsRepository]
    end
    
    subgraph "ConnectionHandler Instances"
        H1[ConnectionHandler 1]
        H2[ConnectionHandler 2]
        H3[ConnectionHandler 3]
        H_N[ConnectionHandler N]
    end
    
    subgraph "Each ConnectionHandler contains"
        XA_CONN[XAConnection reference]
        CONN[Connection reference]
        XA_RES[XAResource reference]
        POOL_REF[Pool reference]
        STATE[volatile State]
        THREAD[holdingThread]
        LAST_ACCESS[lastAccess timestamp]
        ENLISTED[enlisted flag]
        DIRTY[dirtyAttributes Set]
    end
    
    subgraph "Thread Local Storage"
        T1_CACHE[Thread 1 Cache]
        T2_CACHE[Thread 2 Cache]
        TN_CACHE[Thread N Cache]
    end
    
    ALL_CONN -->|contains| H1
    ALL_CONN -->|contains| H2
    ALL_CONN -->|contains| H3
    ALL_CONN -->|contains| H_N
    
    TRANSFER_Q -.->|may contain| H1
    TRANSFER_Q -.->|may contain| H2
    
    H1 -->|structure| XA_CONN
    H1 -->|structure| CONN
    H1 -->|structure| STATE
    
    LOCAL_CACHE_REF -->|thread 1| T1_CACHE
    LOCAL_CACHE_REF -->|thread 2| T2_CACHE
    LOCAL_CACHE_REF -->|thread N| TN_CACHE
    
    T1_CACHE -.->|may cache| H2
    T2_CACHE -.->|may cache| H3
    
    style ALL_CONN fill:#fff9c4
    style TRANSFER_Q fill:#e1bee7
    style STATE fill:#ffccbc
```

## 9. Class Hierarchy

```mermaid
classDiagram
    class AgroalDataSource {
        <<interface>>
        +getConnection() Connection
        +close()
        +getMetrics() AgroalDataSourceMetrics
        +flush(FlushMode)
    }
    
    class DataSource {
        -Pool connectionPool
        -AgroalDataSourceConfiguration config
        +getConnection() Connection
        +close()
    }
    
    class Pool {
        <<interface>>
        +init()
        +getConnection() Connection
        +returnConnectionHandler(handler)
        +close()
    }
    
    class ConnectionPool {
        -StampedCopyOnWriteArrayList allConnections
        -TransferQueue handlerTransferQueue
        -ConnectionFactory connectionFactory
        -PriorityScheduledExecutor housekeepingExecutor
        +getConnection() Connection
        +returnConnectionHandler(handler)
    }
    
    class ConnectionHandler {
        -XAConnection xaConnection
        -Connection connection
        -XAResource xaResource
        -volatile State state
        -boolean enlisted
        +acquire() boolean
        +connectionWrapper() ConnectionWrapper
        +xaConnectionWrapper() XAConnectionWrapper
        +resetConnection()
        +transactionStart()
        +transactionEnd()
    }
    
    class ConnectionFactory {
        -XADataSource xaDataSource
        -DataSource dataSource
        -Driver driver
        +createConnection() XAConnection
        +getRecoveryConnection() XAConnection
    }
    
    class TransactionIntegration {
        <<interface>>
        +getTransactionAware() TransactionAware
        +associate(connection, xaResource)
        +disassociate(connection) boolean
    }
    
    class TransactionAware {
        <<interface>>
        +transactionStart()
        +transactionCommit()
        +transactionRollback()
        +transactionEnd()
        +setFlushOnly()
    }
    
    class XAConnection {
        <<interface>>
        +getConnection() Connection
        +getXAResource() XAResource
        +close()
    }
    
    class XAConnectionAdaptor {
        -Connection connection
        +getConnection() Connection
        +getXAResource() XAResource
        +close()
    }
    
    AgroalDataSource <|.. DataSource
    Pool <|.. ConnectionPool
    TransactionAware <|.. ConnectionHandler
    XAConnection <|.. XAConnectionAdaptor
    
    DataSource --> Pool
    ConnectionPool --> ConnectionHandler
    ConnectionPool --> ConnectionFactory
    ConnectionPool --> TransactionIntegration
    ConnectionHandler --> XAConnection
    ConnectionFactory --> XAConnection
    ConnectionFactory --> XAConnectionAdaptor
```

## 10. Comparison: Connection Acquisition Patterns

```mermaid
graph TB
    subgraph "HikariCP (Non-XA only)"
        H_APP[Application] -->|getConnection| H_DS[HikariDataSource]
        H_DS -->|borrow from| H_BAG[ConcurrentBag]
        H_BAG -->|manages| H_PROXY[ProxyConnection]
        H_PROXY -->|wraps| H_JDBC[JDBC Connection]
        H_JDBC -->|from| H_DRIVER[Driver/DataSource]
    end
    
    subgraph "Agroal (XA + Non-XA)"
        A_APP[Application] -->|getConnection| A_DS[AgroalDataSource]
        A_DS -->|delegates to| A_POOL[ConnectionPool]
        A_POOL -->|from queue| A_TRANSFER[TransferQueue]
        A_POOL -->|or scan| A_ALL[allConnections List]
        A_ALL -->|contains| A_HANDLER[ConnectionHandler]
        A_HANDLER -->|wraps| A_XA[XAConnection]
        A_XA -->|from| A_PROVIDER[XADataSource/Driver]
        
        A_POOL -.->|integrates| A_TM[Transaction Manager]
        A_TM -.->|uses| A_XARES[XAResource]
        A_XA -->|provides| A_XARES
    end
    
    style H_BAG fill:#e3f2fd
    style A_TRANSFER fill:#e3f2fd
    style A_TM fill:#e8f5e9
    style A_XARES fill:#e8f5e9
```

---

**These diagrams visualize the architecture described in** `XA_POOLING_ANALYSIS.md`

**Rendering**: Use any Mermaid-compatible viewer or renderer:
- GitHub (native support)
- Mermaid Live Editor: https://mermaid.live/
- VS Code with Mermaid extension
- IntelliJ with Mermaid plugin
