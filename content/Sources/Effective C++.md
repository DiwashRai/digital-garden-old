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

### **Item 5:** Know what functions C++ silently writes and calls
Compilers will declare the following if you do not do so yourself:
- Copy constructor
- Copy assignment operator
- Destructor

If no constructors are declared at all, compilers will also declare:
- Default constructor

### **Item 6:** Explicitly disallow the use of compiler-generated functions you do not want
An example of an unwanted generated function is a copy constructor and copy assignment operator for objects that should not be copied.  

```cpp
HomeForSale h1;
HomeForSale h2;
HomeForSale h3(h1); //should not compile
h1 = h2;            // should not compile
```

To achieve this there is a ==well known trick==. Declare the functions private and do not implement them.

```cpp
class HomeForSale {
public:
    ...
private:
    HomeForSale(const HomeForSale&);
    HomeForSale& operator=(ocnst HomeForSale&);
}
```

This trick works and generates a link-time error. It is possible to move
it up to a compile time error by declaring the functions private in a base
class designed just to prevent copying.

>[!abstract] Summary
>- Disallow compiler generated function by declaring the function private and not providing an implementation

### **Item 7:** Declare destructors virtual in polymorphic base classes
Lets say you have a base class `TimeKeeper` and some derivided classes
that inherit from it:  
- AtomicClock
- WaterClock
- WristWatch

If you try to delete a derived class (e.g. WristWatch) with a base class
pointer (TimeKeeper), the results are undefined. Most likely, the derived
class parts of the object won't be deleted. This is a good way to leak
memory.

Solution is simple: ==give the base class a virtual destructor==.

However, when a class is not meant to be used as a base class, making it
virtual is usually a bad idea. This is because virtual functions require the
use of vptr ("virtual table pointers") which increases the size of the type.
A simple data type that contains two ints, would go from 64bits in size to
128bits. An increase of 100%. It also makes the object less portable to C.

If you want to make a class abstract that does not have functions to make
virtual, you can just give it a pure virtual destructor. This is because a
abstract classes are meant to be base classes and base classes should
have a virtual destructor.

> [!abstract] Summary
> - Polymorphic base classes should declare virtual destructors.
> - Any class with virtual functions should have a virtual destructor.
> - Classes that aren't base classes should not declare virtual destructors.
> - Use a pure virtual destructor to make classes abstract that do not have functions

### **Item 8:** Prevent exceptions from leaving destructors  
C++ allows destructors to emit exceptions but it is heavily discouraged.
This is because if an exception is thrown, for example, in a vector
containing 10 widgets, the rest of the widgets still need to be deleted or
there will be leak. However, if you then have another exception when
deleting the other widgets, now you have two simultaneous exceptions
which results in undefined behaviour.

Example class that closes database connection in destructor:
```cpp
class DBConn {
public:
    ...
    ~DBConn()
    {
        db.close();
    }
private:
    DBConnection db;
}
```
This will cause issues if the call to close results in an exception. The
exception will be propagated.

There are two options to fix this:  
- ==Terminate the program== (e.g. with std::abort()). This is a reasonable solution if the program can no longer run due to the error.
- ==Swallow the exception==. This is a bad idea as we need to know when something fails.

The actual ideal solution is to create a function that closes the connection
in the DBConn interface so the client can react to the exception. Then
the backup call to 'close()' can still be placed in the destructor. This may
seem like it goes against *item 11*, by placing an extra burden on the client,
however it is not as it actually **gives** them a chance to react to the
exception.

> [!abstract] Summary
> - Destructors should never emit exceptions.
> - If a classes clients need to react to exceptions during an operation, offer a regular 'non-destrcutor' function that performs the operation.

### **Item 9:** Never call virtual functions during construction or destruction


### **Item 10:** Have assignment operators return a reference to `this`


### **Item 11:** Handle assignment to self in operator=.


### **Item 12:** Copy all parts of an object.



## Ch3: Resource management  

## Ch4: Designs and declarations  

## Ch5: Implementations  

## Ch6: Inheritance and Object-Oriented Design  

## Ch7: Templates and generic programming  

## Ch8: Customising `new` and `delete`  

## Ch9: Miscellany  


