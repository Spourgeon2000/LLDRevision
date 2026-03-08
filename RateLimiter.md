# Rate Limiter - Low Level Design (Revised)

## Requirements
// We need to support rate limiting at multiple levels: service, API endpoint, and user level
// Support multiple algorithms: Token Bucket, Fixed Window, Sliding Window
// All rate limit checks must pass for a request to be allowed

---

## 1. Core Entities

### Request
```java
class RateLimitRequest {
    String userId;
    String apiEndpoint;
    String serviceName;
}
```

### Enums
```java
enum RateLimitType {
    USER, API, SERVICE
}

enum AlgorithmType {
    TOKEN_BUCKET, FIXED_WINDOW, SLIDING_WINDOW
}

enum TimeWindow {
    SECOND, MINUTE, HOUR;
    
    long toMillis() {
        // Convert to milliseconds
    }
}
```

---

## 2. Configuration (What are the limits?)

```java
// Base configuration
abstract class RateLimitConfig {
    AlgorithmType algorithmType;
    RateLimitType rateLimitType;
}

class TokenBucketConfig extends RateLimitConfig {
    int refillRate;        // tokens per time window
    TimeWindow timeWindow;
    int capacity;          // max tokens
}

class FixedWindowConfig extends RateLimitConfig {
    TimeWindow timeWindow;
    int allowedRequests;
}

class SlidingWindowConfig extends RateLimitConfig {
    TimeWindow timeWindow;
    int allowedRequests;
}
```

---

## 3. State (Current runtime data)

```java
// Base state for algorithms
abstract class AlgorithmState {
    AlgorithmType algorithmType;
}

class TokenBucketState extends AlgorithmState {
    long lastRefillTime;
    int currentTokens;
}

class FixedWindowState extends AlgorithmState {
    long windowStartTime;
    int currentRequestCount;
}

class SlidingWindowState extends AlgorithmState {
    Queue<Long> requestTimestamps;  // FIXED: Use Queue<Long>, not List<Integer>
    
    void addTimestamp(long timestamp) {
        requestTimestamps.offer(timestamp);
    }
    
    void removeExpiredTimestamps(long currentTime, long windowSizeMs) {
        while (!requestTimestamps.isEmpty() && 
               requestTimestamps.peek() < currentTime - windowSizeMs) {
            requestTimestamps.poll();
        }
    }
    
    int getRequestCount() {
        return requestTimestamps.size();
    }
}
```

---

## 4. Rate Limit State (Combines identifier + algorithm state)

```java
// Base class - renamed from RateLimitInfo to RateLimitState for clarity
class RateLimitState {
    AlgorithmState algorithmState;
    RateLimitType rateLimitType;
}

class ServiceRateLimitState extends RateLimitState {
    String serviceName;
}

class ApiRateLimitState extends RateLimitState {
    String apiEndpoint;
}

class UserRateLimitState extends RateLimitState {
    String userId;
}
```

---

## 5. Algorithm Strategy Pattern

```java
// Strategy Pattern - Different algorithms for rate limiting
interface RateLimitAlgorithm {
    boolean allowRequest(RateLimitConfig config, AlgorithmState state);
}

class TokenBucketAlgorithm implements RateLimitAlgorithm {
    @Override
    public boolean allowRequest(RateLimitConfig config, AlgorithmState state) {
        TokenBucketState tbState = (TokenBucketState) state;
        TokenBucketConfig tbConfig = (TokenBucketConfig) config;
        
        refill(tbState, tbConfig);
        
        if (tbState.currentTokens >= 1) {
            tbState.currentTokens--;
            return true;
        }
        return false;
    }
    
    private void refill(TokenBucketState state, TokenBucketConfig config) {
        long now = System.currentTimeMillis();
        long timePassed = now - state.lastRefillTime;
        long tokensToAdd = (timePassed * config.refillRate) / config.timeWindow.toMillis();

        state.currentTokens = Math.min(config.capacity, state.currentTokens + tokensToAdd);
        state.lastRefillTime = now;
    }
}

class FixedWindowAlgorithm implements RateLimitAlgorithm {
    @Override
    public boolean allowRequest(RateLimitConfig config, AlgorithmState state) {
        FixedWindowState fwState = (FixedWindowState) state;
        FixedWindowConfig fwConfig = (FixedWindowConfig) config;

        long now = System.currentTimeMillis();
        long windowSize = fwConfig.timeWindow.toMillis();

        // Check if window expired, reset if needed
        if (now - fwState.windowStartTime >= windowSize) {
            fwState.windowStartTime = now;
            fwState.currentRequestCount = 0;
        }

        // Check if within limit
        if (fwState.currentRequestCount < fwConfig.allowedRequests) {
            fwState.currentRequestCount++;
            return true;
        }
        return false;
    }
}

class SlidingWindowAlgorithm implements RateLimitAlgorithm {
    @Override
    public boolean allowRequest(RateLimitConfig config, AlgorithmState state) {
        SlidingWindowState swState = (SlidingWindowState) state;
        SlidingWindowConfig swConfig = (SlidingWindowConfig) config;

        long now = System.currentTimeMillis();
        long windowSize = swConfig.timeWindow.toMillis();

        // Remove expired timestamps
        swState.removeExpiredTimestamps(now, windowSize);

        // Check if within limit
        if (swState.getRequestCount() < swConfig.allowedRequests) {
            swState.addTimestamp(now);
            return true;
        }
        return false;
    }
}
```

---

## 6. Algorithm Factory Pattern

```java
// Factory Pattern - Creates appropriate algorithm based on type
class RateLimitAlgorithmFactory {
    private Map<AlgorithmType, RateLimitAlgorithm> algorithms;

    public RateLimitAlgorithmFactory() {
        algorithms = new HashMap<>();
        algorithms.put(AlgorithmType.TOKEN_BUCKET, new TokenBucketAlgorithm());
        algorithms.put(AlgorithmType.FIXED_WINDOW, new FixedWindowAlgorithm());
        algorithms.put(AlgorithmType.SLIDING_WINDOW, new SlidingWindowAlgorithm());
    }

    public RateLimitAlgorithm getAlgorithm(AlgorithmType type) {
        return algorithms.get(type);
    }
}
```

---

## 7. Chain of Responsibility + Template Method Pattern

```java
// Chain of Responsibility + Template Method Pattern
abstract class RateLimitHandler {
    protected RateLimitHandler next;
    protected RateLimitConfigRepo configRepo;
    protected RateLimitStateRepo stateRepo;
    protected RateLimitAlgorithmFactory algorithmFactory;

    // Chain of Responsibility - delegates to next handler
    public boolean allowRequest(RateLimitRequest request) {
        if (canHandle(request)) {
            boolean allowed = checkRateLimit(request);
            if (!allowed) {
                return false; // Blocked by this handler
            }
        }

        // Continue to next handler
        if (next != null) {
            return next.allowRequest(request);
        }

        return true; // All checks passed
    }

    // Template Method - common logic for all handlers
    protected boolean checkRateLimit(RateLimitRequest request) {
        // Step 1: Extract key (different for each handler)
        String key = extractKey(request);

        // Step 2: Get config (common)
        RateLimitConfig config = configRepo.getConfig(key, getRateLimitType());

        // Step 3: Get state (common)
        RateLimitState state = stateRepo.getState(key, getRateLimitType());

        // Step 4: Get algorithm (common)
        RateLimitAlgorithm algorithm = algorithmFactory.getAlgorithm(config.getAlgorithmType());

        // Step 5: Check with algorithm (common)
        boolean allowed = algorithm.allowRequest(config, state.getAlgorithmState());

        // Step 6: Update state if allowed (common)
        if (allowed) {
            stateRepo.updateState(key, getRateLimitType(), state);
        }

        return allowed;
    }

    // Setter for chaining
    public void setNext(RateLimitHandler next) {
        this.next = next;
    }

    // Abstract methods - subclasses implement these
    protected abstract boolean canHandle(RateLimitRequest request);
    protected abstract String extractKey(RateLimitRequest request);
    protected abstract RateLimitType getRateLimitType();
}
```

---

## 8. Concrete Handler Implementations

```java
class UserRateLimitHandler extends RateLimitHandler {

    @Override
    protected boolean canHandle(RateLimitRequest request) {
        return request.getUserId() != null;
    }

    @Override
    protected String extractKey(RateLimitRequest request) {
        return request.getUserId();
    }

    @Override
    protected RateLimitType getRateLimitType() {
        return RateLimitType.USER;
    }
}

class ApiRateLimitHandler extends RateLimitHandler {

    @Override
    protected boolean canHandle(RateLimitRequest request) {
        return request.getApiEndpoint() != null;
    }

    @Override
    protected String extractKey(RateLimitRequest request) {
        return request.getApiEndpoint();
    }

    @Override
    protected RateLimitType getRateLimitType() {
        return RateLimitType.API;
    }
}

class ServiceRateLimitHandler extends RateLimitHandler {

    @Override
    protected boolean canHandle(RateLimitRequest request) {
        return request.getServiceName() != null;
    }

    @Override
    protected String extractKey(RateLimitRequest request) {
        return request.getServiceName();
    }

    @Override
    protected RateLimitType getRateLimitType() {
        return RateLimitType.SERVICE;
    }
}
```

---

## 9. Repository Interfaces

```java
// Repository for configurations
interface RateLimitConfigRepo {
    RateLimitConfig getConfig(String key, RateLimitType type);
    void saveConfig(String key, RateLimitType type, RateLimitConfig config);
}

// Repository for runtime state (renamed from RateLimitInfoRepo)
interface RateLimitStateRepo {
    RateLimitState getState(String key, RateLimitType type);
    void updateState(String key, RateLimitType type, RateLimitState state);
}
```

---

## 10. Handler Factory Pattern

```java
// Factory Pattern - Creates handlers with dependencies injected
class RateLimitHandlerFactory {
    private RateLimitConfigRepo configRepo;
    private RateLimitStateRepo stateRepo;
    private RateLimitAlgorithmFactory algorithmFactory;

    public RateLimitHandlerFactory(RateLimitConfigRepo configRepo,
                                   RateLimitStateRepo stateRepo,
                                   RateLimitAlgorithmFactory algorithmFactory) {
        this.configRepo = configRepo;
        this.stateRepo = stateRepo;
        this.algorithmFactory = algorithmFactory;
    }

    public RateLimitHandler createUserHandler() {
        return createHandler(new UserRateLimitHandler());
    }

    public RateLimitHandler createApiHandler() {
        return createHandler(new ApiRateLimitHandler());
    }

    public RateLimitHandler createServiceHandler() {
        return createHandler(new ServiceRateLimitHandler());
    }

    // Common method to inject dependencies
    private RateLimitHandler createHandler(RateLimitHandler handler) {
        handler.configRepo = configRepo;
        handler.stateRepo = stateRepo;
        handler.algorithmFactory = algorithmFactory;
        return handler;
    }
}
```

---

## 11. Service Layer

```java
// Service - Orchestrates the rate limiting flow
class RateLimitService {
    private RateLimitHandler handlerChain;

    // Constructor - builds the handler chain using factory
    public RateLimitService(RateLimitHandlerFactory handlerFactory) {
        // Create handlers using factory (follows DIP - Dependency Inversion Principle)
        RateLimitHandler userHandler = handlerFactory.createUserHandler();
        RateLimitHandler apiHandler = handlerFactory.createApiHandler();
        RateLimitHandler serviceHandler = handlerFactory.createServiceHandler();

        // Chain them: User -> API -> Service
        userHandler.setNext(apiHandler);
        apiHandler.setNext(serviceHandler);

        // Set the head of the chain
        this.handlerChain = userHandler;
    }

    // Main method - check if request is allowed
    public boolean allowRequest(RateLimitRequest request) {
        return handlerChain.allowRequest(request);
    }
}
```

---

## 12. Usage Example

```java
public class Main {
    public static void main(String[] args) {
        // 1. Setup repositories (can swap implementations easily - DIP)
        RateLimitConfigRepo configRepo = new InMemoryConfigRepo();
        RateLimitStateRepo stateRepo = new InMemoryStateRepo();

        // 2. Create algorithm factory
        RateLimitAlgorithmFactory algorithmFactory = new RateLimitAlgorithmFactory();

        // 3. Create handler factory
        RateLimitHandlerFactory handlerFactory = new RateLimitHandlerFactory(
            configRepo, stateRepo, algorithmFactory
        );

        // 4. Create service (builds handler chain internally)
        RateLimitService rateLimitService = new RateLimitService(handlerFactory);

        // 5. Create request
        RateLimitRequest request = new RateLimitRequest(
            "user123",           // userId
            "/api/orders",       // apiEndpoint
            "OrderService"       // serviceName
        );

        // 6. Check if request is allowed
        boolean allowed = rateLimitService.allowRequest(request);

        if (allowed) {
            System.out.println("Request allowed - process it");
            // Process the request
        } else {
            System.out.println("Rate limit exceeded - reject request");
            // Return 429 Too Many Requests
        }
    }
}
```

---

## 13. Design Patterns Used

### 1. **Strategy Pattern**
- **Where**: `RateLimitAlgorithm` interface with `TokenBucketAlgorithm`, `FixedWindowAlgorithm`, `SlidingWindowAlgorithm`
- **Why**: Different rate limiting algorithms can be swapped without changing the handler logic
- **Benefit**: Easy to add new algorithms (e.g., Leaky Bucket)

### 2. **Chain of Responsibility Pattern**
- **Where**: `RateLimitHandler` chain (User → API → Service)
- **Why**: Request must pass through multiple rate limit checks
- **Benefit**: Each handler can independently block the request; easy to add/remove handlers

### 3. **Template Method Pattern**
- **Where**: `RateLimitHandler.checkRateLimit()` method
- **Why**: Common logic (get config, get state, check algorithm) is the same for all handlers
- **Benefit**: No code duplication; subclasses only implement what's different

### 4. **Factory Pattern**
- **Where**: `RateLimitAlgorithmFactory` and `RateLimitHandlerFactory`
- **Why**: Centralize object creation and dependency injection
- **Benefit**: Loose coupling; easy to change implementations

### 5. **Dependency Inversion Principle (DIP)**
- **Where**: Service depends on interfaces (`RateLimitConfigRepo`, `RateLimitStateRepo`), not concrete classes
- **Why**: High-level modules shouldn't depend on low-level modules
- **Benefit**: Can swap InMemory → Redis → Database without changing service code

---

## 14. Request Flow Diagram

```
Request: userId="user123", apiEndpoint="/api/orders", serviceName="OrderService"
    ↓
RateLimitService.allowRequest(request)
    ↓
handlerChain.allowRequest(request)  // Starts with UserHandler
    ↓
┌─────────────────────────────────────────────────────────────┐
│ UserRateLimitHandler                                        │
│  ├─ canHandle()? → YES (userId exists)                      │
│  ├─ checkRateLimit()                                        │
│  │   ├─ extractKey() → "user123"                            │
│  │   ├─ configRepo.getConfig("user123", USER)               │
│  │   ├─ stateRepo.getState("user123", USER)                 │
│  │   ├─ algorithmFactory.getAlgorithm(TOKEN_BUCKET)         │
│  │   ├─ algorithm.allowRequest() → TRUE                     │
│  │   └─ stateRepo.updateState()                             │
│  └─ Continue to next handler                                │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ ApiRateLimitHandler                                         │
│  ├─ canHandle()? → YES (apiEndpoint exists)                 │
│  ├─ checkRateLimit()                                        │
│  │   ├─ extractKey() → "/api/orders"                        │
│  │   ├─ configRepo.getConfig("/api/orders", API)            │
│  │   ├─ stateRepo.getState("/api/orders", API)              │
│  │   ├─ algorithmFactory.getAlgorithm(SLIDING_WINDOW)       │
│  │   ├─ algorithm.allowRequest() → TRUE                     │
│  │   └─ stateRepo.updateState()                             │
│  └─ Continue to next handler                                │
└─────────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────────┐
│ ServiceRateLimitHandler                                     │
│  ├─ canHandle()? → YES (serviceName exists)                 │
│  ├─ checkRateLimit()                                        │
│  │   ├─ extractKey() → "OrderService"                       │
│  │   ├─ configRepo.getConfig("OrderService", SERVICE)       │
│  │   ├─ stateRepo.getState("OrderService", SERVICE)         │
│  │   ├─ algorithmFactory.getAlgorithm(FIXED_WINDOW)         │
│  │   ├─ algorithm.allowRequest() → TRUE                     │
│  │   └─ stateRepo.updateState()                             │
│  └─ No more handlers                                        │
└─────────────────────────────────────────────────────────────┘
    ↓
Return TRUE (all checks passed)
    ↓
Service returns true
```

---

## 15. Key Improvements from Original Design

| Issue | Original | Revised |
|-------|----------|---------|
| **Sliding Window Type** | `List<Integer>` | `Queue<Long>` with cleanup methods |
| **Chain Logic** | Inverted, had typo | Correct flow, clean code |
| **Code Duplication** | Each handler duplicated logic | Template Method eliminates duplication |
| **Object Creation** | Manual creation | Factory Pattern |
| **Naming** | `RateLimitInfo` (confusing) | `RateLimitState` (clear) |
| **Strategy Pattern** | Mixed with Template Method | Clean separation |
| **Dependencies** | Concrete classes | Interfaces (DIP) |
| **Handler Type** | Concrete types | Abstract type (`RateLimitHandler`) |

---

## 16. Summary

**Design Patterns**: Strategy, Chain of Responsibility, Template Method, Factory
**SOLID Principles**: Dependency Inversion Principle, Single Responsibility
**Key Features**: Multi-level rate limiting, Multiple algorithms, Extensible design
**Benefits**: Easy to test, Easy to extend, Loose coupling, No code duplication

