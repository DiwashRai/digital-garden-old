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
Don't call virtual functions during construction or destruction as they won't
do what you expect them to. If they did, it would lead to undefined
behaviour.

Example: Say you have a base class `Transaction` that is inherited by a 
class called `BuyTransaction`. If you have a virtual function in Transaction
that is called during construction, it will call the base class version. Not
the derived version. This is because the base class is constructed first so
the derived members will not have been constructed yet.

> [!abstract] Summary
> - Don't call virtual functions from constructors or destructors.

### **Item 10:** Have assignment operators return a reference to `this`
Assignments can be chained together and is right associative.
```cpp
int x, y, z;
x = y = z = 15;
//equivalent to
x = (y = (z = 15));
```
This is implemented by assignments returning a reference to it's left hand argument. This is the convention that should be followed.

```cpp
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs)
    {
        ...
        return *this;
    }
    ...
}
```

> [!abstract] Summary
> - Have assignment operators return a reference to `*this`

### **Item 11:** Handle assignment to self in operator=.
This looks silly but is allowed:
```cpp
Widget w;
...
w = w;
```
``
If it's allowed, clients will end up doing it. A less obvious version may look
like this:
```cpp
// potential assignment to selfs
a[i] = a[j];
*px = *py;
```

When writing resource managing classes you can fall into the trap of
accidentally releasing a resource before you're done with it:
```cpp
class Bitmap {...};
class Widget{
    ...
private:
    Bitmap *pb;
}

Widget& Widget::operator=(const Widget& rhs)
{
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

Here you delete rhs not realising that it is the same as `this`. The
traditional fix is this:
```cpp

Widget& Widget::operator=(const Widget& rhs)
{
    if (this == &rhs) return *this;
    
    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```
This is now self-assignment safe but not exception safe. If `new Bitmap`
causes an exception, it will still cause issues.

Making operator= exception-safe also typically also renders it
self-assignment-safe. It is common to deal with self-assignment issues by
ignoring them and isntead focusing on exception safety. A careful ordering
of statements can make code exception safe. For example:

```cpp
Widget& Widget::operator=(const Widget& rhs)
{
    Bitmap *pOrig = pb;
    pb = new Bitmap(*rhs.pb);
    delete pOrig;

    return *this;
}
```

> [!tip] Performance Hint  
> The identity test can still be placed at the top for efficiency, however,
> consider how often self-assignment occurs. The check is not free. It
> makes the source code and object bigger and introduces a branch.
>

Alternative to manual ordering is the "copy and swap" technique.
```cpp
class Widget {
    ...
    void swap(Widget& rhs); // exchanges *this and rhs's data
    ...
};

Widget& Widget::operator=(const Widget& rhs)
{
    Widget temp(rhs);
    swap(temp);
    return *this;
}
```

The final variation sacrifices some clarity for allowing the compiler to
sometimes generate more efficient code.
```cpp
Widget& Widget::operator=(Widget rhs) // rhs is a copy of the object
{
    swap(rhs);

    return *this;
}
```

> [!abstract] Summary  
> - Make sure operator= can handle self-assignment in a well-behaved
> manner
> - Make sure any function operating on more than one object behaves
> well if two or more of the objects are the same.

### **Item 12:** Copy all parts of an object.
Well-designed objects contain only two functions that copy objects: the
copy constructor and copy assignment operator. Compilers will generate
these functions if required and do what you expect them to: copy all the
data of the object being copied.

However, when you declare your own copy functions, they do not warn you
when your implementation is wrong. You may forget to copy a newly
added data member and there will be no warnings.

Another way that you may run into issues is when using inheritance.
```cpp
class PriorityCustomer:public Customer {
public:
    ...
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs);
    ...
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    priority = rhs.priority;
    return *this;
}
```

This may look fine and appear to be copying everything, however, 
crucially, they are not copying the data members it inherits from customer.
The fix to this would be the following:
```cpp

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: Customer(rhs)                 // invoke base class copy ctor
  priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer&
PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    Customer::operator=(rhs);  // assign base class parts
    priority = rhs.priority;
    return *this;
}
```

> [!warning] Warning  
> The two copy functions will have similar code but do not call one from
> the other as it is a non-sensical operation. If you really must, have the
> two call a third private function such as init().

> [!summary] Summary
> - Make sure copy functions copy all of an object's data members
> including the base class parts.
> - Don't implement one copy function in terms of the other. Put common
> code in a third function.

## Ch3: Resource management  

### **Item 13:** Use objects to manage resources  
Managing resources manually is very error prone and should be avoided.
Consider the following:
```cpp
void f()
{
    Investment * pInv = createInvestment(); // factory function
    ...                                     // use pInv

    delete pInv;                            // release object
}
```
This may look reasonable but there are many ways it could fail to delete
`pInv`. if there is a pre-mature return within the function, control would
never reach the delete. Another situation is if `createInvestment()` and
`delete pInv` were in a loop that then breaks early. When this happens,
there will be a leak which will also affect any other resources helf by
*that* object.

Technically, carefull programming an avoid this issues, but this is not
realistic. As the code is maintained, someone may add a pre-mature return
without realising the consequences. Or someone may add an exception in a
function that did not use exceptions which will then cause a leak.

To avoid all of this, we can put resources into a resource managing object
which will then free the resources automatically when the destructor is
called.

The STL has smart pointers that are tailormade for resources that are
dynamically allocated on the heap and are used only within a single
block or function.

```cpp
void f()
{
    std::unique_ptr<Investment> pInv(createInvestment());
    ...
}
```
This example demonstrates two critical aspects of using objects to manage
resources:
- ==**Resources are acquired and immediately turned over to
resource-managing objects.**==  The idea that resources are acquired and
immediately turned over to a resource managing object is referred to as
*Resource Acquisition is Initialization*(RAII).
- ==**Resource-managing objects use their destructors to ensure that
resources are released.**== Destructors are automatically called when an object
goes out of scope so resources are released regardless of how control
leaves a block.

> [!abstract] Summary  
> - To prevent leaks, use RAII objects that acquire objects in their
>constructor and delete them in their destructor
>- Two most common RAII classes are std::unique_ptr and
>std::shared_ptr

### **Item 14:** Think carefully about copying behaviour in resource-managing classes  
Not all resources are heap based. For these resources, unique_ptr and
shared_ptr are generally inappropriate as resource handlers.

For example, if you create a RAII  class called `Lock` that unlocks a mutex
in it's destructor so you never forget to unlock a mutex. What should
happen if this object is copied?
```cpp
Lock ml1(&m);
Lock ml2(ml1);
```

There are a few valid options:
- **Prohibit copying:** In many cases it makes no sense to copy a RAII object.
In these situations you should just prohibit it (item 6). This is likely to be
true for an object such as `Lock`.
- **Reference-count the underlying resource:** Sometimes you may want to
hold on to a resource until there are no more objects using it. In this
situation a copy means that the count of objects referring to the resource
is incremented. A shared_ptr should be used to implement this behaviour.
shared_ptr also allows using a custom "deleter" where you can specify
a different behaviour (such as unlocking a mutex) instead when the
reference count goes to 0.
- **Copy the underlying resource:** Sometimes you can have as many copies
as you want of a resource. The only reason to have a resource-managing
class would be to ensure resources are released. In this case, copying the
RAII object should also mean copying the resource it wraps. An example
would be std::string.
- **Transfer ownership of the underlying resource:** In rare situations, you
may want to ensure that only one RAII object refers to a resource. Here
you will want to make sure that a copy means that ownership is
transferred.

Remember that copying function will be generated by the compiler so you
will need to need to write the custom behaviour yourself.

> [!abstract] Summary  
> - Copying an RAII object entails copying the resource it manages. The
> correct behaviour of the RAII object depends on the copying behaviour
> of the underlying resource.
> - Common RAII class copying behaviours are: disallowing copy and
> reference counting. However, there are others.

### **Item 15:** Provide access to raw resources in resource-managing classes  
Resource-managing classes are great but they can't always be fully relied
on. Many APIs refer to the resources directly so from time to time you will
need to deal with raw resources.
```cpp
int daysHeld(const Investment* pi);
```

There are two ways to convert a RAII object to the resource it contains:
explicit conversion and implicit conversion.

Smart pointers contain a `get` member function that performs an explicit
conversion i.e. returns a copy of the raw pointer inside the smart pointer
object.
```cpp
int days = daysHeld(pInv.get());
```

As for implicit conversion, it is a bit more dangerous but can sometimes
still be the correct choice. Here is an example of a Font class that
manages an underlying FontHandle resource with an implicit conversion
function.
```cpp
class Font {
public:
    ...
    operator FontHandle() const { return f; }
    ...
private:
    FontHandle f;
};

// explicit conversion used to call changeFontSize
changeFontSize(f.get(), newFontSize);
// implicit conversion version of same code
changeFontSize(f, newFontSize);
```

However, now a client might accidentally create a FontHandle when a font
was intended.
```cpp
Font f1(getFont());
...
FontHandle f2 = f1;
// meant to copy a font object but implicityly converted f1
// into its underlying FontHandle and then copied that
```
Now the same FontHandle is being used in f1 and f2 which will then result
in a dangling pointer.

> [!abstract] Summary  
> - APIs often require access to raw resources so RAII classes should
> offer a way to access the resource it manages.
> - Access may be explicit or implicit. Explicit is safer but implicit is
> more convenient.

### **Item 16:** Use the same form in corresponding uses of `new` and `delete`  
```cpp
std::string *stringArray = new std::string[100];
...
delete stringArray;
```
The above code is wrong as you are constructing an array of objects but
deleting only 1 object. `new` allocates only enough space for an object and
then constructs that object if its a single object. For an array, it also
includes the memory required to store information on the size of the array.
Therefore, when you delete an array, you need to access the information.
This can only be done if you tell the compiler that the information is there
by using `delete []`.

> [!warning] Warning  
> - Make sure to use same for of `new` in all of your constructors so you
> know which form to call in the destructor.
> - Abstain from using array types in `typedefs` so there is no confusion
> on which form of `delete` to call when using a typedef.

> [!abstract] Summary  
> - Match the forms of `new` and `delete`. `new` requires a corresponding
> `delete` and `new []` would require a corresponding `delete []`.

### **Item 17:** Store newed objects in smart pointers in standalone statements  
Consider a situation where we have a function that returns a priority and
a function that processes a dynamically allocated Widget.
```cpp
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);
```

Now consider the following way to call processWidget.
```cpp
processWidget(std::shared_Ptr<Widget>(new Widget), priority());
```

Three things need to happen:
- Call priority()
- Execute 'new Widget'
- Call the shared_ptr constructor

Compilers have leeway over which order these things are done. The order
they are done could be:
- Execute 'new Widget'
- Call priority()
- Call shared_ptr constructor

The problem can now be seen. If there is an exception when priority() is
called, the pointer returned from 'new Widget' will be lost and not stored
in the shared_ptr we used to guard against leaks.

To prevent this, use a separate statement to create and store the Widget
resource.
```cpp
std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());
```
The revised code has less leeway to reorder the various components that
made the previous 1 line version of these two statements.


> [!summary] Summary
> - Store newed objects in smart pointers in standalone statements.
> Not doing so can lead to subtle resource leaks if exceptions are thrown.


## Ch4: Designs and declarations  

### **Item 18:** Make interfaces easy to use correctly and hard to use incorrectly.  
If clients use your code incorrectly, ==your interface is partially to blame==.
Consider the design below for a Date class:
```cpp
class Date {
public:
    Date (int month, int day, int year);
    ...
};
```
Two errors can easily happen:
- Client passes in day value into month. `Date d(30, 3, 1995);`
- Client passes in invalid value for month or date. `Date d(3, 40, 1995);`

Many similar client errors can be fixed by using simple wrapper with
explicit constructors.
```cpp
struct Day {
    explicit Day(int d)
    : val(d) {}

    int val;
};
Date::Date(const Month& m, const Day& d, const Year& y);
```

Full fledged classes for day, month and year would be even better but
even structs are enough to show that introducing types can prevent a
interface usage errors.

Once the right types are being used, you can then progress to limiting the
values of those types if it is reasonable to do so. Enums can be used to do
this but they are not as typesafe as we would like as they can be used like
ints.

```cpp
class Month {
    static Month Jan() { return Month(1); }
    static Month Feb() { return Month(2); }
    ...
}
Date d(Month::Mar(), Day(30), Year(1995));
```

> [!tip] Hint  
> This item was probably written before scoped enums (enum classes)
> were available. Now you probably could replace this with enum classes.

*Another way to prevent client errors is to add const.* For example you
can const qualify operator* which can prevent this error  
`if (a * b = c)...`  
This is an assignment when it should have been a comparison.

*Try to make your interfaces behave consistently as possible.* Consistency
is one of the most important characteristic to making easy to use
interfaces. Conversely, inconsistency leads to aggravating interfaces

*Any interface that relies on the client to remember to do something is
prone to incorrect use, because clients cant forget to do it.* An example
is returning raw pointers.

### **Item 19:** Treat class design as type design.  


### **Item 20:** Prefer pass-by-reference-to-const to pass-by-value.  


### **Item 21:** Don't try to return a reference when you must return an object.  


### **Item 22:** Declare data members private.  


### **Item 23:** Prefer non-member non-friend functions to member functions.  


### **Item 24:** Declare non-member functions when type conversions should apply to all parameters.  


### **Item 25:** Consider support for a non-throwing swap.  


## Ch5: Implementations  

## Ch6: Inheritance and Object-Oriented Design  

## Ch7: Templates and generic programming  

## Ch8: Customising `new` and `delete`  

## Ch9: Miscellany  


