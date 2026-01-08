# Porting Agroal Housekeeping to Custom XAConnection Pool - Implementation Guide

## Executive Summary

This document provides a complete specification for porting **only the housekeeping functionality** from Agroal to your custom XAConnection pooling implementation. This includes leak detection, idle connection validation, connection reaping, and max lifetime management.

**Estimated Effort**: 3-5 days for core implementation + 2-3 days for testing
**LOC Estimate**: ~800-1000 lines (including tests)
**Complexity**: Medium - requires careful thread safety and timing

---

## Table of Contents

1. [What is Housekeeping?](#what-is-housekeeping)
2. [Components to Port](#components-to-port)
3. [Required Classes](#required-classes)
4. [Implementation Specification](#implementation-specification)
5. [Testing Strategy](#testing-strategy)
6. [Integration Points](#integration-points)
7. [Caveats and Gotchas](#caveats-and-gotchas)
8. [Copilot Implementation Prompt](#copilot-implementation-prompt)

---

## What is Housekeeping?

Housekeeping in Agroal refers to **background maintenance tasks** that run periodically to ensure pool health:

1. **Leak Detection** - Identifies connections held too long by application threads
2. **Idle Validation** - Validates connections that have been idle for a configured period
3. **Connection Reaping** - Removes idle connections when pool size exceeds minimum
4. **Max Lifetime** - Removes connections that have existed longer than configured maximum

### Why Port This?

**Benefits:**
- ✅ Prevents connection leaks from degrading performance
- ✅ Detects and removes dead connections automatically
- ✅ Keeps pool size optimal (shrinks when demand is low)
- ✅ Forces connection recycling for better reliability
- ✅ Production-ready monitoring and diagnostics

**Without housekeeping, your pool will:**
- ❌ Accumulate leaked connections over time
- ❌ Keep dead connections that fail on use
- ❌ Waste resources maintaining unnecessary connections
- ❌ Have no visibility into connection health issues

---

## Components to Port

### 1. PriorityScheduledExecutor

**Purpose**: Custom executor that runs high-priority tasks immediately while scheduling regular tasks

**Key Features:**
- Extends `ScheduledThreadPoolExecutor`
- Queue for priority tasks that run before scheduled tasks
- Single daemon thread for housekeeping
- Graceful shutdown handling

**Lines of Code**: ~130 lines

### 2. Housekeeping Tasks

**Four main tasks:**

| Task | Frequency | Purpose | LOC |
|------|-----------|---------|-----|
| LeakTask | Every N seconds | Detect leaked connections | ~30 |
| ValidationTask | Every M seconds | Validate idle connections | ~35 |
| ReapTask | Every R seconds | Remove excess idle connections | ~45 |
| MaxLifetimeTask | One-shot per connection | Remove aged connections | ~15 |

**Total**: ~125 lines for tasks

### 3. Connection State Tracking

**Required fields in your connection wrapper:**

```java
private long lastAccessTime;           // For idle/leak detection
private Thread holdingThread;          // For leak detection
private StackTraceElement[] leakTrace; // For enhanced leak reporting
private Future<?> maxLifetimeTask;     // For max lifetime
private volatile State state;          // For validation state machine
```

**Required methods:**

```java
public void touch();                           // Update lastAccessTime
public boolean isIdle(Duration timeout);       // Check if idle too long
public boolean isLeak(Duration timeout);       // Check if leaked
public boolean isValid();                      // Validate connection
public boolean setState(State from, State to); // Atomic state transition
```

**LOC**: ~50 lines

### 4. Configuration

**Required timeout settings:**

```java
Duration leakTimeout;          // 0 = disabled
Duration validationTimeout;    // 0 = disabled  
Duration reapTimeout;          // 0 = disabled
Duration idleValidationTimeout;// 0 = disabled
Duration maxLifetime;          // 0 = disabled
boolean enhancedLeakReport;    // Stack traces for leaks
```

**LOC**: ~30 lines

---

## Required Classes

### Core Classes to Create

```
your.pool.housekeeping/
├── HousekeepingExecutor.java          (~130 lines)
├── LeakDetectionTask.java             (~80 lines)
├── ValidationTask.java                (~90 lines)
├── ReapTask.java                      (~100 lines)
├── HousekeepingConfig.java            (~50 lines)
└── ConnectionState.java               (~60 lines)

your.pool.core/
├── PooledConnection.java              (modify existing, add ~100 lines)
└── ConnectionPool.java                (modify existing, add ~50 lines)
```

### Dependencies

**Required:**
- Java 11+ (for `ScheduledThreadPoolExecutor`, `Duration`, etc.)
- Your existing connection pool implementation
- Optional: SLF4J or java.util.logging for logging

**No external dependencies needed** - can be implemented with JDK classes only

---

## Implementation Specification

### Step 1: Create HousekeepingExecutor

**File**: `HousekeepingExecutor.java`

```java
package your.pool.housekeeping;

import java.util.concurrent.*;
import java.util.concurrent.atomic.AtomicLong;

/**
 * Executor for housekeeping tasks with priority execution support.
 * Based on Agroal's PriorityScheduledExecutor.
 */
public class HousekeepingExecutor extends ScheduledThreadPoolExecutor {
    
    private static final Runnable EMPTY_TASK = () -> {};
    
    private final Queue<RunnableFuture<?>> priorityTasks = new ConcurrentLinkedQueue<>();
    private final HousekeepingListener listener;
    
    public HousekeepingExecutor(String threadName, HousekeepingListener listener) {
        super(1, new HousekeepingThreadFactory(threadName), new CallerRunsPolicy());
        setRemoveOnCancelPolicy(true);
        this.listener = listener;
    }
    
    /**
     * Execute a task immediately with high priority
     */
    public void executeNow(Runnable task) {
        executeNow(new FutureTask<>(task, null));
    }
    
    public <T> Future<T> executeNow(Callable<T> task) {
        return executeNow(new FutureTask<>(task));
    }
    
    private <T> Future<T> executeNow(RunnableFuture<T> future) {
        if (isShutdown()) {
            throw new RejectedExecutionException("Executor is shutdown");
        }
        priorityTasks.add(future);
        if (!future.isDone()) {
            execute(EMPTY_TASK);  // Trigger beforeExecute
        }
        return future;
    }
    
    @Override
    protected void beforeExecute(Thread thread, Runnable lowPriorityTask) {
        // Run all high-priority tasks first
        RunnableFuture<?> priorityTask;
        while ((priorityTask = priorityTasks.poll()) != null) {
            if (isShutdown()) {
                priorityTask.cancel(false);
            } else {
                try {
                    priorityTask.run();
                } catch (Throwable t) {
                    if (listener != null) {
                        listener.onHousekeepingError("Priority task failed", t);
                    }
                }
            }
        }
        super.beforeExecute(thread, lowPriorityTask);
    }
    
    @Override
    protected void afterExecute(Runnable r, Throwable t) {
        if (t != null && listener != null) {
            listener.onHousekeepingError("Task failed", t);
        }
        super.afterExecute(r, t);
    }
    
    private static class HousekeepingThreadFactory implements ThreadFactory {
        private final String threadName;
        private final AtomicLong counter = new AtomicLong();
        
        HousekeepingThreadFactory(String threadName) {
            this.threadName = threadName;
        }
        
        @Override
        public Thread newThread(Runnable r) {
            Thread thread = new Thread(r, threadName + "-" + counter.incrementAndGet());
            thread.setDaemon(true);
            return thread;
        }
    }
}

/**
 * Callback interface for housekeeping events
 */
interface HousekeepingListener {
    void onLeakDetected(Object connection, Thread holdingThread, StackTraceElement[] trace);
    void onValidationFailed(Object connection);
    void onConnectionReaped(Object connection);
    void onHousekeepingError(String message, Throwable cause);
}
```

### Step 2: Add State Tracking to PooledConnection

**Modify your existing connection wrapper class:**

```java
package your.pool.core;

import java.time.Duration;
import java.util.concurrent.Future;
import java.util.concurrent.atomic.AtomicReferenceFieldUpdater;

/**
 * Add this to your existing XAConnection wrapper
 */
public class PooledXAConnection {
    
    // Existing fields...
    private final XAConnection xaConnection;
    
    // Add these fields for housekeeping:
    private volatile State state = State.CHECKED_IN;
    private long lastAccessTime;
    private Thread holdingThread;
    private StackTraceElement[] acquisitionTrace;
    private Future<?> maxLifetimeTask;
    private boolean enlisted; // Track if in transaction
    
    private static final AtomicReferenceFieldUpdater<PooledXAConnection, State> stateUpdater =
        AtomicReferenceFieldUpdater.newUpdater(PooledXAConnection.class, State.class, "state");
    
    /**
     * Connection states for validation state machine
     */
    public enum State {
        CHECKED_IN,    // Available in pool
        CHECKED_OUT,   // In use by application
        VALIDATION,    // Being validated
        FLUSH          // Marked for removal
    }
    
    // Housekeeping methods to add:
    
    /**
     * Update last access time - call on acquire and return
     */
    public void touch() {
        lastAccessTime = System.nanoTime();
    }
    
    /**
     * Check if connection has been idle too long
     */
    public boolean isIdle(Duration timeout) {
        return System.nanoTime() - lastAccessTime > timeout.toNanos();
    }
    
    /**
     * Check if connection is leaked (checked out too long)
     */
    public boolean isLeak(Duration timeout) {
        return state == State.CHECKED_OUT && !enlisted && isIdle(timeout);
    }
    
    /**
     * Atomic state transition - critical for thread safety
     */
    public boolean setState(State expected, State newState) {
        return stateUpdater.compareAndSet(this, expected, newState);
    }
    
    /**
     * Force state (used on close)
     */
    public void setState(State newState) {
        stateUpdater.set(this, newState);
    }
    
    /**
     * Get current state
     */
    public State getState() {
        return state;
    }
    
    /**
     * Validate connection is alive
     */
    public boolean isValid() {
        try {
            // Option 1: Use JDBC isValid
            Connection conn = xaConnection.getConnection();
            return conn.isValid(5); // 5 second timeout
            
            // Option 2: Execute test query
            // try (Statement stmt = conn.createStatement()) {
            //     stmt.execute("SELECT 1");
            //     return true;
            // }
        } catch (SQLException e) {
            return false;
        }
    }
    
    /**
     * Set thread that acquired this connection (for leak detection)
     */
    public void setHoldingThread(Thread thread) {
        this.holdingThread = thread;
    }
    
    public Thread getHoldingThread() {
        return holdingThread;
    }
    
    /**
     * Set stack trace for enhanced leak reporting
     */
    public void setAcquisitionTrace(StackTraceElement[] trace) {
        this.acquisitionTrace = trace;
    }
    
    public StackTraceElement[] getAcquisitionTrace() {
        return acquisitionTrace;
    }
    
    /**
     * Set max lifetime task for this connection
     */
    public void setMaxLifetimeTask(Future<?> task) {
        this.maxLifetimeTask = task;
    }
    
    public void cancelMaxLifetimeTask() {
        if (maxLifetimeTask != null && !maxLifetimeTask.isDone()) {
            maxLifetimeTask.cancel(false);
        }
    }
    
    /**
     * Mark as enlisted in transaction (prevents leak detection)
     */
    public void setEnlisted(boolean enlisted) {
        this.enlisted = enlisted;
    }
}
```

### Step 3: Create Leak Detection Task

**File**: `LeakDetectionTask.java`

```java
package your.pool.housekeeping;

import your.pool.core.PooledXAConnection;
import java.time.Duration;
import java.util.Collection;
import java.util.concurrent.TimeUnit;

/**
 * Periodic task to detect connection leaks.
 * A leak is a connection that has been checked out too long.
 */
public class LeakDetectionTask implements Runnable {
    
    private final Collection<PooledXAConnection> allConnections;
    private final Duration leakTimeout;
    private final HousekeepingExecutor executor;
    private final HousekeepingListener listener;
    private final boolean enhancedReporting;
    
    public LeakDetectionTask(
            Collection<PooledXAConnection> allConnections,
            Duration leakTimeout,
            HousekeepingExecutor executor,
            HousekeepingListener listener,
            boolean enhancedReporting) {
        this.allConnections = allConnections;
        this.leakTimeout = leakTimeout;
        this.executor = executor;
        this.listener = listener;
        this.enhancedReporting = enhancedReporting;
    }
    
    @Override
    public void run() {
        // Reschedule self for next run
        executor.schedule(this, leakTimeout.toNanos(), TimeUnit.NANOSECONDS);
        
        // Check each connection for leaks
        for (PooledXAConnection connection : allConnections) {
            executor.execute(new CheckLeakTask(connection));
        }
    }
    
    /**
     * Inner task to check a single connection
     */
    private class CheckLeakTask implements Runnable {
        private final PooledXAConnection connection;
        
        CheckLeakTask(PooledXAConnection connection) {
            this.connection = connection;
        }
        
        @Override
        public void run() {
            if (connection.isLeak(leakTimeout)) {
                Thread thread = connection.getHoldingThread();
                StackTraceElement[] trace = enhancedReporting 
                    ? connection.getAcquisitionTrace() 
                    : null;
                
                if (listener != null) {
                    listener.onLeakDetected(connection, thread, trace);
                }
            }
        }
    }
}
```

### Step 4: Create Validation Task

**File**: `ValidationTask.java`

```java
package your.pool.housekeeping;

import your.pool.core.PooledXAConnection;
import your.pool.core.PooledXAConnection.State;
import java.time.Duration;
import java.util.Collection;
import java.util.concurrent.TimeUnit;

/**
 * Periodic task to validate idle connections.
 * Removes invalid connections from pool.
 */
public class ValidationTask implements Runnable {
    
    private final Collection<PooledXAConnection> allConnections;
    private final Duration validationTimeout;
    private final HousekeepingExecutor executor;
    private final HousekeepingListener listener;
    private final ConnectionRemover remover;
    
    public ValidationTask(
            Collection<PooledXAConnection> allConnections,
            Duration validationTimeout,
            HousekeepingExecutor executor,
            HousekeepingListener listener,
            ConnectionRemover remover) {
        this.allConnections = allConnections;
        this.validationTimeout = validationTimeout;
        this.executor = executor;
        this.listener = listener;
        this.remover = remover;
    }
    
    @Override
    public void run() {
        // Reschedule self
        executor.schedule(this, validationTimeout.toNanos(), TimeUnit.NANOSECONDS);
        
        // Validate each connection
        for (PooledXAConnection connection : allConnections) {
            executor.execute(new ValidateConnectionTask(connection));
        }
    }
    
    private class ValidateConnectionTask implements Runnable {
        private final PooledXAConnection connection;
        
        ValidateConnectionTask(PooledXAConnection connection) {
            this.connection = connection;
        }
        
        @Override
        public void run() {
            // Only validate connections in CHECKED_IN state
            if (connection.setState(State.CHECKED_IN, State.VALIDATION)) {
                boolean valid = connection.isValid();
                
                if (valid && connection.setState(State.VALIDATION, State.CHECKED_IN)) {
                    // Connection is valid, return to pool
                    return;
                } else {
                    // Connection is invalid, remove it
                    connection.setState(State.FLUSH);
                    remover.removeConnection(connection);
                    
                    if (listener != null) {
                        listener.onValidationFailed(connection);
                    }
                }
            }
        }
    }
}

/**
 * Callback interface for removing connections
 */
interface ConnectionRemover {
    void removeConnection(PooledXAConnection connection);
}
```

### Step 5: Create Reap Task

**File**: `ReapTask.java`

```java
package your.pool.housekeeping;

import your.pool.core.PooledXAConnection;
import your.pool.core.PooledXAConnection.State;
import java.time.Duration;
import java.util.Collection;
import java.util.concurrent.TimeUnit;

/**
 * Periodic task to reap (remove) idle connections when pool exceeds minimum size.
 * This keeps the pool size optimal.
 */
public class ReapTask implements Runnable {
    
    private final Collection<PooledXAConnection> allConnections;
    private final Duration reapTimeout;
    private final int minPoolSize;
    private final HousekeepingExecutor executor;
    private final HousekeepingListener listener;
    private final ConnectionRemover remover;
    private final ThreadLocalCache threadLocalCache; // Optional
    
    public ReapTask(
            Collection<PooledXAConnection> allConnections,
            Duration reapTimeout,
            int minPoolSize,
            HousekeepingExecutor executor,
            HousekeepingListener listener,
            ConnectionRemover remover,
            ThreadLocalCache threadLocalCache) {
        this.allConnections = allConnections;
        this.reapTimeout = reapTimeout;
        this.minPoolSize = minPoolSize;
        this.executor = executor;
        this.listener = listener;
        this.remover = remover;
        this.threadLocalCache = threadLocalCache;
    }
    
    @Override
    public void run() {
        // Reschedule self
        executor.schedule(this, reapTimeout.toNanos(), TimeUnit.NANOSECONDS);
        
        // Clear thread-local cache (forces fresh validation on next access)
        if (threadLocalCache != null) {
            threadLocalCache.clear();
        }
        
        // Reap each eligible connection
        for (PooledXAConnection connection : allConnections) {
            executor.execute(new ReapConnectionTask(connection));
        }
    }
    
    private class ReapConnectionTask implements Runnable {
        private final PooledXAConnection connection;
        
        ReapConnectionTask(PooledXAConnection connection) {
            this.connection = connection;
        }
        
        @Override
        public void run() {
            // Only reap if we're above minimum pool size
            if (allConnections.size() <= minPoolSize) {
                return;
            }
            
            // Try to mark connection for flushing
            if (connection.setState(State.CHECKED_IN, State.FLUSH)) {
                // Check if it's been idle long enough
                if (connection.isIdle(reapTimeout)) {
                    remover.removeConnection(connection);
                    
                    if (listener != null) {
                        listener.onConnectionReaped(connection);
                    }
                } else {
                    // Not idle enough, put it back
                    connection.setState(State.CHECKED_IN);
                }
            }
        }
    }
}

/**
 * Optional: Thread-local cache interface
 */
interface ThreadLocalCache {
    void clear();
}
```

### Step 6: Create Max Lifetime Support

**Add to ConnectionPool class:**

```java
package your.pool.core;

import java.time.Duration;
import java.util.concurrent.Future;
import java.util.concurrent.TimeUnit;

/**
 * Add this to your ConnectionPool implementation
 */
public class YourConnectionPool {
    
    private final HousekeepingExecutor housekeepingExecutor;
    private final Duration maxLifetime;
    
    /**
     * Called when creating a new connection
     */
    private PooledXAConnection createConnection() throws SQLException {
        XAConnection xaConn = xaDataSource.getXAConnection();
        PooledXAConnection pooled = new PooledXAConnection(xaConn, this);
        
        // Schedule max lifetime task if configured
        if (!maxLifetime.isZero()) {
            Future<?> task = housekeepingExecutor.schedule(
                new MaxLifetimeTask(pooled),
                maxLifetime.toNanos(),
                TimeUnit.NANOSECONDS
            );
            pooled.setMaxLifetimeTask(task);
        }
        
        return pooled;
    }
    
    /**
     * One-shot task to flush a connection after max lifetime
     */
    private class MaxLifetimeTask implements Runnable {
        private final PooledXAConnection connection;
        
        MaxLifetimeTask(PooledXAConnection connection) {
            this.connection = connection;
        }
        
        @Override
        public void run() {
            // Try to mark for flush (only works if CHECKED_IN)
            if (connection.setState(State.CHECKED_IN, State.FLUSH)) {
                removeConnection(connection);
            } else if (connection.setState(State.CHECKED_OUT, State.FLUSH)) {
                // If checked out, mark it - will be removed on return
                // Connection stays usable until returned
            }
        }
    }
    
    /**
     * Cancel max lifetime task when connection is destroyed
     */
    private void destroyConnection(PooledXAConnection connection) {
        connection.cancelMaxLifetimeTask();
        // ... rest of destruction logic
    }
}
```

### Step 7: Initialize Housekeeping

**Add to your pool initialization:**

```java
package your.pool.core;

public class YourConnectionPool {
    
    private HousekeepingExecutor housekeepingExecutor;
    private HousekeepingConfig config;
    private HousekeepingListener listener;
    
    public void initialize() {
        // Create executor
        housekeepingExecutor = new HousekeepingExecutor(
            "yourpool-housekeeping",
            listener
        );
        
        // Start leak detection if enabled
        if (!config.getLeakTimeout().isZero()) {
            LeakDetectionTask leakTask = new LeakDetectionTask(
                allConnections,
                config.getLeakTimeout(),
                housekeepingExecutor,
                listener,
                config.isEnhancedLeakReport()
            );
            housekeepingExecutor.schedule(
                leakTask,
                config.getLeakTimeout().toNanos(),
                TimeUnit.NANOSECONDS
            );
        }
        
        // Start validation if enabled
        if (!config.getValidationTimeout().isZero()) {
            ValidationTask validationTask = new ValidationTask(
                allConnections,
                config.getValidationTimeout(),
                housekeepingExecutor,
                listener,
                this::removeConnection
            );
            housekeepingExecutor.schedule(
                validationTask,
                config.getValidationTimeout().toNanos(),
                TimeUnit.NANOSECONDS
            );
        }
        
        // Start reaping if enabled
        if (!config.getReapTimeout().isZero()) {
            ReapTask reapTask = new ReapTask(
                allConnections,
                config.getReapTimeout(),
                config.getMinPoolSize(),
                housekeepingExecutor,
                listener,
                this::removeConnection,
                threadLocalCache
            );
            housekeepingExecutor.schedule(
                reapTask,
                config.getReapTimeout().toNanos(),
                TimeUnit.NANOSECONDS
            );
        }
    }
    
    public void shutdown() {
        if (housekeepingExecutor != null) {
            housekeepingExecutor.shutdown();
            try {
                housekeepingExecutor.awaitTermination(30, TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                housekeepingExecutor.shutdownNow();
            }
        }
    }
}
```

---

## Testing Strategy

### Unit Tests

**Test each component in isolation:**

#### 1. HousekeepingExecutor Tests

```java
@Test
void testPriorityExecution() {
    AtomicInteger counter = new AtomicInteger();
    HousekeepingExecutor executor = new HousekeepingExecutor("test", null);
    
    // Schedule regular task
    executor.schedule(() -> counter.addAndGet(1), 100, TimeUnit.MILLISECONDS);
    
    // Execute priority task
    executor.executeNow(() -> counter.addAndGet(10));
    
    // Priority task should run first
    Thread.sleep(50);
    assertEquals(10, counter.get());
    
    Thread.sleep(100);
    assertEquals(11, counter.get());
    
    executor.shutdown();
}

@Test
void testGracefulShutdown() {
    HousekeepingExecutor executor = new HousekeepingExecutor("test", null);
    executor.schedule(() -> {}, 1, TimeUnit.HOURS);
    
    executor.shutdown();
    assertTrue(executor.isShutdown());
    assertTrue(executor.awaitTermination(5, TimeUnit.SECONDS));
}
```

#### 2. Leak Detection Tests

```java
@Test
void testLeakDetection() throws Exception {
    List<PooledXAConnection> connections = new ArrayList<>();
    AtomicBoolean leakDetected = new AtomicBoolean(false);
    
    HousekeepingListener listener = new HousekeepingListener() {
        @Override
        public void onLeakDetected(Object conn, Thread thread, StackTraceElement[] trace) {
            leakDetected.set(true);
        }
        // ... other methods
    };
    
    HousekeepingExecutor executor = new HousekeepingExecutor("test", listener);
    
    // Create connection and mark as leaked
    PooledXAConnection conn = createMockConnection();
    conn.setState(State.CHECKED_OUT);
    conn.touch();
    connections.add(conn);
    
    LeakDetectionTask task = new LeakDetectionTask(
        connections,
        Duration.ofMillis(100),
        executor,
        listener,
        false
    );
    
    // Run immediately
    task.run();
    
    // Should not detect yet (too recent)
    Thread.sleep(50);
    assertFalse(leakDetected.get());
    
    // Wait for leak timeout
    Thread.sleep(100);
    
    // Should detect now
    assertTrue(leakDetected.get());
    
    executor.shutdown();
}

@Test
void testEnlistedConnectionNotLeaked() {
    // Connections in transaction should not be flagged as leaks
    PooledXAConnection conn = createMockConnection();
    conn.setState(State.CHECKED_OUT);
    conn.setEnlisted(true);
    conn.touch();
    
    Thread.sleep(200); // Longer than leak timeout
    
    assertFalse(conn.isLeak(Duration.ofMillis(100)));
}
```

#### 3. Validation Tests

```java
@Test
void testValidationRemovesInvalidConnections() throws Exception {
    List<PooledXAConnection> connections = new CopyOnWriteArrayList<>();
    AtomicInteger removed = new AtomicInteger();
    
    HousekeepingListener listener = new HousekeepingListener() {
        @Override
        public void onValidationFailed(Object conn) {
            connections.remove(conn);
            removed.incrementAndGet();
        }
        // ... other methods
    };
    
    ConnectionRemover remover = conn -> {
        connections.remove(conn);
        // Close connection
    };
    
    // Create valid and invalid connections
    PooledXAConnection validConn = createMockConnection(true);
    PooledXAConnection invalidConn = createMockConnection(false);
    
    connections.add(validConn);
    connections.add(invalidConn);
    
    HousekeepingExecutor executor = new HousekeepingExecutor("test", listener);
    ValidationTask task = new ValidationTask(
        connections,
        Duration.ofMillis(100),
        executor,
        listener,
        remover
    );
    
    task.run();
    Thread.sleep(200);
    
    // Only invalid should be removed
    assertEquals(1, removed.get());
    assertTrue(connections.contains(validConn));
    assertFalse(connections.contains(invalidConn));
    
    executor.shutdown();
}
```

#### 4. Reap Tests

```java
@Test
void testReapOnlyWhenAboveMinSize() throws Exception {
    List<PooledXAConnection> connections = new CopyOnWriteArrayList<>();
    int minSize = 5;
    
    // Add exactly min size connections
    for (int i = 0; i < minSize; i++) {
        connections.add(createIdleConnection());
    }
    
    AtomicInteger reaped = new AtomicInteger();
    ConnectionRemover remover = conn -> {
        connections.remove(conn);
        reaped.incrementAndGet();
    };
    
    HousekeepingExecutor executor = new HousekeepingExecutor("test", null);
    ReapTask task = new ReapTask(
        connections,
        Duration.ofMillis(100),
        minSize,
        executor,
        null,
        remover,
        null
    );
    
    task.run();
    Thread.sleep(200);
    
    // Should not reap - at min size
    assertEquals(0, reaped.get());
    assertEquals(minSize, connections.size());
    
    // Add extra connections
    for (int i = 0; i < 3; i++) {
        connections.add(createIdleConnection());
    }
    
    task.run();
    Thread.sleep(200);
    
    // Should reap excess
    assertTrue(reaped.get() > 0);
    
    executor.shutdown();
}

@Test
void testReapOnlyIdleConnections() throws Exception {
    PooledXAConnection idleConn = createIdleConnection();
    Thread.sleep(200); // Make it idle
    
    PooledXAConnection activeConn = createIdleConnection();
    activeConn.touch(); // Just accessed
    
    List<PooledXAConnection> connections = new CopyOnWriteArrayList<>();
    connections.add(idleConn);
    connections.add(activeConn);
    
    AtomicInteger reaped = new AtomicInteger();
    ConnectionRemover remover = conn -> {
        connections.remove(conn);
        reaped.incrementAndGet();
    };
    
    HousekeepingExecutor executor = new HousekeepingExecutor("test", null);
    ReapTask task = new ReapTask(
        connections,
        Duration.ofMillis(100),
        0, // min size
        executor,
        null,
        remover,
        null
    );
    
    task.run();
    Thread.sleep(200);
    
    // Only idle should be reaped
    assertEquals(1, reaped.get());
    assertFalse(connections.contains(idleConn));
    assertTrue(connections.contains(activeConn));
    
    executor.shutdown();
}
```

#### 5. Max Lifetime Tests

```java
@Test
void testMaxLifetimeRemovesConnection() throws Exception {
    List<PooledXAConnection> connections = new CopyOnWriteArrayList<>();
    AtomicInteger removed = new AtomicInteger();
    
    HousekeepingExecutor executor = new HousekeepingExecutor("test", null);
    
    PooledXAConnection conn = createMockConnection();
    connections.add(conn);
    
    // Schedule max lifetime task
    Future<?> task = executor.schedule(
        () -> {
            if (conn.setState(State.CHECKED_IN, State.FLUSH)) {
                connections.remove(conn);
                removed.incrementAndGet();
            }
        },
        100,
        TimeUnit.MILLISECONDS
    );
    
    conn.setMaxLifetimeTask(task);
    
    // Wait for max lifetime
    Thread.sleep(200);
    
    // Connection should be removed
    assertEquals(1, removed.get());
    assertEquals(State.FLUSH, conn.getState());
    
    executor.shutdown();
}
```

### Integration Tests

**Test with real database:**

```java
@Test
void testFullHousekeepingCycle() throws Exception {
    // Setup real connection pool with PostgreSQL test container
    YourConnectionPool pool = new YourConnectionPool(
        HousekeepingConfig.builder()
            .leakTimeout(Duration.ofSeconds(5))
            .validationTimeout(Duration.ofSeconds(10))
            .reapTimeout(Duration.ofSeconds(15))
            .maxLifetime(Duration.ofMinutes(1))
            .minPoolSize(2)
            .maxPoolSize(10)
            .build()
    );
    
    pool.initialize();
    
    try {
        // 1. Test leak detection
        Connection conn = pool.getConnection();
        // Don't close - should be detected as leak
        Thread.sleep(6000);
        // Check logs for leak warning
        
        // 2. Test validation
        // Kill backend connection
        // Housekeeping should detect and remove
        Thread.sleep(11000);
        
        // 3. Test reaping
        // Create many connections
        List<Connection> conns = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            conns.add(pool.getConnection());
        }
        // Return all
        for (Connection c : conns) {
            c.close();
        }
        // Wait for reap
        Thread.sleep(16000);
        // Pool should shrink to min size
        
        // 4. Test max lifetime
        // Wait for connections to age out
        Thread.sleep(61000);
        // All connections should be recycled
        
    } finally {
        pool.shutdown();
    }
}
```

### Load Tests

```java
@Test
void testHousekeepingUnderLoad() throws Exception {
    YourConnectionPool pool = createPoolWithHousekeeping();
    
    ExecutorService loadExecutor = Executors.newFixedThreadPool(50);
    AtomicLong successCount = new AtomicLong();
    AtomicLong errorCount = new AtomicLong();
    
    // Generate load for 5 minutes
    for (int i = 0; i < 10000; i++) {
        loadExecutor.submit(() -> {
            try {
                Connection conn = pool.getConnection();
                // Simulate work
                Thread.sleep(ThreadLocalRandom.current().nextInt(10, 100));
                conn.close();
                successCount.incrementAndGet();
            } catch (Exception e) {
                errorCount.incrementAndGet();
            }
        });
    }
    
    loadExecutor.shutdown();
    loadExecutor.awaitTermination(10, TimeUnit.MINUTES);
    
    // Housekeeping should have run multiple times
    // Pool should be healthy
    assertTrue(successCount.get() > 9000);
    assertTrue(errorCount.get() < 100);
    
    pool.shutdown();
}
```

---

## Integration Points

### 1. Connection Acquisition

**Modify your `getConnection()` method:**

```java
public Connection getConnection() throws SQLException {
    PooledXAConnection pooled = acquireConnection();
    
    // Update housekeeping state
    pooled.touch();
    
    // For leak detection
    if (leakDetectionEnabled) {
        pooled.setHoldingThread(Thread.currentThread());
        if (enhancedLeakReport) {
            pooled.setAcquisitionTrace(Thread.currentThread().getStackTrace());
        }
    }
    
    return pooled.getConnection();
}
```

### 2. Connection Return

**Modify your connection close/return logic:**

```java
public void returnConnection(PooledXAConnection pooled) throws SQLException {
    // Clear leak detection state
    if (leakDetectionEnabled) {
        pooled.setHoldingThread(null);
        if (enhancedLeakReport) {
            pooled.setAcquisitionTrace(null);
        }
    }
    
    // Update access time for reaping
    if (reapEnabled || idleValidationEnabled) {
        pooled.touch();
    }
    
    // Reset connection state
    pooled.setState(State.CHECKED_IN);
    
    // Return to pool
    returnToPool(pooled);
}
```

### 3. Logging Integration

**Implement HousekeepingListener:**

```java
public class LoggingHousekeepingListener implements HousekeepingListener {
    
    private static final Logger log = LoggerFactory.getLogger(LoggingHousekeepingListener.class);
    
    @Override
    public void onLeakDetected(Object connection, Thread holdingThread, StackTraceElement[] trace) {
        if (trace != null) {
            log.warn("Connection leak detected. Held by thread: {}. Acquisition trace:\n{}",
                holdingThread.getName(),
                formatStackTrace(trace));
        } else {
            log.warn("Connection leak detected. Held by thread: {}", holdingThread.getName());
        }
    }
    
    @Override
    public void onValidationFailed(Object connection) {
        log.info("Removing invalid connection from pool: {}", connection);
    }
    
    @Override
    public void onConnectionReaped(Object connection) {
        log.debug("Reaped idle connection: {}", connection);
    }
    
    @Override
    public void onHousekeepingError(String message, Throwable cause) {
        log.error("Housekeeping error: {}", message, cause);
    }
    
    private String formatStackTrace(StackTraceElement[] trace) {
        return Arrays.stream(trace)
            .map(StackTraceElement::toString)
            .collect(Collectors.joining("\n  at "));
    }
}
```

---

## Caveats and Gotchas

### 1. Thread Safety ⚠️

**Critical:** Use atomic operations for state transitions:

```java
// ❌ WRONG - Race condition
if (connection.getState() == State.CHECKED_IN) {
    connection.setState(State.VALIDATION);
}

// ✅ CORRECT - Atomic compare-and-set
if (connection.setState(State.CHECKED_IN, State.VALIDATION)) {
    // Safely transitioned
}
```

### 2. Validation Performance

**Issue:** Validating all connections on every cycle can be expensive.

**Solution:**
- Use idle validation timeout instead of validating on every cycle
- Validate only connections idle longer than threshold
- Use fast validation queries (e.g., `SELECT 1` or `isValid(timeout)`)

```java
// Only validate if idle
if (connection.isIdle(idleValidationTimeout)) {
    if (connection.setState(State.CHECKED_IN, State.VALIDATION)) {
        performValidation(connection);
    }
}
```

### 3. Max Lifetime vs Active Connections

**Issue:** Max lifetime task may try to flush an active connection.

**Solution:** Mark as FLUSH but let it complete current use:

```java
if (connection.setState(State.CHECKED_OUT, State.FLUSH)) {
    // Connection will be removed when returned
    // Still usable for current operation
}
```

### 4. Executor Shutdown

**Issue:** Housekeeping tasks may still be running during shutdown.

**Solution:** Shutdown sequence:

```java
public void shutdown() {
    // 1. Stop accepting new connections
    closed = true;
    
    // 2. Shutdown executor (stops new tasks)
    housekeepingExecutor.shutdown();
    
    // 3. Wait for running tasks
    housekeepingExecutor.awaitTermination(30, TimeUnit.SECONDS);
    
    // 4. Force shutdown if needed
    if (!housekeepingExecutor.isTerminated()) {
        housekeepingExecutor.shutdownNow();
    }
    
    // 5. Close all connections
    for (PooledXAConnection conn : allConnections) {
        conn.setState(State.FLUSH);
        closeConnection(conn);
    }
}
```

### 5. False Positive Leaks

**Issue:** Long-running queries flagged as leaks.

**Solution:**
- Set leak timeout appropriately (e.g., 5+ minutes)
- Exclude connections in transactions from leak detection
- Use transaction enlistment tracking

```java
public boolean isLeak(Duration timeout) {
    return state == State.CHECKED_OUT 
        && !enlisted          // Not in transaction
        && isIdle(timeout);
}
```

### 6. Reap Too Aggressive

**Issue:** Connections reaped too quickly, causing thrashing.

**Solution:**
- Set reap timeout significantly longer than typical usage (e.g., 10+ minutes)
- Ensure min pool size matches steady-state load
- Monitor reap rate in production

### 7. Memory Leaks in Executor

**Issue:** Canceled tasks not removed from queue.

**Solution:** Enable `setRemoveOnCancelPolicy(true)`:

```java
public HousekeepingExecutor(String name, HousekeepingListener listener) {
    super(1, new HousekeepingThreadFactory(name), new CallerRunsPolicy());
    setRemoveOnCancelPolicy(true); // ← Important!
    this.listener = listener;
}
```

### 8. Clock Drift

**Issue:** Using `System.currentTimeMillis()` affected by clock adjustments.

**Solution:** Use `System.nanoTime()` for elapsed time:

```java
// ❌ WRONG
private long lastAccessTime = System.currentTimeMillis();

public boolean isIdle(Duration timeout) {
    return System.currentTimeMillis() - lastAccessTime > timeout.toMillis();
}

// ✅ CORRECT
private long lastAccessTime = System.nanoTime();

public boolean isIdle(Duration timeout) {
    return System.nanoTime() - lastAccessTime > timeout.toNanos();
}
```

### 9. State Machine Deadlocks

**Issue:** Circular state transitions can deadlock.

**Solution:** Only allow specific transitions:

```java
public boolean setState(State expected, State newState) {
    // Validate transition
    if (!isValidTransition(expected, newState)) {
        throw new IllegalStateException("Invalid transition: " + expected + " -> " + newState);
    }
    return stateUpdater.compareAndSet(this, expected, newState);
}

private boolean isValidTransition(State from, State to) {
    switch (from) {
        case CHECKED_IN:
            return to == State.CHECKED_OUT || to == State.VALIDATION || to == State.FLUSH;
        case CHECKED_OUT:
            return to == State.CHECKED_IN || to == State.VALIDATION || to == State.FLUSH;
        case VALIDATION:
            return to == State.CHECKED_IN || to == State.FLUSH;
        case FLUSH:
            return false; // Terminal state
        default:
            return false;
    }
}
```

### 10. JMX/Metrics Integration

**Caveat:** Housekeeping events should be visible for monitoring.

**Solution:** Add metrics:

```java
public interface HousekeepingMetrics {
    long getLeaksDetected();
    long getValidationFailures();
    long getConnectionsReaped();
    long getHousekeepingErrors();
    Duration getLastHousekeepingDuration();
}
```

---

## Copilot Implementation Prompt

Here's a ready-to-use prompt for GitHub Copilot to implement this in your repository:

```
I need to add housekeeping functionality to my custom XAConnection pool implementation based on Agroal's design. The housekeeping system should include:

1. **Leak Detection** - Detect connections held too long (checked out but not returned)
2. **Idle Validation** - Validate connections that have been idle for a configured period
3. **Connection Reaping** - Remove idle connections when pool exceeds minimum size
4. **Max Lifetime** - Remove connections after a maximum lifetime

## Requirements

### Core Components to Implement:

1. **HousekeepingExecutor** (~130 lines)
   - Extends ScheduledThreadPoolExecutor
   - Supports priority task execution (high-priority tasks run before scheduled)
   - Single daemon thread named "poolname-housekeeping-N"
   - Graceful shutdown support

2. **Connection State Tracking** (modify existing connection wrapper)
   - Add fields: lastAccessTime (long), holdingThread (Thread), acquisitionTrace (StackTraceElement[]), maxLifetimeTask (Future), state (volatile State enum)
   - Add methods: touch(), isIdle(Duration), isLeak(Duration), setState(expected, new), isValid()
   - Use AtomicReferenceFieldUpdater for thread-safe state transitions
   - State enum: CHECKED_IN, CHECKED_OUT, VALIDATION, FLUSH

3. **LeakDetectionTask** (~80 lines)
   - Runnable that reschedules itself at configured interval
   - For each connection, submit CheckLeakTask to executor
   - CheckLeakTask: if connection.isLeak(timeout), call listener.onLeakDetected()
   - A leak is: state==CHECKED_OUT && !enlisted && isIdle(leakTimeout)

4. **ValidationTask** (~90 lines)
   - Runnable that reschedules itself at configured interval
   - For each connection, submit ValidateConnectionTask
   - ValidateConnectionTask: setState(CHECKED_IN, VALIDATION), call isValid(), setState(VALIDATION, CHECKED_IN) or FLUSH
   - Remove invalid connections via ConnectionRemover callback

5. **ReapTask** (~100 lines)
   - Runnable that reschedules itself at configured interval
   - Clear thread-local cache (if exists)
   - For each connection, submit ReapConnectionTask
   - ReapConnectionTask: if poolSize > minSize && setState(CHECKED_IN, FLUSH) && isIdle(reapTimeout), remove connection

6. **Max Lifetime Support** (add to pool)
   - When creating connection, schedule one-shot MaxLifetimeTask after maxLifetime duration
   - MaxLifetimeTask: setState(CHECKED_IN or CHECKED_OUT, FLUSH), remove if CHECKED_IN
   - Cancel task when connection destroyed

7. **HousekeepingConfig**
   - Fields: Duration leakTimeout, validationTimeout, reapTimeout, idleValidationTimeout, maxLifetime
   - boolean enhancedLeakReport (capture stack traces)
   - Zero duration means feature disabled

8. **HousekeepingListener** interface
   - Methods: onLeakDetected(connection, thread, trace), onValidationFailed(connection), onConnectionReaped(connection), onHousekeepingError(message, throwable)

### Integration Points:

- **On connection acquisition**: call touch(), setHoldingThread(currentThread()), optionally setAcquisitionTrace()
- **On connection return**: call touch(), setHoldingThread(null), clear acquisition trace, setState(CHECKED_IN)
- **On pool init**: create executor, schedule enabled tasks (leak, validation, reap), set maxLifetime for connections
- **On pool shutdown**: executor.shutdown(), awaitTermination(), mark all connections FLUSH, close

### Testing Requirements:

1. **Unit tests** for each task:
   - Test priority execution in HousekeepingExecutor
   - Test leak detection with mock connections
   - Test validation removes invalid connections
   - Test reaping only when above min size
   - Test reaping only idle connections
   - Test max lifetime removes aged connections

2. **Integration test** with real database (PostgreSQL test container):
   - Full housekeeping cycle
   - Leak detection with actual leaked connection
   - Validation with killed backend
   - Reaping after burst load
   - Max lifetime expiration

3. **Load test**:
   - 50 concurrent threads
   - 10,000 operations
   - Housekeeping runs during load
   - < 1% errors acceptable

### Thread Safety Considerations:

- ALWAYS use AtomicReferenceFieldUpdater for state changes
- NEVER use if-then-set pattern, always compare-and-set
- Use System.nanoTime() not currentTimeMillis() for elapsed time
- Enable setRemoveOnCancelPolicy(true) on executor
- Validate state transitions are allowed

### Configuration Recommendations:

- Leak timeout: 5+ minutes (avoid false positives on long queries)
- Validation timeout: 5-10 minutes
- Reap timeout: 10-15 minutes (avoid thrashing)
- Max lifetime: 30 minutes - 1 hour
- Idle validation: 5 minutes (validate idle connections)

### Files to Create:

```
src/main/java/your/pool/housekeeping/
├── HousekeepingExecutor.java
├── HousekeepingConfig.java
├── HousekeepingListener.java
├── LeakDetectionTask.java
├── ValidationTask.java
├── ReapTask.java
└── ConnectionRemover.java (interface)

src/main/java/your/pool/core/
├── PooledConnection.java (modify - add state tracking)
└── ConnectionPool.java (modify - add initialization)

src/test/java/your/pool/housekeeping/
├── HousekeepingExecutorTest.java
├── LeakDetectionTaskTest.java
├── ValidationTaskTest.java
├── ReapTaskTest.java
└── HousekeepingIntegrationTest.java
```

### Implementation Steps:

1. Create HousekeepingExecutor with priority queue support
2. Add state tracking fields/methods to PooledConnection
3. Implement LeakDetectionTask with reschedule logic
4. Implement ValidationTask with state machine
5. Implement ReapTask with min size check
6. Add max lifetime scheduling to pool
7. Integrate with getConnection() and returnConnection()
8. Add HousekeepingListener logging implementation
9. Write unit tests for each component
10. Write integration test with real DB
11. Write load test

Use Agroal's implementation as reference but adapt to our existing pool architecture. Focus on correctness and thread safety. Include comprehensive JavaDoc comments explaining the housekeeping lifecycle.

Start by creating the HousekeepingExecutor class and its unit tests, then proceed component by component.
```

---

## Summary

**What You're Porting:**
- Custom scheduled executor with priority support (~130 LOC)
- Three periodic tasks: leak detection, validation, reaping (~200 LOC)
- Max lifetime one-shot tasks (~20 LOC)
- Connection state tracking (~100 LOC)
- Configuration and interfaces (~80 LOC)
- Integration points (~50 LOC)

**Total Estimate:** ~800-1000 lines of code + tests

**Time Estimate:**
- Core implementation: 3-5 days
- Testing: 2-3 days
- Integration & debugging: 2-3 days
- **Total: 1-2 weeks**

**Key Success Factors:**
1. Atomic state transitions (most critical for correctness)
2. Proper timing (nanoTime, not currentTimeMillis)
3. Graceful shutdown handling
4. Comprehensive testing with real connections
5. Monitoring/metrics integration

**This gives you production-grade pool maintenance without importing the entire Agroal codebase.**

---

**Document Version**: 1.0  
**Date**: January 2026  
**Based on**: Agroal 3.0-SNAPSHOT housekeeping implementation
