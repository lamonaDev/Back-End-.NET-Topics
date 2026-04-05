# OOP in C# - Professional Documentation

A comprehensive guide to Object-Oriented Programming concepts in C#, extracted and enhanced from the original research materials.

---

## 📚 Table of Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | [Encapsulation](01-encapsulation.md) | Data hiding and controlled access |
| 02 | [Abstraction](02-abstraction.md) | Hiding complexity, showing essentials |
| 03 | [Inheritance](03-inheritance.md) | IS-A relationships and code reuse |
| 04 | [Polymorphism](04-polymorphism.md) | Many forms, runtime dispatch |
| 05 | [Class Relationships](05-class-relationships.md) | Dependency, Association, Aggregation, Composition |
| 06 | [Interview Questions](06-interview-questions.md) | Common OOP interview Q&A |

---

## 🎯 Quick Reference

### The Four Pillars of OOP

```
┌─────────────────────────────────────────────────────────────────────┐
│                    FOUR PILLARS OF OOP                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   ENCAPSULATION              ABSTRACTION                             │
│   ─────────────              ──────────                              │
│   Hiding data and            Hiding complexity                       │
│   implementation details     Implementation details                  │
│                                                                      │
│   Example:                   Example:                                │
│   private fields with        Abstract classes &                      │
│   public getters/setters     interfaces                              │
│                                                                      │
│   INHERITANCE                POLYMORPHISM                            │
│   ───────────                ───────────                             │
│   IS-A relationship          Many forms                              │
│   Code reuse                 Same call, different                    │
│                              behavior                                │
│                                                                      │
│   Example:                   Example:                                │
│   class Dog : Animal         animal.MakeSound()                      │
│                              ├─ Dog: "Woof!"                         │
│                              ├─ Cat: "Meow!"                         │
│                              └─ Bird: "Tweet!"                       │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### Class Relationships Quick Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│                    CLASS RELATIONSHIPS                                │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│   Weakest ──────────────────────────────────────────────▶ Strongest │
│                                                                      │
│   DEPENDENCY    ASSOCIATION    AGGREGATION    COMPOSITION            │
│   ──────────    ──────────    ──────────      ───────────            │
│                                                                      │
│   "uses"        "has"         "groups"        "owns"                │
│   temporarily   reference    (weak)          (strong)                │
│                                                                      │
│   UML: -- -▷    UML: ───▷     UML: ◇───      UML: ◆───              │
│                                                                      │
│   Example:      Example:      Example:       Example:              │
│   Method        Order has      Team has        House has             │
│   parameter     Customer      Players        Rooms                 │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🚀 How to Use This Documentation

1. **Start with the basics**: Read files 01-04 in order to understand the four pillars
2. **Understand relationships**: File 05 explains how classes interact
3. **Practice**: Review interview questions in file 06
4. **Cross-reference**: Each file includes memory allocation diagrams and real-world analogies

---

## 💡 Key Concepts Summary

### Access Modifiers

| Modifier | Access Level |
|----------|--------------|
| `public` | Anywhere |
| `private` | Same class only |
| `protected` | Class + derived classes |
| `internal` | Same assembly |
| `protected internal` | Same assembly OR derived |
| `private protected` | Same assembly AND derived |

### Inheritance Keywords

| Keyword | Purpose |
|---------|---------|
| `virtual` | Can be overridden |
| `override` | Replaces base implementation |
| `abstract` | Must be implemented by derived |
| `sealed` | Prevents further override/inheritance |
| `new` | Hides base member (not polymorphic) |
| `base` | Access base class members |

### Method Binding

| Type | When | Mechanism |
|------|------|-----------|
| Static | Compile time | Method signature matching |
| Dynamic | Runtime | Virtual Method Table (vtable) |

---

## 📖 Interview Preparation

See [06-interview-questions.md](06-interview-questions.md) for:
- Conceptual questions
- Code-based questions
- Scenario-based questions
- Memory allocation questions
- Best practices and anti-patterns

---

## 🎓 Learning Path

```
Week 1: Core Concepts
├─ Day 1-2: Encapsulation + Abstraction
├─ Day 3-4: Inheritance
└─ Day 5-7: Polymorphism

Week 2: Relationships & Practice
├─ Day 1-2: Class Relationships
├─ Day 3-4: Interview Questions
└─ Day 5-7: Code Practice & Review
```

---

## 📄 File Structure

```
E:\comp_research_repo\OOP\mds\
├── README.md                          ← You are here
├── 01-encapsulation.md
├── 02-abstraction.md
├── 03-inheritance.md
├── 04-polymorphism.md
├── 05-class-relationships.md
└── 06-interview-questions.md
```

---

## 🔗 Additional Resources

- [C# Documentation](https://docs.microsoft.com/en-us/dotnet/csharp/)
- [.NET API Reference](https://docs.microsoft.com/en-us/dotnet/api/)
- [Design Patterns in C#](https://refactoring.guru/design-patterns/csharp)

---

*Documentation generated from original OOP research materials. Each concept includes real-world analogies, code examples, and memory allocation explanations.*
