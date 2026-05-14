# Design Patterns — Beginner-Friendly Revision Guide

> **What is a Design Pattern?**
> A design pattern is a **reusable solution to a commonly occurring problem** in software design. Think of it like a recipe — you don't reinvent how to bake bread every time; you follow a proven recipe and adjust ingredients. Patterns aren't code you copy-paste; they are **templates for how to structure your classes and objects**.

> **Why learn them?**
> 1. **Common vocabulary** — saying "use a Factory here" is faster than describing the structure.
> 2. **Avoid reinventing the wheel** — these solutions are battle-tested.
> 3. **Better design** — patterns push you toward loose coupling, testability, and extensibility.

> **Three categories of patterns:**
> - **Creational** → How objects are *created* (Factory, Builder)
> - **Structural** → How objects are *composed* together (Adapter)
> - **Behavioral** → How objects *communicate and behave* (Observer, Strategy, State)

---

## 1. Factory Pattern (Creational)

### The Problem
Imagine you're building a notification system. Your code needs to send messages via Email, SMS, or Push notifications. Without a pattern, you'd write:

```java
// BAD — client code knows about every concrete class
if (channel.equals("email")) {
    EmailNotification n = new EmailNotification();
    n.send("Hello");
} else if (channel.equals("sms")) {
    SmsNotification n = new SmsNotification();
    n.send("Hello");
} // ... and so on
```

**Problems:**
- Every time you add a new notification type, you have to hunt down every `if/else` and update it.
- Your business logic is tangled with object-creation logic.
- Hard to test — you can't easily substitute a fake notification.

### The Solution
**Move object creation into a dedicated class (the "Factory").** The client just says *what* it wants; the factory decides *how* to build it.

### Key Players
| Role | What it does |
|---|---|
| **Product** (interface) | The common type all created objects share (`Notification`) |
| **Concrete Products** | The actual implementations (`EmailNotification`, `SmsNotification`) |
| **Factory** | A class with a method that returns a Product based on input |

### Code Example

```java
// 1. Product interface — the common contract
interface Notification {
    void send(String message);
}

// 2. Concrete products — different implementations
class EmailNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending Email: " + message);
    }
}

class SmsNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending SMS: " + message);
    }
}

class PushNotification implements Notification {
    @Override
    public void send(String message) {
        System.out.println("Sending Push: " + message);
    }
}

// 3. The Factory — centralizes the creation logic
class NotificationFactory {
    public static Notification create(String channel) {
        return switch (channel.toLowerCase()) {
            case "email" -> new EmailNotification();
            case "sms"   -> new SmsNotification();
            case "push"  -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown channel: " + channel);
        };
    }
}

// 4. Client code — clean and unaware of concrete classes
public class Demo {
    public static void main(String[] args) {
        Notification n = NotificationFactory.create("sms");
        n.send("Your OTP is 4821");
        // Output: Sending SMS: Your OTP is 4821
    }
}
```

### Walk-through (line by line)
1. We define `Notification` so any sender shares the same `send()` method.
2. Each concrete class implements that contract differently.
3. `NotificationFactory.create()` is the **single place** that knows about concrete classes.
4. The client (`main`) only knows about `Notification` and the factory — it has zero knowledge of `EmailNotification`, `SmsNotification`, etc.

### When to Use
- You have multiple subclasses of a common type and the choice depends on runtime input (config, user choice, request type).
- You want to centralize object creation in one place.

### Real-World Examples
- `Calendar.getInstance()` in Java returns a different `Calendar` subclass based on locale.
- `LoggerFactory.getLogger()` in SLF4J.
- A `PaymentProcessorFactory` that returns Stripe, PayPal, or Square processors.

### Pros and Cons
- ✅ Decouples client from concrete classes
- ✅ Easy to add new types (just update the factory)
- ❌ Adding a new type still requires modifying the factory (use **Abstract Factory** for more flexibility)

---

## 2. Builder Pattern (Creational)

### The Problem
You need to build an `HttpRequest` object that has many fields — some required, most optional: URL, method, headers, body, timeout, retries, auth token, query params, etc.

The naive ways are painful:

```java
// Telescoping constructor — many overloads, hard to read
new HttpRequest(url);
new HttpRequest(url, "POST");
new HttpRequest(url, "POST", headers);
new HttpRequest(url, "POST", headers, body);
// What does new HttpRequest(url, "POST", headers, body, 5000, 3, null, true) even mean?

// Or: setters that allow invalid intermediate states
HttpRequest r = new HttpRequest();
r.setUrl(...);          // What if you forget?
r.setMethod(...);       // Object is mutable & might be half-built
```

### The Solution
Use a **separate Builder class** that collects parameters step-by-step via fluent (chainable) methods, then produces an immutable object when you call `build()`.

### Key Players
| Role | What it does |
|---|---|
| **Product** | The complex object you ultimately want (`HttpRequest`) |
| **Builder** | A helper class that collects the parts and assembles the product |
| **build()** | Validates and returns the finished Product |

### Code Example

```java
import java.util.*;

class HttpRequest {
    // Fields are final — object is immutable once built
    private final String method;
    private final String url;
    private final Map<String, String> headers;
    private final String body;
    private final int timeout;

    // Private constructor — only the Builder can create instances
    private HttpRequest(Builder builder) {
        this.method  = builder.method;
        this.url     = builder.url;
        this.headers = Map.copyOf(builder.headers);   // defensive copy
        this.body    = builder.body;
        this.timeout = builder.timeout;
    }

    // Static nested Builder class
    public static class Builder {
        // Required parameter
        private final String url;

        // Optional parameters — initialize with sensible defaults
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private String body = "";
        private int timeout = 30_000; // 30 seconds default

        public Builder(String url) {
            this.url = url; // required field goes in constructor
        }

        // Each setter returns 'this' so calls can be chained
        public Builder method(String method) {
            this.method = method;
            return this;
        }

        public Builder header(String key, String value) {
            this.headers.put(key, value);
            return this;
        }

        public Builder body(String body) {
            this.body = body;
            return this;
        }

        public Builder timeout(int ms) {
            this.timeout = ms;
            return this;
        }

        // Final step — validate and create the actual object
        public HttpRequest build() {
            if (url == null || url.isBlank()) {
                throw new IllegalStateException("URL is required");
            }
            return new HttpRequest(this);
        }
    }
}

// Client usage — reads almost like English
public class Demo {
    public static void main(String[] args) {
        HttpRequest req = new HttpRequest.Builder("https://api.example.com/orders")
                .method("POST")
                .header("Content-Type", "application/json")
                .header("Authorization", "Bearer xyz123")
                .body("{\"orderId\": \"OD-001\"}")
                .timeout(5000)
                .build();
    }
}
```

### Walk-through
1. The constructor of `HttpRequest` is **private** — clients can't call `new HttpRequest()` directly.
2. The `Builder` is a static nested class. It collects values into mutable fields.
3. Each setter returns `this`, enabling the fluent chain (`builder.method(...).header(...).body(...)`).
4. `build()` performs validation and creates the immutable `HttpRequest`.

### When to Use
- An object has **4+ constructor parameters**, especially if many are optional.
- You want to enforce required fields and provide defaults for optional ones.
- You want **immutable objects** (thread-safe, easier to reason about).

### Real-World Examples
- `StringBuilder` in Java.
- `Stream.builder()` in Java Streams API.
- Lombok's `@Builder` annotation auto-generates this pattern.
- OkHttp's `Request.Builder` and `OkHttpClient.Builder`.

### Pros and Cons
- ✅ Readable client code (named "parameters")
- ✅ Immutable products
- ✅ Easy to add new optional fields
- ❌ More code to write (mitigated by Lombok)

---

## 3. Observer Pattern (Behavioral)

### The Problem
When an order is placed, multiple things must happen: reserve inventory, charge the card, send a confirmation email, update analytics, log the event. The naive way:

```java
// BAD — OrderService is tightly coupled to everything
class OrderService {
    void placeOrder(String orderId) {
        // Save order...
        inventoryService.reserve(orderId);
        emailService.sendConfirmation(orderId);
        analyticsService.track(orderId);
        auditLog.write(orderId);
        // Add a new feature? Modify this method again...
    }
}
```

**Problems:**
- `OrderService` knows too much about other services (tight coupling).
- Every new "side effect" forces a code change in `OrderService`.
- Hard to test — can't easily disable email when testing.

### The Solution
Let interested parties **subscribe** to events. The publisher (Subject) just announces "this happened" without knowing who's listening.

This is the **Pub/Sub** model.

### Key Players
| Role | What it does |
|---|---|
| **Subject** (Publisher) | Maintains a list of observers and notifies them on events |
| **Observer** (Subscriber) | Defines an `update()` / `onEvent()` method called on notification |
| **Concrete Observers** | Specific reactions (send email, update inventory, etc.) |

### Code Example

```java
import java.util.*;

// 1. The Observer contract
@FunctionalInterface
interface OrderEventListener {
    void onEvent(String eventType, String orderId);
}

// 2. The Subject (publisher)
class OrderService {
    // Map of eventType -> list of subscribers
    private final Map<String, List<OrderEventListener>> listeners = new HashMap<>();

    public void subscribe(String eventType, OrderEventListener listener) {
        listeners.computeIfAbsent(eventType, k -> new ArrayList<>()).add(listener);
    }

    public void unsubscribe(String eventType, OrderEventListener listener) {
        listeners.getOrDefault(eventType, List.of()).remove(listener);
    }

    private void publish(String eventType, String orderId) {
        listeners.getOrDefault(eventType, List.of())
                 .forEach(l -> l.onEvent(eventType, orderId));
    }

    // Business method
    public void placeOrder(String orderId) {
        System.out.println("Order saved: " + orderId);
        publish("ORDER_PLACED", orderId); // notify everyone interested
    }
}

// 3. Concrete observers — each one independent
class InventoryService implements OrderEventListener {
    @Override
    public void onEvent(String eventType, String orderId) {
        System.out.println("[Inventory] Reserved stock for " + orderId);
    }
}

class EmailService implements OrderEventListener {
    @Override
    public void onEvent(String eventType, String orderId) {
        System.out.println("[Email] Confirmation sent for " + orderId);
    }
}

class AnalyticsService implements OrderEventListener {
    @Override
    public void onEvent(String eventType, String orderId) {
        System.out.println("[Analytics] Tracked event for " + orderId);
    }
}

// 4. Wiring it together
public class Demo {
    public static void main(String[] args) {
        OrderService orderService = new OrderService();

        // Subscribe each observer
        orderService.subscribe("ORDER_PLACED", new InventoryService());
        orderService.subscribe("ORDER_PLACED", new EmailService());
        orderService.subscribe("ORDER_PLACED", new AnalyticsService());

        orderService.placeOrder("ORD-5001");
        // Output:
        //   Order saved: ORD-5001
        //   [Inventory]  Reserved stock for ORD-5001
        //   [Email]      Confirmation sent for ORD-5001
        //   [Analytics]  Tracked event for ORD-5001
    }
}
```

### Walk-through
1. `OrderEventListener` is the contract — observers implement `onEvent()`.
2. `OrderService` has zero knowledge of *what* observers do — it just calls `publish()`.
3. To add a new feature (say, fraud detection), you create a new observer and subscribe it. **No change to `OrderService`.**

### When to Use
- One event needs to trigger multiple independent actions.
- You want loose coupling between an event source and its consumers.
- Building event-driven systems (UI events, message buses, reactive streams).

### Real-World Examples
- DOM `addEventListener('click', ...)` in JavaScript.
- Java's `PropertyChangeListener`.
- Spring's `ApplicationEventPublisher` and `@EventListener`.
- Kafka, RabbitMQ — the same idea at the network/distributed level.

### Pros and Cons
- ✅ Loose coupling — publisher doesn't know subscribers
- ✅ Easy to add/remove observers at runtime
- ❌ Notification order may be unpredictable
- ❌ Memory leaks if observers forget to unsubscribe

---

## 4. Strategy Pattern (Behavioral)

### The Problem
Your `PriceCalculator` needs to compute prices differently based on customer type: regular customer, bulk buyer, insurance copay, employee discount, etc.

```java
// BAD — long if/else chain that grows forever
class PriceCalculator {
    double calculate(double base, int qty, String type) {
        if (type.equals("regular")) {
            return base * qty;
        } else if (type.equals("bulk")) {
            double discount = qty >= 10 ? 0.15 : 0.0;
            return base * qty * (1 - discount);
        } else if (type.equals("insurance")) {
            return 5.00; // flat copay
        }
        // ... and so on
    }
}
```

**Problems:**
- Every new pricing rule means modifying this class (violates Open/Closed Principle).
- The class becomes a "god class" handling unrelated logic.
- Hard to test individual rules in isolation.

### The Solution
Encapsulate each algorithm in its own class implementing a common interface. The context object holds a reference to one strategy at a time and **delegates** to it.

### Key Players
| Role | What it does |
|---|---|
| **Strategy** (interface) | The contract all algorithms follow |
| **Concrete Strategies** | The actual algorithm implementations |
| **Context** | Holds a Strategy and delegates work to it |

### Code Example

```java
// 1. Strategy interface
interface PricingStrategy {
    double calculatePrice(double basePrice, int quantity);
}

// 2. Concrete strategies — each is independent and testable
class RegularPricing implements PricingStrategy {
    @Override
    public double calculatePrice(double basePrice, int quantity) {
        return basePrice * quantity;
    }
}

class BulkDiscountPricing implements PricingStrategy {
    @Override
    public double calculatePrice(double basePrice, int quantity) {
        double discount = quantity >= 10 ? 0.15
                        : quantity >= 5  ? 0.10
                        : 0.0;
        return basePrice * quantity * (1 - discount);
    }
}

class InsuranceCopayPricing implements PricingStrategy {
    private final double copayAmount;

    public InsuranceCopayPricing(double copayAmount) {
        this.copayAmount = copayAmount;
    }

    @Override
    public double calculatePrice(double basePrice, int quantity) {
        return copayAmount; // flat copay regardless of quantity/price
    }
}

// 3. Context — uses a strategy, doesn't care which one
class PriceCalculator {
    private PricingStrategy strategy;

    public PriceCalculator(PricingStrategy strategy) {
        this.strategy = strategy;
    }

    // Strategy can be swapped at runtime
    public void setStrategy(PricingStrategy strategy) {
        this.strategy = strategy;
    }

    public double compute(double basePrice, int quantity) {
        return strategy.calculatePrice(basePrice, quantity);
    }
}

// 4. Client picks the strategy
public class Demo {
    public static void main(String[] args) {
        PriceCalculator calc = new PriceCalculator(new RegularPricing());
        System.out.println(calc.compute(9.99, 3));   // 29.97

        calc.setStrategy(new BulkDiscountPricing());
        System.out.println(calc.compute(9.99, 12));  // 101.898 (15% off)

        calc.setStrategy(new InsuranceCopayPricing(5.00));
        System.out.println(calc.compute(9.99, 12));  // 5.0
    }
}
```

### Walk-through
1. `PricingStrategy` defines *what* needs to be done; each implementation defines *how*.
2. `PriceCalculator` (the Context) doesn't know or care which algorithm is in use — it just calls `strategy.calculatePrice(...)`.
3. To add a new pricing rule (e.g., "Black Friday 50% off"), create a new class. **No existing code changes.**

### When to Use
- You have multiple ways to do the same thing and want to switch between them.
- You catch yourself writing long `if/else` or `switch` blocks selecting algorithms.
- Different clients need different versions of an algorithm.

### Real-World Examples
- `Comparator` in Java's `Collections.sort()` — different comparison strategies.
- Different compression algorithms (ZIP, GZIP, BZIP2).
- Different authentication strategies (OAuth, JWT, Basic).
- Sorting algorithms — same input/output contract, different internals.

### Pros and Cons
- ✅ Eliminates long conditional chains
- ✅ Each algorithm is isolated and testable
- ✅ Easy to add new strategies
- ❌ Client must know about strategies to choose one
- ❌ More classes to manage

---

## 5. State Pattern (Behavioral)

### The Problem
An order moves through states: `NEW → PROCESSING → SHIPPED → DELIVERED`. The order's behavior depends on its current state — for example, you can cancel a `NEW` order but not a `SHIPPED` one.

```java
// BAD — every method is a giant switch statement
class Order {
    private String state = "NEW";

    void next() {
        if (state.equals("NEW"))             state = "PROCESSING";
        else if (state.equals("PROCESSING")) state = "SHIPPED";
        else if (state.equals("SHIPPED"))    state = "DELIVERED";
    }

    void cancel() {
        if (state.equals("NEW") || state.equals("PROCESSING")) {
            state = "CANCELLED";
        } else {
            throw new IllegalStateException("Cannot cancel");
        }
    }
    // ... ship(), refund(), etc. — all with the same switch
}
```

**Problems:**
- Every method repeats the same `if/else` over states.
- Adding a new state forces edits to every method.
- Hard to see all the rules for a single state in one place.

### The Solution
Make each **state its own class** with its own implementation of the operations. The context object delegates to its current state, and the state itself decides what to transition to next.

### Key Players
| Role | What it does |
|---|---|
| **Context** | Holds a reference to the current state and delegates calls to it |
| **State** (interface) | Common operations available in any state |
| **Concrete States** | Implementations for each state, including transition logic |

### Code Example

```java
// 1. State interface
interface OrderState {
    void next(OrderContext ctx);
    void prev(OrderContext ctx);
    void printStatus();
}

// 2. Concrete states — each one is responsible for "what to do next"
class NewState implements OrderState {
    @Override public void next(OrderContext ctx) { ctx.setState(new ProcessingState()); }
    @Override public void prev(OrderContext ctx) { System.out.println("Already at the start"); }
    @Override public void printStatus()          { System.out.println("State: NEW"); }
}

class ProcessingState implements OrderState {
    @Override public void next(OrderContext ctx) { ctx.setState(new ShippedState()); }
    @Override public void prev(OrderContext ctx) { ctx.setState(new NewState()); }
    @Override public void printStatus()          { System.out.println("State: PROCESSING"); }
}

class ShippedState implements OrderState {
    @Override public void next(OrderContext ctx) { ctx.setState(new DeliveredState()); }
    @Override public void prev(OrderContext ctx) { ctx.setState(new ProcessingState()); }
    @Override public void printStatus()          { System.out.println("State: SHIPPED"); }
}

class DeliveredState implements OrderState {
    @Override public void next(OrderContext ctx) { System.out.println("Already delivered"); }
    @Override public void prev(OrderContext ctx) { ctx.setState(new ShippedState()); }
    @Override public void printStatus()          { System.out.println("State: DELIVERED"); }
}

// 3. Context — knows the current state and forwards calls
class OrderContext {
    private OrderState state;

    public OrderContext() {
        this.state = new NewState(); // start state
    }

    public void setState(OrderState state) { this.state = state; }
    public void next()        { state.next(this); }
    public void prev()        { state.prev(this); }
    public void printStatus() { state.printStatus(); }
}

// 4. Client just works with the context
public class Demo {
    public static void main(String[] args) {
        OrderContext order = new OrderContext();
        order.printStatus();   // State: NEW
        order.next();
        order.printStatus();   // State: PROCESSING
        order.next();
        order.printStatus();   // State: SHIPPED
        order.next();
        order.printStatus();   // State: DELIVERED
        order.next();          // Already delivered
    }
}
```

### Walk-through
1. Each state class encapsulates the logic of "what to do when in this state".
2. The state itself is responsible for telling the context to transition (`ctx.setState(...)`).
3. The context just delegates — `order.next()` calls `state.next(this)`.
4. Adding a new state (say, `CancelledState`) means adding one class, not editing many.

### Strategy vs State — Important Distinction!
They look similar but differ in intent:
- **Strategy:** The *client* chooses which algorithm to use. Strategies don't know about each other.
- **State:** The *object itself* transitions between behaviors. States usually know about other states.

### When to Use
- An object's behavior depends on its state, and it transitions between many states.
- You're modeling a workflow, finite state machine, or lifecycle.
- You see the same `switch(state)` repeated across many methods.

### Real-World Examples
- A vending machine: NoCoin → HasCoin → Dispensing → Out of Stock.
- TCP connection states: Listen, Established, Closed.
- Document workflow: Draft → Review → Published → Archived.
- A media player: Playing, Paused, Stopped.

### Pros and Cons
- ✅ Eliminates massive `switch` statements
- ✅ Each state's behavior is in one place
- ✅ Easy to add new states
- ❌ More classes (one per state)
- ❌ States may need references to each other, creating coupling

---

## 6. Adapter Pattern (Structural)

### The Problem
Your application uses a clean `PaymentGateway` interface. You need to integrate a third-party payment library, but its interface is completely different — it uses cents instead of dollars, integer transaction IDs instead of strings, and weird method names.

You **cannot modify the third-party code**. So how do you make it work with your code?

### The Solution
Build an **Adapter** — a wrapper class that implements your interface and translates calls to the third-party API.

It's exactly like a power plug adapter: a US plug (your code) into a UK socket (third-party API).

### Key Players
| Role | What it does |
|---|---|
| **Target** (interface) | The interface your code expects (`PaymentGateway`) |
| **Adaptee** | The existing class with an incompatible interface (`LegacyPaymentProcessor`) |
| **Adapter** | Implements Target, holds an Adaptee, and translates between them |
| **Client** | Uses the Target interface — unaware of the Adaptee |

### Code Example

```java
// 1. The Target — what your code expects
interface PaymentGateway {
    boolean charge(String customerId, double amountInDollars);
    String checkStatus(String transactionId);
}

// 2. The Adaptee — third-party class you can't change
class LegacyPaymentProcessor {
    // Takes amount in cents, returns int reference
    public int makePayment(String account, int amountInCents) {
        System.out.println("Legacy paid " + amountInCents + " cents from " + account);
        return 12345;
    }

    // Takes int reference, returns status string
    public String queryPayment(int reference) {
        return "COMPLETED";
    }
}

// 3. The Adapter — bridges the two
class LegacyPaymentAdapter implements PaymentGateway {
    private final LegacyPaymentProcessor legacy;

    public LegacyPaymentAdapter(LegacyPaymentProcessor legacy) {
        this.legacy = legacy;
    }

    @Override
    public boolean charge(String customerId, double amountInDollars) {
        // Translate dollars -> cents, call adaptee, translate result
        int cents = (int) (amountInDollars * 100);
        int reference = legacy.makePayment(customerId, cents);
        return reference > 0;
    }

    @Override
    public String checkStatus(String transactionId) {
        // Translate String -> int and call adaptee
        int reference = Integer.parseInt(transactionId);
        return legacy.queryPayment(reference);
    }
}

// 4. Client code — sees only the clean interface
public class Demo {
    public static void main(String[] args) {
        PaymentGateway gateway = new LegacyPaymentAdapter(new LegacyPaymentProcessor());

        boolean ok = gateway.charge("CUST-42", 29.99);
        System.out.println("Charged: " + ok);

        String status = gateway.checkStatus("12345");
        System.out.println("Status: " + status);
    }
}
```

### Walk-through
1. The client only knows about `PaymentGateway` — clean and simple.
2. `LegacyPaymentAdapter` implements `PaymentGateway` so it can be used in client code.
3. Inside, it holds a `LegacyPaymentProcessor` and translates each call (dollars↔cents, String↔int).
4. If you switch to a new payment provider tomorrow, write a new adapter — **client code doesn't change**.

### When to Use
- You need to integrate a third-party library/legacy code with an incompatible interface.
- You want to migrate gradually from an old API to a new one.
- You're working with multiple data sources that all need to look the same to your app.

### Real-World Examples
- Java's `Arrays.asList()` adapts an array into a `List`.
- `InputStreamReader` adapts an `InputStream` (byte stream) into a `Reader` (character stream).
- Logging frameworks: SLF4J adapts to Log4j, Logback, JUL, etc.
- A `SquareToCircleAdapter` if your code expects circles but you have squares.

### Pros and Cons
- ✅ Lets incompatible code work together without modification
- ✅ Isolates third-party dependencies behind your own interface
- ✅ Easy to swap implementations
- ❌ Adds an extra layer of indirection
- ❌ Too many adapters can make the system harder to follow

---

## 7. Singleton Pattern (Creational)

### The Problem
Some objects should only ever exist **once** in your application — for example, a configuration manager, a connection pool, a logger, or a cache. If multiple parts of code create their own instance, you end up with:

```java
// BAD — every caller creates its own config
class OrderService {
    AppConfig config = new AppConfig();   // loads config from disk
}
class PaymentService {
    AppConfig config = new AppConfig();   // loads it AGAIN
}
// Now there are two configs in memory, possibly out of sync.
// Worse: heavy resources (DB pools, caches) get duplicated.
```

**Problems:**
- Wasted memory/resources (multiple copies of expensive objects).
- Inconsistent state — one part of the app updates one instance; others see stale data.
- No central point of control.

### The Solution
Make the class **control its own instantiation**: a private constructor + a public static method that always returns the same instance.

### Key Players
| Role | What it does |
|---|---|
| **Singleton class** | Hides its constructor and exposes a `getInstance()` method |
| **Static instance** | The single shared object kept inside the class |

### Code Example

```java
// Thread-safe Singleton using "lazy holder" idiom (recommended in Java)
class AppConfig {
    private final Map<String, String> properties;

    // 1. Private constructor — nobody outside can do `new AppConfig()`
    private AppConfig() {
        properties = new HashMap<>();
        properties.put("env", "production");
        properties.put("dbUrl", "jdbc:postgres://...");
        System.out.println("AppConfig loaded (this should print only ONCE)");
    }

    // 2. Inner static holder — JVM guarantees thread-safe lazy initialization
    private static class Holder {
        private static final AppConfig INSTANCE = new AppConfig();
    }

    // 3. The single global access point
    public static AppConfig getInstance() {
        return Holder.INSTANCE;
    }

    public String get(String key) {
        return properties.get(key);
    }
}

// 4. Client usage
public class Demo {
    public static void main(String[] args) {
        AppConfig a = AppConfig.getInstance();
        AppConfig b = AppConfig.getInstance();

        System.out.println(a == b);             // true — same object!
        System.out.println(a.get("env"));        // production
    }
}
```

### Walk-through
1. The constructor is **private**, so no one can create new instances from outside.
2. The `Holder` class isn't loaded until `getInstance()` is called — that's the "lazy" part.
3. The JVM guarantees class initialization is thread-safe, so `INSTANCE` is created exactly once even with many threads.
4. Every call to `getInstance()` returns the **same** object (`a == b` is `true`).

### Common Pitfalls
- **Don't use Singleton as a global variable in disguise.** It often becomes a hidden dependency that makes testing nightmarish.
- **Thread safety:** the simplest `if (instance == null) instance = new ...` is **not** thread-safe.
- **Reflection** can break Singleton — `enum` is the most reflection-proof variant in Java:
  ```java
  enum AppConfig { INSTANCE; public String get(String k) { ... } }
  ```

### When to Use
- You truly need exactly one instance for the whole app (logger, config, cache, thread pool, hardware driver).
- The instance must be globally accessible.

### Real-World Examples
- `Runtime.getRuntime()` in Java.
- Spring beans with default scope (`singleton`) — managed by the framework.
- `java.util.logging.Logger` (per-name singletons).

### Pros and Cons
- ✅ Guaranteed single instance, controlled creation
- ✅ Lazy initialization saves resources
- ❌ Hard to unit test (global state, hard to mock)
- ❌ Often considered an anti-pattern when overused — prefer **Dependency Injection**

---

## 8. Decorator Pattern (Structural)

### The Problem
You have a `Coffee` class. Now you want variations: coffee with milk, with sugar, with whipped cream, with milk + sugar, with milk + sugar + cream, etc. Subclassing every combination explodes:

```java
// BAD — combinatorial explosion of subclasses
class Coffee { ... }
class CoffeeWithMilk extends Coffee { ... }
class CoffeeWithMilkAndSugar extends CoffeeWithMilk { ... }
class CoffeeWithMilkAndSugarAndCream extends CoffeeWithMilkAndSugar { ... }
// ... what if you want sugar without milk? Add another class? :(
```

**Problems:**
- Too many subclasses (2^n combinations).
- Can't add features dynamically at runtime.
- Hard to mix and match.

### The Solution
Wrap the original object in another object (a "decorator") that adds behavior **before or after** delegating to the inner object. Decorators implement the same interface, so they can be stacked.

### Key Players
| Role | What it does |
|---|---|
| **Component** (interface) | The common type both core and decorators share (`Coffee`) |
| **Concrete Component** | The basic, undecorated object (`SimpleCoffee`) |
| **Decorator** (abstract) | Holds a reference to a Component and forwards calls |
| **Concrete Decorators** | Add specific behavior (`MilkDecorator`, `SugarDecorator`) |

### Code Example

```java
// 1. The Component interface
interface Coffee {
    String description();
    double cost();
}

// 2. The base concrete component
class SimpleCoffee implements Coffee {
    @Override public String description() { return "Coffee"; }
    @Override public double cost()        { return 2.00; }
}

// 3. Abstract decorator — wraps a Coffee
abstract class CoffeeDecorator implements Coffee {
    protected final Coffee wrapped;
    protected CoffeeDecorator(Coffee wrapped) {
        this.wrapped = wrapped;
    }
}

// 4. Concrete decorators — each adds one feature
class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee c) { super(c); }
    @Override public String description() { return wrapped.description() + ", milk"; }
    @Override public double cost()        { return wrapped.cost() + 0.50; }
}

class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee c) { super(c); }
    @Override public String description() { return wrapped.description() + ", sugar"; }
    @Override public double cost()        { return wrapped.cost() + 0.20; }
}

class WhippedCreamDecorator extends CoffeeDecorator {
    public WhippedCreamDecorator(Coffee c) { super(c); }
    @Override public String description() { return wrapped.description() + ", whip"; }
    @Override public double cost()        { return wrapped.cost() + 0.70; }
}

// 5. Client — stack decorators in any combination
public class Demo {
    public static void main(String[] args) {
        Coffee order = new WhippedCreamDecorator(
                           new SugarDecorator(
                               new MilkDecorator(
                                   new SimpleCoffee())));

        System.out.println(order.description());  // Coffee, milk, sugar, whip
        System.out.println("$" + order.cost());   // $3.40
    }
}
```

### Walk-through
1. Every layer (base + decorators) implements `Coffee`, so the client can treat any combination uniformly.
2. Each decorator wraps another `Coffee` and **adds** to its result.
3. The order of wrapping = the order of effects. You can rearrange, add, or remove layers freely.
4. **No new subclass needed** for combinations — just stack decorators.

### When to Use
- You need to add responsibilities to objects dynamically and transparently.
- Subclassing would create too many combinations.
- You want to follow the Open/Closed Principle (extend without modifying).

### Real-World Examples
- Java I/O: `new BufferedReader(new InputStreamReader(new FileInputStream(...)))` — each layer adds a feature.
- Java Collections: `Collections.unmodifiableList(list)` returns a decorated list.
- HTTP middleware in web frameworks (Express, Spring filters) — each middleware wraps the next.

### Pros and Cons
- ✅ Add behavior at runtime, in any combination
- ✅ Avoids subclass explosion
- ✅ Single Responsibility — each decorator does one thing
- ❌ Many small classes can be confusing
- ❌ Order of decorators matters and isn't always obvious

### Decorator vs Inheritance
- **Inheritance:** behavior fixed at compile time, one chain of parents.
- **Decorator:** behavior added at runtime, any combination — but only works through composition.

---

## 9. Facade Pattern (Structural)

### The Problem
You're building a feature that needs to coordinate **many subsystems**: validate input, check inventory, charge payment, generate invoice, send email, update analytics. The client code becomes a mess:

```java
// BAD — client must know every subsystem
class Checkout {
    void process(Cart cart, Customer customer) {
        Validator v = new Validator();
        v.validateCart(cart);
        v.validateCustomer(customer);

        InventoryService inv = new InventoryService();
        if (!inv.reserve(cart.items())) throw ...;

        PaymentService pay = new PaymentService();
        String txn = pay.charge(customer.card(), cart.total());

        InvoiceService invoice = new InvoiceService();
        invoice.generate(customer, cart, txn);

        EmailService email = new EmailService();
        email.sendConfirmation(customer.email(), txn);

        AnalyticsService analytics = new AnalyticsService();
        analytics.trackPurchase(customer, cart);
    }
}
```

**Problems:**
- Client has to know every subsystem and the right call order.
- Any change in subsystems forces edits to every client.
- Coupling everywhere — testing is hard.

### The Solution
Build a **Facade** — a single class that provides a simple, high-level interface to a complex subsystem. The client talks only to the facade; the facade orchestrates the rest.

### Key Players
| Role | What it does |
|---|---|
| **Facade** | A single, simplified entry point that delegates to subsystem classes |
| **Subsystem classes** | The complex internals doing real work (unchanged by the facade) |
| **Client** | Uses only the Facade, oblivious to subsystem details |

### Code Example

```java
// 1. The complex subsystem — many specialized classes
class Validator {
    void validateCart(String cartId)        { System.out.println("Validating cart " + cartId); }
    void validateCustomer(String customerId){ System.out.println("Validating customer " + customerId); }
}
class InventoryService {
    boolean reserve(String cartId)          { System.out.println("Reserving inventory for " + cartId); return true; }
}
class PaymentService {
    String charge(String customerId, double amount) {
        System.out.println("Charging $" + amount + " to " + customerId);
        return "TXN-9001";
    }
}
class InvoiceService {
    void generate(String customerId, String txnId) {
        System.out.println("Invoice generated for " + customerId + " (" + txnId + ")");
    }
}
class EmailService {
    void sendConfirmation(String customerId, String txnId) {
        System.out.println("Confirmation email sent to " + customerId);
    }
}

// 2. The Facade — one simple method hides all the orchestration
class CheckoutFacade {
    private final Validator validator         = new Validator();
    private final InventoryService inventory  = new InventoryService();
    private final PaymentService payment      = new PaymentService();
    private final InvoiceService invoices     = new InvoiceService();
    private final EmailService emails         = new EmailService();

    public String checkout(String customerId, String cartId, double total) {
        validator.validateCustomer(customerId);
        validator.validateCart(cartId);

        if (!inventory.reserve(cartId)) {
            throw new IllegalStateException("Out of stock");
        }
        String txnId = payment.charge(customerId, total);
        invoices.generate(customerId, txnId);
        emails.sendConfirmation(customerId, txnId);
        return txnId;
    }
}

// 3. Client — clean, single call
public class Demo {
    public static void main(String[] args) {
        CheckoutFacade checkout = new CheckoutFacade();
        String txnId = checkout.checkout("CUST-42", "CART-7", 49.99);
        System.out.println("Done. Transaction: " + txnId);
    }
}
```

### Walk-through
1. The subsystem classes (`Validator`, `InventoryService`, etc.) stay independent and reusable.
2. The `CheckoutFacade` knows about all of them and orchestrates the right sequence.
3. The client just calls `checkout(...)` — it doesn't care about the steps.
4. If tomorrow you add a fraud-check step, you change only the facade. Client code is untouched.

### When to Use
- You have a complex subsystem with many interdependent classes.
- You want to provide a simple entry point for common use cases.
- You're integrating a messy/legacy library and want to give your team a clean API.

### Real-World Examples
- `javax.faces.context.FacesContext` in JSF.
- jQuery's `$.ajax()` — hides the complexity of `XMLHttpRequest`, headers, parsing, callbacks.
- A "BFF" (Backend-For-Frontend) service that calls 5 microservices and returns one tidy response.
- Spring's `JdbcTemplate` — facade over JDBC's verbose connection/statement/result-set dance.

### Pros and Cons
- ✅ Drastically simpler client code
- ✅ Decouples client from subsystem details
- ✅ Easy to refactor subsystem internals later
- ❌ Facade can become a "god class" if it grows too big — split it then

### Facade vs Adapter
- **Adapter:** changes the *interface* of one existing object so it fits.
- **Facade:** provides a *new, simpler interface* over a whole set of objects.

---

## Big Picture: Cheat Sheet

| Pattern | Type | One-Sentence Summary | Code Smell That Suggests It |
|---|---|---|---|
| **Factory** | Creational | Centralize object creation behind a method | `if/switch` followed by `new` |
| **Builder** | Creational | Fluently assemble a complex object step-by-step | Constructors with 4+ params |
| **Observer** | Behavioral | Notify many listeners when something happens | One method calling many unrelated services |
| **Strategy** | Behavioral | Swap algorithms at runtime via a common interface | Long `if/else` selecting an algorithm |
| **State** | Behavioral | Object behavior changes with its internal state | `switch(state)` repeated in many methods |
| **Adapter** | Structural | Wrap an incompatible interface into the one you need | "Their API doesn't match ours" |
| **Singleton** | Creational | Ensure exactly one instance with global access | Multiple `new Config()` calls scattered around |
| **Decorator** | Structural | Add behavior by wrapping objects of the same interface | Subclass explosion for feature combinations |
| **Facade** | Structural | Single simple interface over a complex subsystem | Client juggling many subsystem classes |

---

## How They Relate to SOLID Principles

| Principle | What it Means | Pattern That Helps |
|---|---|---|
| **S**ingle Responsibility | A class should do one thing | Builder, State, Strategy, Decorator |
| **O**pen/Closed | Open for extension, closed for modification | Factory, Strategy, Observer, State, Decorator |
| **L**iskov Substitution | Subtypes should work where the base is expected | All of them rely on this |
| **I**nterface Segregation | Don't force clients to depend on unused methods | Adapter, Facade |
| **D**ependency Inversion | Depend on abstractions, not concrete classes | All — every pattern uses interfaces |

> **Note on Singleton:** It often *violates* SRP (manages its own lifecycle + business logic) and DIP (clients depend on concrete class). Use it sparingly — prefer dependency injection.

---

## Tips for Beginners

1. **Don't force patterns where they don't fit.** Patterns solve specific problems. If you don't have the problem, the pattern adds complexity.
2. **Start with the simplest design.** Refactor *into* a pattern when you see the problem emerge — don't over-engineer up front.
3. **Recognize the symptom, not just the pattern.** Learn the smell ("long if/else", "many constructor params") more than the structure.
4. **Patterns combine.** A Builder may build objects produced by a Factory. An Observer's reaction might use a Strategy. Real systems use them together.
5. **Read existing code.** Open the JDK or Spring source — patterns are everywhere once you know what to look for.

---

## Next Steps

After mastering these nine, explore:
- **Template Method** — define an algorithm skeleton, let subclasses fill in steps
- **Chain of Responsibility** — pass a request along a chain of handlers
- **Command** — encapsulate a request as an object (great for undo/redo)
- **Proxy** — a stand-in object that controls access to another (lazy loading, caching, security)
- **Composite** — treat individual objects and groups uniformly (tree structures)
- **Abstract Factory** — a factory of factories (families of related products)
