---
title: "Single Design Pattern"
tags:
- atom
- design-pattern
---
Reference:  
Topics:  

---

### Summary
Is a type of creational pattern that ensures a class has only one instance and provides global access
to this instance. The name comes from the fact that it restricts instantiation of a class to a single
object.

The pattern is commonly used in situations where a class should only have a single instance such as
managing database connections, logging, file management, printer spoolers, and providing a global
state for configuration used across the application.

### Implementation details
A singleton can be implemented by making the constructor of a class private and defining a static
method that creates a static instance of the class.

### Code example

```cpp
#include <iostream>

class Singleton {
private:
    // Private constructor so that no objects
    // can be created.
    Singleton() {
        std::cout << "Singleton instance created" << std::endl;
    }

    // Static member is initiated as null.
    static Singleton* instance;

public:
    // Static method to control access to the Singleton instance.
    static Singleton* getInstance() {
        if (!instance)
            instance = new Singleton;
        return instance;
    }
};

// Initializing pointer to null.
Singleton* Singleton::instance = nullptr;

int main() {
    // New instance will be created.
    Singleton* instance1 = Singleton::getInstance();
  
    // New instance will not be created.
    Singleton* instance2 = Singleton::getInstance();
}

```