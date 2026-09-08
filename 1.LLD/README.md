# 📚 Complete Low-Level Design (LLD) Master Library

A comprehensive, production-grade reference manual covering:
1. **Object-Oriented Fundamentals, SOLID Principles & UML Modeling**
2. **The 23 Gang of Four (GoF) Design Patterns** (Creational, Structural, Behavioral)
3. **Concurrency & Multi-Threading in LLD** (Thread-safety, Race Conditions, Deadlocks, Locks)

Crafted specifically for Software Engineering interviews, machine coding rounds, and clean architecture education.

> 🚀 **Looking for System Design / High-Level Design?** Check out the companion [High-Level Design (HLD) Master Library](file:///Users/ganeshpirikirala/Desktop/LLD/HLD/README.md) covering Scalability, Caches, Sharding, Kafka, CAP Theorem, and Case Studies (TinyURL, Twitter, YouTube, WhatsApp)!

---

## 📁 Repository Directory Structure

```
LLD/
├── 📄 README.md
│
├── 📁 0.Assets/              (High-Res Architecture Mindmaps & Cheat Sheets)
│   ├── lld_mindmap_reference.jpg
│   └── lld_cheat_sheet_reference.jpg
│
├── 📁 1.Fundamentals/        (UML, SOLID, & Core Design Principles)
│   ├── lld_roadmap_and_cheat_sheet.md
│   ├── uml_diagrams_guide.md
│   ├── solid_principles_guide.md
│   └── design_principles_guide.md
│
├── 📁 2.Creational/          (5 Patterns - Object Creation Mechanisms)
│   ├── singleton_pattern_guide.md
│   ├── factory_method_pattern_guide.md
│   ├── abstract_factory_pattern_guide.md
│   ├── builder_pattern_guide.md
│   └── prototype_pattern_guide.md
│
├── 📁 3.Behavioral/          (11 Patterns - Communication & Responsibility Flow)
│   ├── strategy_pattern_guide.md
│   ├── observer_pattern_guide.md
│   ├── command_pattern_guide.md
│   ├── state_pattern_guide.md
│   ├── chain_of_responsibility_pattern_guide.md
│   ├── template_method_pattern_guide.md
│   ├── iterator_pattern_guide.md
│   ├── mediator_pattern_guide.md
│   ├── memento_pattern_guide.md
│   ├── visitor_pattern_guide.md
│   └── interpreter_pattern_guide.md
│
├── 📁 4.Structural/          (7 Patterns - Object Composition & Class Structures)
│   ├── adapter_pattern_guide.md
│   ├── bridge_pattern_guide.md
│   ├── composite_pattern_guide.md
│   ├── decorator_pattern_guide.md
│   ├── facade_pattern_guide.md
│   ├── flyweight_pattern_guide.md
│   └── proxy_pattern_guide.md
│
└── 📁 5.Concurrency-In-LLD/  (Thread-Safety, Locks, Race Conditions & Case Studies)
    └── concurrency_in_lld_guide.md
```

---

## ⚡ 1. Concurrency & Multi-Threading in LLD

| Topic | Focus Area | Key Takeaways | Link |
| :--- | :--- | :--- | :--- |
| **Concurrency in LLD ⚡** | Race Conditions & Locks | JVM Memory Model, `volatile`, `synchronized` vs `ReentrantLock`, `ReadWriteLock`, `ConcurrentHashMap`, CAS atomics, Producer-Consumer, Deadlock prevention, and a runnable Movie Ticket Booking Case Study. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/5.Concurrency-In-LLD/concurrency_in_lld_guide.md) |

---

## 🧠 2. Fundamentals, Modeling & Principles

| Topic | Focus Area | Key Takeaways | Link |
| :--- | :--- | :--- | :--- |
| **LLD Roadmap & Cheat Sheet 🗺️** | Master Architecture Blueprint | Infographics, 9-Step Interview Process, The Golden Formula, Pattern-to-Problem Mapping, Practice Case Studies. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/1.Fundamentals/lld_roadmap_and_cheat_sheet.md) |
| **UML Diagrams** | Visual Software Blueprints | Class Anatomy, 6 Core Relationships (Dependency ──► Association ──► Aggregation ──► Composition ──► Inheritance), Sequence Diagrams, Multiplicity. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/1.Fundamentals/uml_diagrams_guide.md) |
| **SOLID Principles** | Object-Oriented Architecture | S (Single Responsibility), O (Open/Closed), L (Liskov Substitution), I (Interface Segregation), D (Dependency Inversion) with full Java refactorings. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/1.Fundamentals/solid_principles_guide.md) |
| **Core Design Principles** | Foundational Engineering Wisdom | DRY (Single source of truth), KISS (Avoid overengineering), YAGNI, Composition Over Inheritance, Law of Demeter (Train wreck code), SoC. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/1.Fundamentals/design_principles_guide.md) |

---

## 🏗️ 3. Creational Design Patterns (5 Patterns)

| # | Pattern | Unique Domain Scenario | Core Takeaway | Link |
| :-: | :--- | :--- | :--- | :--- |
| **1** | **Singleton** | Centralized Distributed Config Manager ⚙️ | Guarantee 1 instance, double-checked locking, thread-safe enum protection. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/2.Creational/singleton_pattern_guide.md) |
| **2** | **Factory Method** | Document / Report Exporter (PDF/Excel/CSV) 📄 | Let subclasses decide which concrete document to instantiate. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/2.Creational/factory_method_pattern_guide.md) |
| **3** | **Abstract Factory** | Multi-Cloud Infrastructure Provisioner ☁️ | Create whole cohesive families (AWS EC2+S3 vs GCP Compute+Storage). | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/2.Creational/abstract_factory_pattern_guide.md) |
| **4** | **Builder** | High-End Custom Gaming PC Rig 🖥️ | Solve 10-parameter telescoping constructors with fluent method chaining. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/2.Creational/builder_pattern_guide.md) |
| **5** | **Prototype** | RTS Game Battlefield Unit Spawner 🎮 | Clone deep copies of expensive 3D archetypes in microseconds. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/2.Creational/prototype_pattern_guide.md) |

---

## 🧠 4. Behavioral Design Patterns (11 Patterns)

| # | Pattern | Unique Domain Scenario | Core Takeaway | Link |
| :-: | :--- | :--- | :--- | :--- |
| **1** | **Strategy** | Multi-Option Payment Processing (UPI, Card) 💳 | Dynamic runtime algorithm swapping behind a common interface contract. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/strategy_pattern_guide.md) |
| **2** | **Observer** | YouTube Channel Subscriber Alerts 🔔 📹 | Event-driven push notifications; eliminates wasteful polling loops. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/observer_pattern_guide.md) |
| **3** | **Command** | Rich Text Editor Multi-Level Undo / Redo 📝 | Encapsulate operations into objects with `execute()` and `undo()` history stacks. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/command_pattern_guide.md) |
| **4** | **State** | Automated Vending Machine FSM 🥤 | Eliminate giant nested switch blocks; object alters behavior when state changes. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/state_pattern_guide.md) |
| **5** | **Chain of Resp.** | API Gateway Security Pipeline 🛡️ 🌐 | Sequential middleware filters: RateLimiter ──► Authenticator ──► Authorizer. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/chain_of_responsibility_pattern_guide.md) |
| **6** | **Template Method** | CI/CD Automated DevOps Build Pipeline 🚀 🛠️ | Fixed `final` skeleton in base class; subclasses customize specific steps. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/template_method_pattern_guide.md) |
| **7** | **Iterator** | Spotify Music Playlist Traversal 🎵 🎧 | Sequentially access collections without exposing their internal data structure. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/iterator_pattern_guide.md) |
| **8** | **Mediator** | Airport Air Traffic Control (ATC) Tower ✈️ 🛬 | Replace $O(N^2)$ mesh communication with a clean centralized star orchestrator. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/mediator_pattern_guide.md) |
| **9** | **Memento** | Dark Souls Game Checkpoint & Save State 🎮 💾 | Capture and restore internal state without breaking encapsulation. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/memento_pattern_guide.md) |
| **10**| **Visitor** | E-Commerce Cart Tax & Shipping Operations 🛒 📦 | Double dispatch: Add new operations without editing existing domain classes. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/visitor_pattern_guide.md) |
| **11**| **Interpreter** | FinTech Fraud Detection Rule Engine 💳 🔍 | Parse and evaluate dynamic boolean rule expressions via an AST tree. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/3.Behavioral/interpreter_pattern_guide.md) |

---

## 🧩 5. Structural Design Patterns (7 Patterns)

| # | Pattern | Unique Domain Scenario | Core Takeaway | Link |
| :-: | :--- | :--- | :--- | :--- |
| **1** | **Adapter** | JSON Gateway to Legacy Bank XML 💳 ↔️ 🏛️ | Bridge incompatible legacy/vendor SDK interfaces without editing them. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/4.Structural/adapter_pattern_guide.md) |
| **2** | **Bridge** | Universal Smart Remote & Multi-Brand TV 📺 📻 | Decouple abstraction from implementation, preventing $N \times M$ class explosion. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/4.Structural/bridge_pattern_guide.md) |
| **3** | **Composite** | Hierarchical File System (Files & Folders) 📁 📄 | Treat individual leaves and nested composite trees uniformly. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/4.Structural/composite_pattern_guide.md) |
| **4** | **Decorator** | Boutique Cafe Custom Coffee Billing ☕ | Stack Milk, Caramel, and Cream dynamically, avoiding 160 subclasses. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/4.Structural/decorator_pattern_guide.md) |
| **5** | **Facade** | Smart Home Cinema Automation 🍿 🎬 | 1-Click "Movie Mode" orchestrating 5 complex hardware subsystems. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/4.Structural/facade_pattern_guide.md) |
| **6** | **Flyweight** | Rendering 1,000,000 Forest Trees in a Game 🌲 🎮 | Share intrinsic immutable mesh/textures; pass $(X, Y)$ extrinsic context. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/4.Structural/flyweight_pattern_guide.md) |
| **7** | **Proxy** | 4K Video Streaming Access & Lazy Loading 🎬 🔒 | Virtual lazy initialization, security check, and CDN edge caching stub. | [Guide](file:///Users/ganeshpirikirala/Desktop/LLD/4.Structural/proxy_pattern_guide.md) |
