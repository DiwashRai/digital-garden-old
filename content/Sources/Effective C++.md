---
title: "Effective C++"
tags:
- source
- textbook
enableTOC: true
openTOC: true
---

Author: [Scott Meyers](Authors/Scott%20Meyers.md)  
Topics: [Software Engineering](Topics/Software%20Engineering.md)  

---

## Ch1: Accustoming yourself to C++  

### **Item 1:** View C++ as a federation of languages  
C++ is better understood as a group of related sub-languages. These are:
- C
- Object-oriented C++
- Template C++
- The STL

This is important to keep in mind as different sublanguages have different effective strategies and different conventions.


### **Item 2:** Prefer consts, enums, and inlines to \#defines  
Rule could also be called "Prefer the compiler over the preprocessor".   

```cpp
// BAD
#define ASPECT_RATIO 1.653

// GOOD
const double AspectRation = 1.653;
const char* const authorName = "Scott Meyers";
const std::string authorName("Scott Meyers");
```

The preprocessor will blindly subsitute ASPECT_RATIO for 1.653 resulting in multple copies in the object code.   

```cpp
// BAD
#define CALL_WITH_MAX(a,b) f((a) > (b) ? (a) : (b))

// GOOD
template<typename T>
inline void callWithMax(const T& a, const T& b)
{
    f(a > b ? a: b);
}
```
Function like macros can be very error prone. Calling `CALL_WITH_MAX(++a, b)` for example results in a being incremented twice.


### **Item 3:** Use const whenever possible  
- Helps detect usage errors.
- Using `const` in function declaration is extremely powerful so make use of it. It can refer to the function's return value, the individual parameters or even the function as a whole.
- Compilers enforce 'bitwise constness' but you should program using 'logical constness'.
	- Bitwise constness/Physical: does not modify any of object's data members
	- Logical constness: Some data members of an object can be modified but should be in ways the client cannot detect. Achieved using `mutable`. Also considers situations where something is bitwise const, but does not behave const. e.g. the altering of data members by returning non-const references.
- When const and non-const member functions have similar implementations, avoid duplication by calling const version in the non-const function and using `const_cast`.


### **Item 4:** Make sure objects are initialised before use
The rules about when objects initialisation is guaranteed or not is too complicated to be worth memorising. Avoid the issue by ==always initialising==. 
- For non-member built in types, this needs to be done manually. e.g. `int x = 0;`
- For the rest, initialisation is the responsibility of the constructors and the use of initialisation list is preferred.


## Ch2: Constructors, destructors and assignment operators  

## Ch3: Resource management  

## Ch4: Designs and declarations  

## Ch5: Implementations  

## Ch6: Inheritance and Object-Oriented Design  

## Ch7: Templates and generic programming  

## Ch8: Customising `new` and `delete`  

## Ch9: Miscellany  


