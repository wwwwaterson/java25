
# Java 25 (JDK 25 – LTS) Manual

A single-file manual with **all code examples embedded in this README**. Copy/paste the snippets below to try the new features of **Java 25 (LTS)**.

> Tip: This README is English-first for broader GitHub reach. If you want a **PT-BR** version, I can generate it too.

---

## Table of Contents
- [Overview](#overview)
- [Requirements](#requirements)
- [Quick Start](#quick-start)
- [Language Features](#language-features)
  - [Primitive Types in Patterns (JEP 507 – 3rd Preview)](#primitive-types-in-patterns-jep-507--3rd-preview)
  - [Module Import Declarations (JEP 511)](#module-import-declarations-jep-511)
  - [Compact Source Files & Instance `main` (JEP 512)](#compact-source-files--instance-main-jep-512)
  - [Flexible Constructor Bodies (JEP 513)](#flexible-constructor-bodies-jep-513)
- [Concurrency](#concurrency)
  - [Scoped Values (JEP 506 – Final)](#scoped-values-jep-506--final)
  - [Structured Concurrency (JEP 505 – 5th Preview)](#structured-concurrency-jep-505--5th-preview)
- [Performance & Startup](#performance--startup)
  - [AOT Command-Line Ergonomics (JEP 514)](#aot-command-line-ergonomics-jep-514)
  - [AOT Method Profiling (JEP 515)](#aot-method-profiling-jep-515)
  - [Compact Object Headers (JEP 519)](#compact-object-headers-jep-519)
- [Monitoring with JFR](#monitoring-with-jfr)
  - [CPU-Time Profiling (JEP 509 – Experimental)](#cpu-time-profiling-jep-509--experimental)
  - [Cooperative Sampling (JEP 518)](#cooperative-sampling-jep-518)
  - [Method Timing & Tracing (JEP 520)](#method-timing--tracing-jep-520)
- [Security & Cryptography](#security--cryptography)
  - [PEM Encodings (JEP 470)](#pem-encodings-jep-470)
  - [Key Derivation Function API (JEP 510)](#key-derivation-function-api-jep-510)
- [JVM & GC Changes](#jvm--gc-changes)
- [Migration Guide](#migration-guide)
- [References](#references)

---

## Overview
**Java 25** is a **Long-Term Support (LTS)** release (Sept 16, 2025) bringing 18 features across language, APIs, performance, monitoring, and security.

---

## Requirements
- **JDK 25** (LTS). Download builds at: https://jdk.java.net/25/
- Some examples use **preview** features. Compile/run with `--enable-preview`.
- Use `--release 25` (or `--source 25` / `--target 25`) if your toolchain requires explicit level.

> Windows line-endings tip: avoid warnings using `git config core.autocrlf true`.

---

## Quick Start
#### Windows PowerShell
```powershell
# Example (preview) compile/run
javac --enable-preview --release 25 PrimitivePatterns.java
java  --enable-preview PrimitivePatterns
```

#### macOS/Linux (bash)
```bash
javac --enable-preview --release 25 PrimitivePatterns.java
java  --enable-preview PrimitivePatterns
```

---

## Language Features

### Primitive Types in Patterns (JEP 507 – 3rd Preview)
> Use primitives (`int`, `long`, `double`, etc.) in `switch` and `instanceof` pattern matching.

**File:** `PrimitivePatterns.java`
```java
// JEP 507 (3rd preview): primitive types in patterns, instanceof, and switch
public class PrimitivePatterns {
    static String describe(Object o) {
        return switch (o) {
            case int i     -> "int:" + i;
            case long l    -> "long:" + l;
            case double d  -> "double:" + d;
            case null      -> "null";
            default        -> "other:" + o.getClass().getSimpleName();
        };
    }
    public static void main(String[] args) {
        System.out.println(describe(42));
        System.out.println(describe(3.14));
        System.out.println(describe(1L));
        System.out.println(describe("text"));
        System.out.println(describe(null));
    }
}
```
**Run:**
```bash
javac --enable-preview --release 25 PrimitivePatterns.java
java  --enable-preview PrimitivePatterns
```

---

### Module Import Declarations (JEP 511)
> Import all packages exported by a module using `import module`.

**File:** `ModuleImportDemo.java`
```java
// JEP 511: Module import declarations
import module java.base;  // exports java.util.* including java.util.Date
import module java.sql;   // exports java.sql.* including java.sql.Date

import java.sql.Date;     // disambiguate

public class ModuleImportDemo {
    public static void main(String[] args) {
        Date d = Date.valueOf("2025-06-15");
        System.out.println("Resolved java.sql.Date: " + d);
    }
}
```
**Run:**
```bash
javac ModuleImportDemo.java && java ModuleImportDemo
```

---

### Compact Source Files & Instance `main` (JEP 512)
> Write quick scripts without a class declaration; `void main()` at top-level.

**File:** `HelloCompact.java`
```java
// JEP 512: Compact source file + instance main method (no class declaration required)
void main() {
    System.out.println("Hello from Java 25 compact source file!");
}
```
**Run:**
```bash
java HelloCompact.java
```

---

### Flexible Constructor Bodies (JEP 513)
> Run validation before calling `super()` or `this()` (cannot reference the not-yet-constructed object).

**File:** `FlexibleConstructorDemo.java`
```java
// JEP 513: Flexible constructor bodies
class Person {
    final int age;
    Person(int age) { this.age = age; }
}
class Employee extends Person {
    final String name;
    Employee(String name, int age) {
        // Pre-validation before calling super(...)
        if (age < 18 || age > 67) throw new IllegalArgumentException("Age must be between 18 and 67");
        super(age); // not required as the first statement anymore
        this.name = name;
    }
}
public class FlexibleConstructorDemo {
    public static void main(String[] args) {
        var e = new Employee("Alice", 35);
        System.out.println("Employee OK: " + e.name + ", age=" + e.age);
        try {
            new Employee("Bob", 10);
        } catch (IllegalArgumentException ex) {
            System.out.println("Validation works: " + ex.getMessage());
        }
    }
}
```
**Run:**
```bash
javac FlexibleConstructorDemo.java && java FlexibleConstructorDemo
```

---

## Concurrency

### Scoped Values (JEP 506 – Final)
> Immutable, thread-safe context propagation—ideal with **Virtual Threads**.

**File:** `ScopedValuesDemo.java`
```java
// JEP 506: Scoped Values (finalized in JDK 25)
import java.lang.ScopedValue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class ScopedValuesDemo {
    static final ScopedValue<String> USER = ScopedValue.newInstance();

    public static void main(String[] args) throws Exception {
        try (ExecutorService exec = Executors.newVirtualThreadPerTaskExecutor()) {
            exec.submit(() -> ScopedValue.where(USER, "Alice").run(() -> {
                System.out.println("VT: " + Thread.currentThread() + " USER=" + USER.get());
            }));
            exec.submit(() -> ScopedValue.where(USER, "Bob").run(() -> {
                System.out.println("VT: " + Thread.currentThread() + " USER=" + USER.get());
            }));
            Thread.sleep(200);
        }
    }
}
```
**Run:**
```bash
javac ScopedValuesDemo.java && java ScopedValuesDemo
```

---

### Structured Concurrency (JEP 505 – 5th Preview)
> Treat related tasks as a single unit; simpler lifecycle, error handling, and cancellation.

**File:** `StructuredConcurrencyDemo.java`
```java
// JEP 505 (5th preview): Structured Concurrency
import java.util.concurrent.StructuredTaskScope;

public class StructuredConcurrencyDemo {
    static String fetchUser() throws InterruptedException { Thread.sleep(100); return "Alice"; }
    static String fetchOrder() throws InterruptedException { Thread.sleep(150); return "Order#42"; }

    public static void main(String[] args) throws Exception {
        try (var scope = StructuredTaskScope.<String>open()) {
            var userTask = scope.fork(StructuredConcurrencyDemo::fetchUser);
            var orderTask = scope.fork(StructuredConcurrencyDemo::fetchOrder);
            scope.join();
            System.out.println(userTask.get() + " - " + orderTask.get());
        }
    }
}
```
**Run:**
```bash
javac --enable-preview --release 25 StructuredConcurrencyDemo.java
java  --enable-preview StructuredConcurrencyDemo
```

---

## Performance & Startup

### HttpClient Limiting (New in JDK 25)
> Limit accepted response bytes with `BodyHandlers.limiting(...)` / `BodySubscribers.limiting(...)`.

**File:** `HttpClientLimitDemo.java`
```java
// New in JDK 25: BodyHandlers.limiting / BodySubscribers.limiting
import java.net.URI;
import java.net.http.*;

public class HttpClientLimitDemo {
    public static void main(String[] args) throws Exception {
        var client = HttpClient.newHttpClient();
        var req = HttpRequest.newBuilder(URI.create("https://example.com")).GET().build();
        var handler = HttpResponse.BodyHandlers.limiting(HttpResponse.BodyHandlers.ofString(), 1024);
        var resp = client.send(req, handler);
        System.out.println("Status: " + resp.statusCode());
        System.out.println("Body (<=1KB):\n" + resp.body());
    }
}
```
**Run:**
```bash
javac HttpClientLimitDemo.java && java HttpClientLimitDemo
```

---

### CharSequence#getChars (New in JDK 25)
> Bulk-read characters from `CharSequence` into a `char[]`.

**File:** `CharSequenceGetCharsDemo.java`
```java
// New in JDK 25: CharSequence#getChars(int, int, char[], int)
public class CharSequenceGetCharsDemo {
    public static void main(String[] args) {
        CharSequence cs = "Java 25 – getChars demo";
        char[] buf = new char[8];
        cs.getChars(5, 13, buf, 0); // copies "25 – get"
        System.out.println(new String(buf));
    }
}
```
**Run:**
```bash
javac CharSequenceGetCharsDemo.java && java CharSequenceGetCharsDemo
```

---

### ForkJoinPool as ScheduledExecutorService (JDK 25)
> `ForkJoinPool` now implements `ScheduledExecutorService` for delayed tasks/timeouts.

**File:** `ForkJoinScheduledDemo.java`
```java
// JDK 25: ForkJoinPool implements ScheduledExecutorService
import java.util.concurrent.*;

public class ForkJoinScheduledDemo {
    public static void main(String[] args) throws Exception {
        ForkJoinPool pool = ForkJoinPool.commonPool();
        ScheduledExecutorService ses = pool; // valid in JDK 25
        var future = ses.schedule(() -> System.out.println("Delayed on FJP!"), 200, TimeUnit.MILLISECONDS);
        future.get();
    }
}
```
**Run:**
```bash
javac ForkJoinScheduledDemo.java && java ForkJoinScheduledDemo
```

---

## Security & Cryptography

### PEM Encodings (JEP 470)
> Read/write cryptographic objects in PEM format via standard APIs (keys, certs, CRLs).

> See references for API details.

### Key Derivation Function API (JEP 510)
> Access KDFs (e.g., HKDF, Argon2) from standard API; usable in KEMs and protocols.

> See references for API details.

---

## Monitoring with JFR

### CPU-Time Profiling (JEP 509 – Experimental)
Enhances JFR to capture CPU-time profiles on Linux using the kernel timer.

### Cooperative Sampling (JEP 518)
Improves stability by sampling only at safepoints while minimizing bias.

### Method Timing & Tracing (JEP 520)
Record exact statistics for selected methods without source modification.

> Configure via CLI, `jcmd`, or JMX. See references.

---

## JVM & GC Changes
- **Generational Shenandoah GC** promoted to product feature (better throughput/resilience/memory).
- **Removal of x86 32-bit port**.
- Additional core-libs/networking/security improvements (see Release Notes).

---

## Migration Guide
1. **Dependencies**: check build plugins/agents and bytecode tools for 25-compat.
2. **Preview flags**: add `--enable-preview` for JEP 505/507 when compiling **and** running.
3. **Profiling**: compare JDK 21 vs 25 using JFR Method Timing & CPU-Time Profiling.
4. **GC evaluation**: benchmark Shenandoah vs G1/ZGC on prod-like workloads.
5. **Security**: migrate PEM/KDF handling to the new APIs when applicable.

---

## References
- **JDK 25 Release Notes & JEP list**: https://jdk.java.net/25/release-notes
- **OpenJDK Project JDK 25** (status/schedule): https://openjdk.org/projects/jdk/25/
- **InfoWorld – What’s new in JDK 25**: https://www.infoworld.com/article/3846172/jdk-25-the-new-features-in-java-25.html
- **Baeldung – New Features in Java 25**: https://www.baeldung.com/java-25-features
- **Java Almanac (JDK 25)**: https://javaalmanac.io/jdk/25/

---



