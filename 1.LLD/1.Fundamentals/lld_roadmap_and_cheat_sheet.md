# 🗺️ Master Low-Level Design (LLD) Roadmap & Complete Reference Guide

> **Core Philosophy:** *Low-Level Design (LLD) is about writing clean, maintainable, extensible, and testable code using good object-oriented design principles, robust UML modeling, and battle-tested design patterns.*

---

## 📌 Visual Infographics & Reference Charts
![LLD Mindmap Overview](file:///Users/ganeshpirikirala/Desktop/LLD/assets/lld_mindmap_reference.jpg)
*Figure 1: Complete LLD Mindmap Overview (OOP, SOLID, UML, Design Patterns, 9-Step Process, and Case Studies)*

---

![LLD Comprehensive Cheat Sheet](file:///Users/ganeshpirikirala/Desktop/LLD/assets/lld_cheat_sheet_reference.jpg)
*Figure 2: Master LLD Cheat Sheet (Interview Flow, Pattern Mapping, Checklists, and the Golden LLD Formula)*

---

## 📌 Table of Contents
1. [The Primary Goal of LLD](#1-the-primary-goal-of-lld)
2. [The 9-Step LLD Interview Execution Process](#2-the-9-step-lld-interview-execution-process)
3. [The Golden LLD Formula](#3-the-golden-lld-formula)
4. [Core OOP Principles & Concepts](#4-core-oop-principles--concepts)
5. [Class Relationships & Multiplicity Quick Reference](#5-class-relationships--multiplicity-quick-reference)
6. [Interface vs Abstract Class & Inheritance vs Composition](#6-interface-vs-abstract-class--inheritance-vs-composition)
7. [High Cohesion vs Low Coupling](#7-high-cohesion-vs-low-coupling)
8. [Dependency Injection & Inversion of Control (IoC)](#8-dependency-injection--inversion-of-control-ioc)
9. [Pattern-to-Problem Mapping Matrix](#9-pattern-to-problem-mapping-matrix)
10. [The Ultimate LLD Class Design Checklist](#10-the-ultimate-lld-class-design-checklist)
11. [Curated LLD Case Studies & Practice Track](#11-curated-lld-case-studies--practice-track)
12. [Recommended Design & Diagramming Tools](#12-recommended-design--diagramming-tools)

---

## 1. The Primary Goal of LLD

The objective of Low-Level Design is to **design the internal structure of a software system**.
While High-Level Design (HLD) focuses on servers, microservices, caches, and databases, LLD zooms in on:
* **Classes & Objects:** Data encapsulation, state management, and clear boundaries.
* **Methods & Signatures:** Parameter definitions, return types, error handling, and contracts.
* **Interactions & Message Passing:** How objects collaborate at runtime via composition, interfaces, and patterns.
* **Non-Functional Attributes:** Modularity, extensibility, testability, and adherence to SOLID principles.

---

## 2. The 9-Step LLD Interview Execution Process

When presented with an LLD problem (e.g., *"Design a Movie Ticket Booking System like BookMyShow"*), execute these 9 steps sequentially:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Understand Requirements & Constraints                    │
│    Clarify functional scope, users, edge cases, capacities. │
├─────────────────────────────────────────────────────────────┤
│ 2. Identify Core Entities (Classes & Objects)               │
│    Extract nouns from requirements (User, Movie, Show, Seat)│
├─────────────────────────────────────────────────────────────┤
│ 3. Identify Relationships                                   │
│    Determine Association, Aggregation, Composition, Is-A.   │
├─────────────────────────────────────────────────────────────┤
│ 4. Draw the UML Class Diagram                               │
│    Visualize interfaces, visibility (+, -), and inheritance.│
├─────────────────────────────────────────────────────────────┤
│ 5. Define Attributes & Method Signatures                    │
│    Specify fields, types, and primary public operations.    │
├─────────────────────────────────────────────────────────────┤
│ 6. Apply SOLID Principles                                   │
│    Check SRP, decouple low-level modules, eliminate fat API.│
├─────────────────────────────────────────────────────────────┤
│ 7. Identify & Apply Design Patterns                         │
│    Choose Strategy, State, Observer, Factory, or Decorator. │
├─────────────────────────────────────────────────────────────┤
│ 8. Handle Concurrency & Edge Cases                          │
│    Race conditions (double-booking seats), payment timeout. │
├─────────────────────────────────────────────────────────────┤
│ 9. Review for Extensibility, Modularity & Testing           │
│    Verify if adding new payment/seat types breaks old code. │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. The Golden LLD Formula

In any design round, follow this exact workflow:

$$\text{Requirements} \longrightarrow \text{Entities} \longrightarrow \text{Responsibilities (Methods)} \longrightarrow \text{Relationships} \longrightarrow \text{Interfaces}$$

$$\big\downarrow$$

$$\text{Composition / Polymorphism} \longrightarrow \text{SOLID} \longrightarrow \text{Design Patterns} \longrightarrow \text{UML Class Diagram} \longrightarrow \mathbf{Production\ Code}$$

---

## 4. Core OOP Principles & Concepts

| Concept | Definition | Real-World Software Example |
| :--- | :--- | :--- |
| **Class** | Blueprint or template describing state and behavior. | `class Vehicle { ... }` |
| **Object** | Concrete instance of a class occupying memory. | `Vehicle tesla = new Vehicle();` |
| **Encapsulation** | Bundling data with methods operating on that data, hiding internal state. | Marking fields `private` and exposing public getters/business methods. |
| **Abstraction** | Hiding implementation details and showing only essential features. | `PaymentStrategy.pay()` hides banking networks, encryption, and protocols. |
| **Inheritance** | Reusing and extending state/behavior from a parent class ("Is-A"). | `ElectricCar extends Car` |
| **Polymorphism** | "Many forms": same interface/method behaving differently at runtime. | Method Overloading (Compile-time) & Method Overriding (Runtime Dynamic Dispatch). |

### Types of Polymorphism
* **Compile-Time (Static):** Method Overloading (`search(String title)` vs `search(String title, Genre genre)`).
* **Runtime (Dynamic):** Method Overriding via base classes or interfaces (`paymentStrategy.pay(amount)` dynamically executing `UpiPayment.pay()` vs `CardPayment.pay()`).

---

## 5. Class Relationships & Multiplicity Quick Reference

```
┌─────────────┬──────────────────────────────────────────────────────────┬────────────────────────┐
│ Relationship│ UML Symbol & Representation                              │ Lifecycle Dependency   │
├─────────────┼──────────────────────────────────────────────────────────┼────────────────────────┤
│ Dependency  │ ClassA - - - - - - - - - ► ClassB (Uses-A)               │ Temporary parameter/var│
│ Association │ ClassA ──────────────────► ClassB (Has-A)                │ Independent existences │
│ Aggregation │ ClassA ◇───────────────── ClassB (Weak Has-A)            │ Child survives parent  │
│ Composition │ ClassA ◆───────────────── ClassB (Strong Part-Of)        │ Child dies with parent │
│ Inheritance │ ClassA ──────────────────▷ ClassB (Is-A)                 │ Tight compile-time link│
│ Realization │ ClassA - - - - - - - - - ▷ InterfaceB (Implements)       │ Contract adherence     │
└─────────────┴──────────────────────────────────────────────────────────┴────────────────────────┘
```

### The Life-Cycle Test: Aggregation vs Composition
* **Aggregation (`◇`):** A `Department` has `Professors`. If the Department is closed, the Professors still exist in the university.
* **Composition (`◆`):** A `Car` has an `Engine` and a `Building` has `Rooms`. If the Car/Building is destroyed, the specific Engine assembly / Rooms cease to exist as part of that entity.

---

## 6. Interface vs Abstract Class & Inheritance vs Composition

### Interface vs Abstract Class
* **Use an Interface when:** You want to define a strict **contract** (capabilities) without any state or shared implementation across unrelated classes (e.g. `Payable`, `Serializable`, `Cloneable`).
* **Use an Abstract Class when:** You want to provide **common shared state + default behavior** across closely related classes in an inheritance tree (e.g. `abstract class Vehicle { protected Engine engine; void start() { ... } }`).

### Composition Over Inheritance ("Favor HAS-A over IS-A")
* **Why Inheritance can be Brittle:** Subclasses are tightly coupled to the superclass. A change in the parent class ripples through all subclasses. You cannot change a parent class behavior at runtime.
* **Why Composition Wins:** Composing objects with pluggable interface components allows swapping behaviors dynamically at runtime via setters or constructor injection:
  ```java
  // Inheritance (Rigid):
  class Car extends CombustionEngine { ... }

  // Composition (Flexible & Pluggable):
  class Car {
      private Engine engine; // Can be swapped to ElectricEngine dynamically!
      public Car(Engine engine) { this.engine = engine; }
  }
  ```

---

## 7. High Cohesion vs Low Coupling

```
HIGH COHESION (GOOD)                    LOW COUPLING (GOOD)
┌──────────────────────────────┐         ┌──────────────┐         ┌───────────────────────────┐
│        PaymentService        │         │ OrderService │ ──────► │ <<interface>> PaymentGate │
├──────────────────────────────┤         └──────────────┘         └─────────────▲─────────────┘
│ + processPayment()           │                                                │
│ + refund()                   │                                                ┆
│ + validateCard()             │                                  ┌─────────────┴─────────────┐
└──────────────────────────────┘                                  ▼                           ▼
All methods relate directly to                               ┌─────────┐                 ┌─────────┐
one domain: Payments!                                        │ Stripe  │                 │ PayPal  │
                                                             └─────────┘                 └─────────┘
                                                 OrderService does not know about Stripe or PayPal!
```

* **High Cohesion:** A class has focused, closely related responsibilities. A class shouldn't handle payments, write database logs, and send emails simultaneously.
* **Low Coupling:** Classes depend on abstractions (interfaces) rather than concrete implementations. Changing the database or payment provider doesn't impact business services.

---

## 8. Dependency Injection & Inversion of Control (IoC)

### ❌ The Tightly Coupled Anti-Pattern
```java
public class OrderService {
    // Tightly coupled to a specific implementation!
    private PaymentService payment = new StripePaymentService(); 
}
```

### ✅ Clean Architecture with Dependency Injection
```java
public class OrderService {
    private final PaymentService payment;

    // Dependency is injected from outside (Constructor Injection)
    public OrderService(PaymentService payment) {
        this.payment = payment;
    }
}
```

### Core Benefits of DI:
1. **Loose Coupling:** Easily switch between `Stripe`, `Razorpay`, and `PayPal`.
2. **Effortless Unit Testing:** Pass mock or stub objects (`MockPaymentService`) during testing without network calls.
3. **Flexibility & Configuration:** Wire beans dynamically using Spring or Guice.

---

## 9. Pattern-to-Problem Mapping Matrix

| Problem Scenario / Feature | Recommended Design Pattern | Why It Fits |
| :--- | :--- | :--- |
| **Payment Gateways (UPI, Card, NetBanking)** | **Strategy + Factory** | Strategy encapsulates algorithms; Factory instantiates the selected strategy. |
| **Notifications (Email, SMS, Push, Webhook)** | **Observer / Strategy** | Broadcast updates to multiple listeners without tight coupling. |
| **Parking Lot / Vehicle Allocation** | **Factory + Strategy** | Factory creates spots/tickets; Strategy selects optimal parking spot. |
| **Vending Machine State Lifecycle** | **State Pattern** | Object behavior changes as states transition (`NoCoin` ──► `HasCoin` ──► `Sold`). |
| **ATM Cash Dispenser ($100, $50, $20, $10 bills)**| **Chain of Responsibility** | Request flows down denominations until requested cash amount is fulfilled. |
| **Centralized App Logging / Connection Pool**| **Singleton + Chain of Resp.** | 1 shared instance across threads; levels pass through severity filters. |
| **Coffee Machine with Custom Condiments** | **Decorator Pattern** | Dynamically stack Milk, Caramel, Mocha without combinatorial subclassing. |
| **Chess / Board Game Character Movements** | **Strategy + Command** | Strategy validates pieces movement; Command executes & records moves for Undo. |
| **Elevator Control Management** | **State + Strategy** | State tracks `MovingUp`, `Idle`, `MovingDown`; Strategy schedules floor stops. |
| **Text Editor / Document Formatting** | **Command + Memento** | Command handles actions (`Insert`, `Delete`); Memento stores undo checkpoints. |
| **File Storage / Directory Explorer** | **Composite Pattern** | Treat individual files and nested folders uniformly. |
| **Ride Sharing (Driver Matching & Pricing)** | **Strategy + Observer** | Strategy matches nearest/cheapest drivers; Observer alerts riders & drivers. |
| **Legacy Bank API Integration** | **Adapter Pattern** | Translates modern JSON payload into legacy SOAP/XML gateway format. |
| **Media Player / Smart Home One-Touch Play** | **Facade Pattern** | 1 unified method coordinates 5 complex backend hardware subsystems. |
| **Access Control & Video Streaming Lazy Load**| **Proxy Pattern** | Validates user subscription and lazy-loads heavy video streams on demand. |

---

## 10. The Ultimate LLD Class Design Checklist

Before writing code in an interview or review, evaluate your design against this checklist:

* [ ] **Entities Identified:** Have all key nouns been modeled as classes?
* [ ] **Single Responsibility:** Does each class have only one reason to change?
* [ ] **Visibility Enforced:** Are all fields `private`, with access controlled through methods?
* [ ] **Interfaces Defined:** Are consumers programming to interfaces rather than concrete classes?
* [ ] **Inheritance vs Composition:** Are you using Composition (`HAS-A`) rather than rigid multi-level inheritance?
* [ ] **Design Patterns Justified:** Does each pattern solve a concrete problem rather than adding unnecessary complexity?
* [ ] **Concurrency & Thread-Safety:** Are shared singletons thread-safe (e.g. Double-Checked Locking / volatile)?
* [ ] **Edge Cases Covered:** Are invalid inputs, null references, and capacity limits validated?
* [ ] **Testability:** Can every component be isolated and unit-tested with mock dependencies?

---

## 11. Curated LLD Case Studies & Practice Track

To master Low-Level Design, practice these standard interview problems categorized by complexity:

### 🟢 Beginner to Intermediate
1. **Parking Lot System** (Slot sizing, ticket calculation, entry/exit gates)
2. **Tic-Tac-Toe / Snake & Ladder** (Board modeling, players, winning condition algorithms)
3. **Vending Machine** (State pattern, coin handling, inventory tracking)
4. **Elevator System** (Request dispatching algorithms, multi-elevator controller)
5. **Library Management System** (Book lending, reservations, barcode indexing, fines)
6. **ATM System** (PIN verification, account balances, Chain of Responsibility cash dispensing)
7. **Splitwise / Expense Sharing** (Equal, exact, and percentage splits, debt simplification)
8. **Car Rental System** (Vehicle reservations, store inventory, pickup/return workflows)

### 🔴 Advanced & Distributed LLD
9. **Movie Ticket Booking System (BookMyShow)** (Seat locking, concurrency, payment callbacks)
10. **Ride Sharing Platform (Uber / Lyft)** (Location tracking, pricing strategies, driver dispatch)
11. **Food Delivery App (Swiggy / Zomato)** (Restaurant menus, cart checkout, delivery assignment)
12. **Notification Engine** (Multi-channel routing, priority queues, rate-limiting)
13. **Centralized Logging Framework (Log4j mini)** (Sinks, async appenders, log levels)
14. **Distributed In-Memory Cache (Redis mini)** (Eviction policies: LRU, LFU, TTL expiration)
15. **Pub/Sub Messaging System (Kafka mini)** (Topics, consumer offsets, message acknowledgments)
16. **Rate Limiter** (Token Bucket, Leaky Bucket, Sliding Window Counter algorithms)

---

## 12. Recommended Design & Diagramming Tools

When preparing diagrams for production systems or architecture documentation:
* **Interactive Diagramming:** [Draw.io / diagrams.net](https://app.diagrams.net), [Lucidchart](https://www.lucidchart.com), [Excalidraw](https://excalidraw.com).
* **Code-to-Diagram (Diagrams as Code):** [PlantUML](https://plantuml.com), [Mermaid.js](https://mermaid.js.org).
* **IDEs & Modeling Plugins:** IntelliJ IDEA UML Diagram Support, Visual Studio Code (PlantUML / Mermaid Preview).
