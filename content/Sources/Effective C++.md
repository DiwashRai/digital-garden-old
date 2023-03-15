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
`Investment* createInvestment();`  
This function creates the opportunity for two errors: not deleting the
pointer and deleting it twice. This can be avoided by returning a smart
pointer in the first place.  
`std::shared_ptr<Investment> createInvestment();`  
In fact shared_pt also make sit possible to get rid of other errors with the
ability to specify a custom release function (deleter). It can also get rid
of the cross-DLL problem. This is a problem where an object is created
using `new` in one DLL but is deleted in a different DLL. shared_ptr gets
rid of this as its default deleter uses `delete` from the same DLL where
shared_ptr is created.

> [!abstract] Summary  
> - Strive to create interfaces that are easy to use correctly but difficult
> to use incorrectly.
> - For correct use: maintain consistency in interfaces and use built-in
> types for behavioural compatiblity.
> - For error prevention: create new types, restrict operations on types,
> constrain object values, and eliminate client resource management
> responsibilities.
> - std::shared_ptr supports custom deleters. This prevents cross DLL
> problem and can be used to automatically unlock mutexes.

### **Item 19:** Treat class design as type design.  
Look at class designing as type designing. This means as a C++ developer
you are a *type designer*. Designing good classes is a challenge because
designing good types are a challenge.

To understand the issues you face which will help you design your type
you should consider the following questions:
- How should objects of your new type be created and destroyed?
- How should object initialisation differ from object assignment?
- What does it mean for objects of your new type to be passed by value?
- What are the restrictions on legal values of your new type?
- Does your new type fit into an inheritance graph?
- What kind of type conversion are allowed for your new type? - consider
implicit and explicit conversions. You may need conversion functions if you
are looking to make it explicit only.
- What operators and functions make sense for the new type?
- What standard functions should be disallowed? - declare them private or
delete them(C++11).
- Who should have access to the members of your new type?
- What is the "undeclared interface" of your new type? - what guarantees
does it offer with respect to performance, exception safety and resource
usage.
- How general is your new type?
- Is a new type really what you need? - if it's a family of types, you
might want a class template.

> [!abstract] Summary  
> Class design is type design. When designing a new type, consider all 12
> questions listed in this item.


### **Item 20:** Prefer pass-by-reference-to-const to pass-by-value.  
C++ passes by value by default - function parameters are initialised with
copies of the actual argument and function callers get back a copy of the
returned value. These copies are produced by the objects copy constructor
which makes it an expensive operation.

Consider the following:  
```cpp
class Person {
public:
    Person();
    virtual ~Person();
    ...
private:
    std::string name;
    std::string address;
};

class Student: public Person {
public:
    Student();
    virtual ~Student();
    ...
private:
    std::string schoolName;
    std::string schoolAddress;
};

// function declaration
bool validateStudent(Student s);

Student plato;
bool platoIsOk = validateStudent(plato);
```

When the validateStudent function is called the cost is 1 call to the
copy constructor to initialise the parameter and one call to the destructor
when the function returns. Furthermore, the strong objects are also copied
and destroyed. With construction and destruction of the 4 strings
accounted for, ther are 6 calls to constructors and destructors.

Pass by reference-to-const solves this. The const is essential as
previously the callers were shielded from any changes happening to the
Student object as it was a copy. Now that it is a reference, const is
required to provide that assurance.

Passig parameters by reference also prevents the *slicing problem*. This is
when you pass a derived class into a function that accepts the base class.
If you do this by value, the base class constructor will be called and the
derived class part of the object will not be copied. Then when you call the
function that may have been overriden in the derived class, the base class
version will be called.

If you pass by reference (or pointer) the object will behave like the actual
object that is passed in.

References are implemented as pointers under the hood of a C++ compiler.
This means that for built-in types (e.g.int) it is reasonable to choose to
pass-by-value.

Built-in types are small which can lead people to conclude that small types
can be good pass-by-value candidates. However, there are a few reasons
that this should not be done. Small types can sometimes be only small
as they contain only a pointer, however, the object that the pointer points
to might then be expensive to copy. Another reason is that compilers will
treat built-in types differently to user-defined types. For example,
compilers can refuse to put user-defined types that consist of just a
double in the register even if it will do so with a naked double. Finally,
user-defined types might change in the future so something that is small
now can increase in size later.

==Generally, only built-in types, STL iterators and function object types can
reasonably be assumed as good pass-by-value candidates==.

> [!abstract] Summary  
> - Prefer pass-by-reference-to-const over pass-by-value for efficiency
> and to sidestep the slicing problem.
> - The exceptions to the rule are built-in types, STL iterators and
> function object types.

### **Item 21:** Don't try to return a reference when you must return an object.  
Do not be over zealous in the pursuit of eliminating the 'evil' of
pass-by-value. This can lead to passing references of objects that do not
exist.

Consider a class representing rational numbers.
```cpp
class Rational {
public:
    Rational(int numerator = 0,
             int denominator = 1);
    ...
private:
    int n, d;
friend:
    const Rational
      operator*(const Rational& lhs,
                const Rational& rhs);
};
```

The `operator*` function returns the result object by value. The question is,
can you avoid the cost of the object's deconstruction and construction?

Example scenario:
```cpp
Rational a(1,2); // 1/2
Rational b(3,5); // 3/5

Rational c = a * b; // 1/2 x 3/5 = 3/10
```

It may be tempting to consider the following implementation:
```cpp
const Rational& operator*(const Rational& lhs,
                          const Rational& rhs)
{
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d);
    return result; // returns a reference
}
```
This is useless as the constructor and deconstructor is still being called.
More crucially though, the function returns a reference to a local object
which will immediately go out of scope and we are in undefined territory.

Now you might consider returning a reference to an object on the heap.
Yet again, this would be bad. You still have to pay for the constructor, but
now the client needs to remember to delete the newed object. And even if
they are, how do they handle a situation like this:  
```cpp
Rational w, x, y, z;
w = x * y * z;
```
There are two calls to operator* but now way to delete the first call. This
is a guaranteed resource leak.

Using a static 'result' object is also a no go. It will run into problems with
situations like this:
```cpp
Rational a, b, c, d;
if ((a * b) == (c * d))
    ...
```
This will always be true as there are two active calls to operator* when
operator== is called. They will always be equal as they share the static
result object.

The right answer is simply something such as:
```cpp
inline const Rational operator*(const Rational& lhs,
                                const Rational& rhs)
{
    return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

> [!abstract] Summary  
> Never return a pointer or reference to a local stack object, a reference
> to a heap-allocated object, or a pointer or reference to a local static
> object if there is a chance that more than one such object will be
> needed.

### **Item 22:** Declare data members private.  
The case for this item will follow this format: highlight why public data
members is bad -> highlight why all the arguments also apply to protected
members -> conclude that private is the way to go.

If everything in the public interface is a function, clients won't have to
recall whether to use parantheses to access members or not. Consistency.

Functions provide fine-grained control over the accessiblity of it's data
members.

Finally, encapsulation. If access to a data member is via a function, it is
possible to then later replace the data member with a computation and your
clients will not be aware or need to be aware at all.

A quick example is a class that contains speed data:
```cpp
class SpeedDataCollection {
    ...
public:
    void addValue(int speed);
    double averageSoFar() const;
    ...
}
```
You can use an approach that maintains a data member that contains the
running average and the `averageSoFar()` function just returns this value OR
an approach where the `averageSoFar()` is calculated from all the stored
values whenever it is called. One has better performance on
`averageSoFar()` at the cost of space and the other has worse performance
but doesn't take as much space.  
With encapsulation, you can interchange these easily.

==Encapsulation is more valuabled than it might initially appear==. If data
members are hidden, class invariants can be maintained. This has benefits
when it comes to threaded environments as well. ==Hiding details is like
reserving the right to change the implementation details at a later date==.
You will realise that even if you own the source code to a class, your
ability to change anything public will be severely limited.

*The argument for protected members is identical*. This is because
somethings encapsulation is inversely proportional to the amount of code
that could break if that something changes. For public members this can
be 'all client code' which is an unknowably large amount. How about for
protected members? The answer is 'all the derived classes that access it'
which is yet again an unknowably large amount of code.

==Essentially there are two levels of access: private and everything
else.==

> [!abstract] Summary  
> - Declare data members private.
> - Protected is no more encapsulated than private.

### **Item 23:** Prefer non-member non-friend functions to member functions.  
Consider a web browser class which may have functions to manage the
cache, history and cookies:
```cpp
class WebBrowser {
public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
    ...
};
```

Now consider two ways of implementing a function that combines all three.
```cpp
class WebBrowser {
    public:
    ...
    void clearEverything();
    ...
};

void clearBrowser(WebBrowser& wb) // non-member function
{
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```

Which is better? The member or non-member function? A misunderstanding
of object oriented principles may lead people to conclude that data and
functions that operate on them should be bundled together. This is wrong.
Object oriented principles dictate that data should be as *encapsulated* as
possible.

The greater something is encapsulated, the greater our ability to change
it. Adding another member function increases the public functions that
access the private data members. In the example it would go from 3 to 4.
However, the non-member function does not affect the number of
functions that can access the private parts of the class. Therefore, it is
more encapsulated with a non-member function.

A way to 'bundle' convenience/utility functions without affecting
encapsulation is to use namespaces. This is a very flexible approach as
convenience functions can be spread out over different files but in the
same namespace. Clients can then selectively `#include` only the parts
that they require. It also allows for easy extensibility by the clients
or by the writers in future.

> [!abstract] Summary  
> Prefer non-member non-friend fucntions to member functions. Doing so
> increases encapsulation, packaging flexibility, and functional
> extensibility.

### **Item 24:** Declare non-member functions when type conversions should apply to all parameters.  
Classes supporting implicit type conversions is generally a bad idea,
however there are exceptions. A common exception is for numerical types.

Consider a `Rational` class:
```cpp
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    int numerator() const;
    int denominator() const;
private:
    ...
};
```

Next lets say you want to support multiplication of Rational numbers. The
question is should it be implemented as a member function, non-member
function or non-member function that are friends.

Well lets consider the member function version:
```cpp
class Rational {
public:
    ...
    const Rational operator*(const Rational& rhs) const;
};
```

This design allows for the following:
```cpp
Rational oneEighth(1,8);
Rational oneHalf(1,2);

Rational result = oneHalf *oneEighth;  //fine
result = result * oneEighth;           //fine
```

However, you also want to support mixed-mode operations where Rationals
might be multiplied with ints for example. Lets see what happens.

```cpp
result = oneHalf * 2;       //fine
result = 2 * oneHalf;       //error
```

The source of the problem is clearer if you rewrite it:
```cpp
result = oneHalf.operator*(2);    //fine
result = 2.operator*(oneHalf);    //error
```

oneHalf is an instance of a class containing the operator* where as 2 is
not. But then this raises the question, why does it work when you pass
in an int (2)? This is because of implicit conversion. Compilers can figure
out that although you are passing an int into a function that requires a
`Rational`, the int can be use to construct the `Rational` so that's what it
does.

This means that if the constructor for `Rational` was declared explicit,
then it would fail.

Anyhow, the key to the solution of supporting operations both ways around
is that implicit conversion is only available for parameters that are listed
in the parameter list. Here is the solution:

```cpp
class Rational {
    ...   // operator* not defined/declared
};

const Rational operator*(const Rational& lhs,
                         const Rational& lhs);
{
    ...
}

Rational oneFourth(1, 4);
Rational result;

result = oneFourth * 2;
result = 2 * oneFourth;   // both finally working
```

Should we make this function a friend function? No. It can already achieve
what it needs to do without breaking encapsulation. A function should not
be made a friend just because it is 'related' to it.

> [!abstract] Summary  
> If you need type conversions on all parameters to a function including
> the one pointed to by `this`, the function must be a non-member.

### **Item 25:** Consider support for a non-throwing swap.  
Swap is an interesting function that is worth some special consideration.
Here is a default implementation for swap:

```cpp
namespace std {
    template<typename T>
    void swap(T& a, T& b)
    {
        T temp(a);
        a = b;
        b = temp;
    }
}
```

This default implementation may not thrill you as it involves copying three
objects. For some types, none of this copies will be necessary. For types
like this, the default swap implementation will slow down your code.

The most common example for types like these are those consisting
primarily of pointers to another type that then contains the real data. A
common manifestation of this design approach is the "pimpl idiom"
("pointer to implementation" - item 31). An example using a Widget class:

```cpp
class WidgetImpl {
public:
    ...
private:
    int a, b,c;             //possibly lots of data
    std::vector<double> v;  //expensive to copy
};

class Widget {
public:
    Widget(const Widget& rhs);
    Widget& operator=(ocnst Widget& rhs)
    {
        ...
        *pImpl = *(rhs.pImpl);
        ...
    }
    ...
private:
    WidgetImpl *pImpl;
};
```

In this example, all we need to do to swap Widget objects is to swap
their underlying `pImpl` pointers. However, the default swap function does
not know this. It would result in copy three widgets and ALSO three
WidgetImpl objects. Very inefficient.

We would like to tell std::swap how to more efficiently handle the swap.
```cpp
namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b)
    {
        swap(a.pImpl, b.pImpl);  // wont compile. private ptrs
    }
}
```
The "template<>" at the begining of the fucntion says that this is a *total
template specialisation* for std::swap and the "< Widget >" after the name
of the function says that the specialisation is for when T is Widget.

In general, we are not permitted to alter contents of the std namespace,
but we are allowed to totally specialise standard template for our own
types.

The above code won't compile due to `pImpl` being private. We could make
the specialisation a friend, but the convention is to declare a public
member function calle dswap that does the actual swapping, then
specialise the std::swap to call the member function.

```cpp
class Widget {
public:
    ...
    void swap(Widget& other)
    {
        using std::swap; //need for this explained below
        swap(pImpl, other.pImpl);
    }
    ...
};
namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b)
    {
        a.swap(b);
    }
}
```

This compiles AND is consistent with the STL containers.

But what if `Widget` and `WidgetImpl` were class *templates* instead of
classes. Putting a swap member function in Widget is as easy as before.
The trouble arises with the specialisation. We 'want' to write:
```cpp
namespace std {
    template<typename T>
    void swap<Widget<T> > (Widget<T>& a,
                           Widget<T>& b)
    {a.swap(b);}
}
```

This looks reasonable but is illegal code. C++ supports partial 
specialisation for classes but not function templates. To partially
specialise a function template, the usual approach is to simply use an
overload.
```cpp
namespace std {
    tempalte<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    {a.swap(b);}
}
```

In general this would be fine, but the std namespace is special. You can
totally specialize templates in std, but you can't add new templates (or
classes or functions or anything else). If you break this rule, you are
once again in the land of `undefined`.

So what do we actually do? We still declare a non-member swap that calls
the member swap, we just don't declare the non-member to be a
specialisation or overloading of std::swap.

```cpp
namespace WidgetStuff {
    ...
    template<typename T>
    class Widget{...};
    ...

    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b)
    { a.swap(b); }
}
```

Now, if any code anywhere calls swap on two Widget objects, the name 
lookup rules in C++ will find the Widget-specific version in WidgetStuff.

This approach works as well for classes as for class templates, so it 
seems like we should use it always. Unfortunately, there is a special
reason for specialising std::swap for classes, so if you want to have
your class specific swap be called in as many contexts as possible
(which you definitely do), you need to write a non-member version in
the same namespace as your class AND a specialisation of std::swap.

The above is for `swap` authors but we should consider one thing for th
clients.
```cpp
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    ...
    swap(obj1, obj2);
    ...
}
```
Which swap does this call? The general one in std? or the specialisation of
the std? or a T-specific one? What you want to do is try to use the
T-specific one, but if not fall back on the general std::swap.  
Heres how you can do this:
```cpp
template<typename T>
void doSomething(T& obj1, T& obj2)
{
    using std::swap;        // make std::swap available
    ...
    swap(obj1, obj2);       // call best swap
    ...
}
```

The C++ lookup rules will ensure the best swap is called so don't worry
too much. Just remember not to qualify it:
`std::swap(obj1, obj2);`  
Qualifying swap does happen, that's why it's in your best interest to aslo
implement the specialised version.

> [!abstract] Summary  
> - Provide a swap member function when std::swap would be 
> infefficient for your type. Make sure swap does not throw exceptions.
> - If you offer a member swap, make sure to also offier a non-member
> swap that calls the member. For classes (not templates), specialise
> std::swap too.
> - When calling swap, employ a using declaration for std::swap, then
> call swap without namespace qualification.
> - It's fine to totally specialise std templates for user-defined type,
> but never try to add something completely new to std.

## Ch5: Implementations  
The lion's share of the battle is coming up with appropriate definitions for
your classes and appropriate declarations for your functions. However,
there are still implementation details to look out for.

### **Item 26:** Postpone variable definitions as long as possible.  
Defining a variable of a type with a constructor or destructor incurs the
cost of construction and then the cost of destruction when the variable
goes out of scope. Due to the inherent cost of using a variable, you want
to avoid using them if possible.

Consider a function that returns an encrypted version of a password if the
password is long enough:
```cpp
std::string encryptPassword(const std::string& password)
{
    using namespace std;
    string encrypted;

    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ...      // encrypted password into encrypted var
    return encrypted;
}
```

The object encrypted is unused an exception is thrown. Therefore, it's
better if you postpone defining it until you know an exception won't be
thrown.
```cpp
std::string encryptPassword(const std::string& password)
{
    using namespace std;

    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    string encrypted;
    ...      // encrypted password into encrypted var
    return encrypted;
}
```

However, even this version isn't tight enough as `encrypted` is defined
without any initialisation arguments. This means it is default constructed
and then only assigned a value later.

Consider if the hard part of the function was done in this function:  
`void encrypt(std::string& s);`  

The code could then look like this:  
```cpp
std::string encryptPassword(const std::string& password)
{
    ...
    string encrypted;
    encrypted = password;

    encrypt(encrypted);
    return encrypted;
}
```

This leads to a pointless default construction of `encrypted` though. Better
would be:  
```cpp
std::string encryptPassword(const std::string& password)
{
    ...
    string encrypted(password);
    encrypt(encrypted);
    return encrypted;
}
```

==Not only should you postpone variable definition until right before you
have to use the variable, but also until you have the initialisation
parameters for it.==

But what about loops?
```cpp
// Approach A: define outside loop
Widget w;
for (int i = 0; i < n; ++i) {
    w = some value dependent on i;
    ...
}

// Approach B: define inside loop
for (int i = 0; i < n; ++i) {
    Widget w(some value dependent on i);
}
```

The associated costs are:
- Approach A: 1 constructor + 1 destructor + n assignments
- Approach B: n constructors + n destructors

For classes where assignment is cheaper than constructor/destructor pair,
approach A is generally more efficient. Otherwise, approach B is probably
better. Default to approach B unless (1) assignment is cheaper than
ctor/dtor pair and (2) you're dealing with performance sensitve area of
code.

> [!abstract] Summary  
> Postpone variable definitons as long as possible. It improves clarity
> and efficiency.

### **Item 27:** Minimise casting.  
Rules of C++ are designed to guarantee that type errors are impossible.
However, casts subvert the type system. They should not be taken
lightly.

Old C style casts:
- `(T) expression`
- `T(expression)`
There is no difference in the meaning between these forms.

New C++ casts
- const_cast< T >(expression);
- dynamic_cast< T >(expression);
- reinterpret< T >(expression);
- static_cast< T >(expression);

Each serves a distinct purpose:
- `const_cast`: cast away constness. The only C++ cast that can do this.
- `dynamic_cast`: Primarily for "safe downcasting" i.e. whether an
object is of a particular type in an inheritance hierarchy. e.g. whether
`shape` can be cast to `triangle`. Cannot be performed in old style cast and
has significant runtime cost.
- `reinterpret_cast`: intended for low-level casts that yield
implementation-dependent (unportable) results. e.g. pointer to int. Should
rarely be used outside of low level code.
- `static_cast`: Used to force implicit conversions (e.g. non-const to
const. int to double etc). It can also be used to perform the reverse
of many such conversions (e.g. void pointers* to type pointers,
pointer-to-base to pointer-to-derived). Just not const to non-const.

New style casts are preferrable as they are easier to identify in code and
their narrower usage provides more helpful usage errors.

==Many programmers believe that casts do nothing but tell compilers to
treat one type as another==. This is wrong. Type conversions of any kind
can lead to code that is executed at runtime.

```cpp
int x, y;
...
double d = static_cast<double>(x) / y;
```
This code almost certainly generates code as the underlying representation
of a double differs from an int on most architectures. Not too surprising
but this may be a bit more unexpected:

```cpp
class Base {...};
class Derived : public Base {...};
Derived d;
Base *pb = &d;
```
We're creating a base class pointer to a derived object, but sometimes
the two pointer values will not be the same. When this happens, an offset
is ==*applied at runtime*== to the Derived* pointer to get the correct Base*
pointer value.

The last example demonstrates that a single object might have more than
one address (e.g. address when pointed by Base* and address when
pointed by Derived*). This does not happen in C, Java or C#, but it does
happen in C++. It happens all the time when multiple inheritance is in use.
But it can also happen in single inheritance. ==This means that you should
generally avoid making assumptions about how things are laid out in
C++ and certainly not make casts based on such assumptions.==

> [!info] Note  
> An offset is 'sometimes' required. Compiler specific implementation
> differs. Just because a cast based on assumed layout works on one
> platform does not mean it will work on another.

Casts make it easy to write code that looks right, but is in fact wrong.
For example, often virtual member function implementations are required
to call their base class counterparts first:
```cpp
class Window {
public:
    virtual void onResize() {...}
    ...
};

class SpecialWindow: public Window {
public:
    virtual void onResize() {
        static_cast<Window>(*this).onResize();
        ...
    }
    ...
};
```
In this code, it may appear to be casting `this`, the current object, into its
base class and then calling `onResize()` on it. However, what it does is
creates a new, ==temporary copy of the base class== part of `this`, calls
`Window::onResize` on the copy of the base class part and continues with
the function and performs the `SpecialWindow` specific actions on the
current object.

The object might now be in an invalid state where the base class
modifications have not been made but the derived class parts have been.
To handle this properly you should do the following:
```cpp
class SpecialWindow: public Window {
public:
    virtual void onResize() {
        Window::onResize();
        ...
    }
    ...
};
```

If you are finding yourself wanting to cast, you could be approaching
things in the wrong way.

`dynamic_cast` is an extremely costly cast. One common implementation
is partly based on string comparisons between class names. In an
inheritance hierarchy of four levels, this would cost 4 calls to `strcmp`. Be
wary of casts in general, but especially wary of dynamic_casts.

The need for dynamic_cast usually arises when you want to perform
derived class operations on what you think is a derived class, but what
you have is a pointer-to-base.

There are two possible solutions to avoid dynamic_casting for this reason:
- Use containers that store pointers to derived class objects directly.
- Use virtual functions in the base class that will allow you to do what
you need.

> [!abstract] Summary  
> - Avoid casts whenever practical, especially dynamic_casts. Explore
> cast-free alternatives if a design requires casting.
> - If casting is necessary, hide it inside a function.
> - Prefer C++ style casts to old-style casts. Easier to spot and
> more specific about what they do.


### **Item 28:** Avoid returning "handles" to object internals.  
Consider an implementation of a rectangle class that stores the upper-left
corner and lower-right corner. Tok keep the `Rectangle` object small, the
points that define it's extent aren't stored in the `Rectangle` itself, but
an auxiliary struct that the rectangle points to.
```cpp
class Point {
public:
    Point(int x, int y);
    ...
    void setX(int newVal);
    void setY(int newVal);
};

struct RectData {
    Point ulhc;
    Point lrhc;
};

class Rectangle {
    ...
private:
    std::shared_ptr<RectData> pData;
}
```

Because the clients of `Rectangle` need to be able to determine the extents
of a rectangle, there are functions `upperLeft` and `lowerRight`. Because
`Point` is a user-defined type and it is more efficient to return a reference,
these functions return references.
```cpp
class Rectangle {
public:
    ...
    Point& upperLeft() const { return pData->ulhc; }
    Point& lowerRight() const { return pData->lrhc; }
    ...
};
```
The design will compile, but it's wrong. The functions are declared const
so they are designed to offer clients a way to learn what the points of the
rectangle are, not to change them. However, since references to internal
data members are returned, they can now be changed.

```cpp
Point coord1(0, 0);
point coord2(100, 100);

const Rectangle rec(coord1, coord2); // rectangle 0,0 x 100,100

rec.upperLeft().setX(50);  // now rec 50,0 x 100,100
```

Two lessons from this example:
- ==A data member is only as encapsulated as the most accessible
function returning reference to it==
  - In this example, `ulhc` and `lrhc` are effectively public.
- ==If a const member function returns a reference to data associated
with an object that is stored outside the object itself, the called can now
change that data.==

The example involved references but the same would be true for pointers
and iterators as they all act as *handlers*.

We generally think of an object's internals as it's data members ,but 
member functions not accessible to the general public are also part of
an object's internals. This means that you should nver have a member
function return a pointer to a less accessible member function.

The rectable example can be simply solved by adding const to their return
types:
```cpp
class Rectangle {
public:
    ...
    const Point& upperLeft() const { return pData->ulhc; }
    const Point& lowerRight() const { return pData->lrhc; }
    ...
};
```
The clients can now 'read' the points but not 'write' into them.
However, `upperLeft` and `lowerRight` are still returning 'handles' to an
object's internals and this can be problematic still. In particular, it can
lead to 'dangling handles'. Consider:  
```cpp
class GUIObject{
...
};

// returns a rectangle by value
const Rectangle boundingBox(const GUIObject& obj);

GUIObject *pgo;
...
// get a ptr to upper left point of bounding box
const Point *pUpperLeft = &(boundingBox(*pgo).upperLeft());
```
Here the call to `boundingBox` returns a rectangle by value. `upperLeft` is
then called on this rectangle and the pointer to this `Point` is assigned to
`pUpperLeft`. Unfortunately, the temporary rectangle object will be
destroyed. That will then also leave `pUpperLeft` pointing to a `Point` that
no longer exists.

> [!abstract] Summary  
> - Avoid returning handles (references, pointers, or iterators) to
> object internals. Doing so maximises encapsulation, helps const
> member functions act const, and minimises the creation of dangling
> handlers.

### **Item 29:** Strive for exception-safe code.  
There are two requirements for exception safety:
- **Leak no resources**
- **Don't allow data structures to be corrupted**

Consider a class for representing GUI menus with background images. The
class is designed to be used in threaded environments so it has a mutex.
```cpp
class PrettyMenu {
public:
    ...
    void changeBackground(std::istream& imgSrc);
    ...
private:
    Mutex mutex;
    Image* bgImage;
    int imageChanges;
}
```

Now here is a possible implementation of the `changeBackground` function:
```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    lock(&mutex);                //acquire mutex
    delete bgImage;              //get rid of old bg
    ++imageChanges;              //update image change count
    bgImage = new Image(imgSrc);  //install new bg
    unlock(&mutex);              //release mutex
}
```

This function fulfils none of the requirements for exception safety. It leaks
resources as if `new Image(imgSrc)` throws, the mutex is never unlocked.
As for data structure corruption, if `new Image` throws, `bgImage` is now
pointing to a deleted object and also `imageChanges` has been incremented
when the new image has not actually been installed.

*Addressing the resource leak*. To do this we just have to use objects to
help us manage resources.
```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock ml(&mutex);   // item 14. ensures release

    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}
```

> [!tip] info  
> Another good thing about resource management classes is that they
> usually make functions shorter.

*Now for the data structure corruption.* We now have a choice. Exception
safe functions offer ==one of three guarantees.==
- **Basic guarantee.** If an exception is thrown, everything in the program
remains in a valid state. No objects or data structures corruped and
internal state is consistent. e.g. for `changeBackground` the original
background is kept or a default background is used.
- **Strong guarantee.** If an exception is thrown, the state of the program
is left unchanged. The functions are ==atomic==. If they succeed, they 
succeed they succeed completely, and if they fail ,the program state is
is as if it was never called at all. Functions offering the 'strong'
guarantee are easier to work with as after they are called there is only
two possible states: same as before or expected state after successful
completion. With the 'basic' guarantee the function could leave the
program in any valid state.
- **The nothrow guarantee**. Never throw exceptions as these functions
always do what they promise to. All operations on built-in types are
nothrow. Functions with an empty exception specification *do not* imply
that they are nothrow. All it says is that if the function does throw an
exception, it is a serious error.

Exception-safe code must offer one of the three guarantees from above.
It is not exception-safe if it does not. The choice is only about which
level of exception-safe code to offer as write exception-unsafe code
will cause resource leaks and corrupt data structures.

The general rule is to offer the strongest guarantee that is practical.
Although nothrow guarantee is great, it is difficult to climb out of just the
C part of C++ whilst guaranteeing it.

*Back to the changeBackground example.* We can do two very simple
things to almost provide the strong guarantee.
```cpp
class PrettyMenu {
    ...
    std::shared_ptr<Image> bgImage;
    ...
};
void PrettyMenu::changeBackgroun(std::istream& imgSrc)
{
    Lock ml(&mutex);
    bgImage.reset(new Image(imgSrc));
    ++imageChanges;
}
```
The two improvements are:
- Using a shared_ptr to avoid resource leaks.
- Reorder the statements so that `imageChanges` is only incremented until
the image has actually been changed.

The improvements make it 'almost' strong as it is possible that the
read marker for input stream 'imgSrc' has been moved.

There is a general design strategy that typically leads to the strong
guarantee. ==Copy and swap==. Make a copy of the object you want to
modify, make the necessary modifications to the copy. If an exception
is thrown, the original is left unchanged. After all changes have been
made, swap the modified object with the original in a non-throwing
operation.

This is usually implemented by putting all the per-object data into a
separate implementation object, then giving the real object a pointer
to its implementation object. This is called the "pimpl idiom".

The ==copy-and-swap== strategy is an excellent way to make all-or-nothing
changes to an object's state, but still does not guarantee that the overall
function is strongly exception safe.

```cpp
void someFunc()
{
    ...    //make copy of local state
    f1();
    f2();
    ...    //swap modified state into place
}
```

If `f1` or `f2` are less than strongly safe, `someFunc` is almost certainly not
strongly safe. But even if they are, `f1` could succeed and `f2` could fail.
This means that the state is not the same as if `f1` and `f2` both ran all the
way successfully. Such issues can prevent you from offering the strong
guarantee.

Another issue with 'copy and swap' is efficiency. The strategy requires
creating a copy of each object to be modified which takes extra time and
space.

If you are unable to provide the strong guarantee because of limitations
such as these, it is perfectly ok to only provide the weak guarantee. Just
make an effort to provide the strong guarantee.

> [!abstract] Summary  
> - Exception-safe functions ==leak no resource== and ==do not allow
> data structure corruption==. Even if exceptions are thrown; basic,
> strong, or nothrow guarantees are provided.
> - Strong guarantee can be provided via copy-and-swap.
> - A function can usually offer a guarantee no stronger than the
> function it itself calls.

### **Item 30:** Understand the ins and outs of linlining.  
Inline functions are a great idea: look like functions, act like functions,
but without the overhead of a real function call and without using macros.
==What more could you ask for?==  
They provide even more benefits! Compiler optimisations also work better
as they are typically designed for long stretches of code that lack function
calls.

However, nothing in life is free. They increase the size of your object
code. This could be a problem on machines with limited memory. Even
with virtual memory, code bloat can lead to additional paging, a reduced
instruction cache hit rate and the performance penalties these incur.

On the other hand, if an inline function body is very short, the code
generated for the function body may be smaller than the code for a
function call. This can lead to *smaller* object code and a higher instruction
cache hit rate.

Bear in mind that `inline` is a 'request' not a command. The request can
be explicit or implicit. The implicit method is to define the function in the
class definition.
```cpp
class Person {
public:
    ...
    int age() const { return theAge; }  //implicit inline req
    ...
private:
    int theAge;
};
```

The explicit way is to precede its definition with the `inline` keyword.
```cpp
inline const T& std::max(const T& a, const T& b)
{ return a < b ? b : a; }
```

Just because inline functions and function templates are usually defined
in header files does **not** mean that function templates must be inline. The
reason that this happens is because inlining and function template
instantiation both happens at compile time. If you inline all function
templates mindlessly, you will bloat the code unnecessarily and incur all of
the associated costs.

*Inline is a request.* Most compilers refuse to inline functions that are too
complicated (contain loops or are recursive) and all but the most trivial
calls to virtual functions. After all virtual means 'figure out which
function to call at runtime'. How you call the function also affects
whether the function is inlined. Calling a function through a pointer will
most likely not result in it being inlined.

It is also important to carefully consider inlining as a library designer as it
is impossible to provide binary upgrades to the client-visible inline
functions in a library. This is because inline functions are compiled into
the clients application. To change this the clients would have to recompile
with the new provided inline function code. This is more onerous than
dynamically linking a non inline function.

This all culminates in the logical strategy that functions should generally
not be declared inline. The 80-20 rule should be recalled and you should
identify the 20% of your code that increases your code's performance
and selectively apply inline.

> [!abstract] Summary  
> - Limit most inlining to small, frequently called functions. This
> facilitate debugging, binary upgradability, minimises code bloat, and
> maxmises chances of greater performance.
> - Don't declare function templates inline just because they appear
> in header files.

### **Item 31:** Minimise compilation dependencies between files.  
C++ does not do a very good job of separating interfaces from
implementations. You might make a small change to an implementation,
just the private stuff and find your whole project recompiling.

```cpp
class Person {
public:
    ...
    Person(const std::string& name, const Date& birthday,
           const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::string theName;  //implementation detail
    Date theBirthDate;    //implementation detail
    Address theAddress;   //implementation detail
};
```

In this example, the `Person` class can't be compiled without access to
definitions that the `Person` ==implementation== uses (string, Date, Address).
Such definitions are typically provided using the `#include` directive.
```cpp
#include <string>
#include "date.h"
#include "address.h"
```

Unfortunately, this leads to compilation dependencies as changes to any
of these files will require Person to also be recompiled. C++ requires the
implementation details of a class in the class definition as compilers need
to know how much memory to allocate for objects. This info cannot be
found in forward declarations. ==This is not a problem in languages like
Java== as in those languages compilers allocate only enough space for a
pointer to an object.

In C++ there are two ways to solve this problem:
- Like Java, play the game of "hide the object implementation behind a
pointer game" using the pimpl idiom.
- Use ==interface classes==.

*pimpl idion method*
```cpp
#include <string>
#include <memory>

class PersonImpl;
class Date;
class Address;

class Person {
    Person(const std::string& name, const Date& birthday,
           const Address addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
    ...
private:
    std::shared_ptr<PersonImpl> pImpl;//ptr to implementation
};
```

With this design, clients of Person are divorced from the details of dates,
addresses and persons. The implementations can be changed at will, but
`Person` clients do not need to recompile. ==Furthermore, since the clients
are unable to view the implementation, the are unlikely to write code that
depends on those details==. Hence:
- *Avoid using objects when object references will do*. You can define
pointers and references to a type with only a declaration for a type.
Defining objects of a type is what necessitates the definition of the class.
- *Depend on class declarations instead of class definitions whenever
you can*. You never need a class definition to declare a function using that
class, not even if the function passes or returns the class type by value.
```cpp
class Date;   //class declaration

Date today(); // fine even though it returns Date by value
void clearAppointments(Date d); // also fine
```
Why declare functions that no one calls? It allows you to have a header
file of function declarations. However, clients of your library will most
likely not use every function. Now the burden of `#include`'ing the types
the clients need is on the client. If you put all of the includes in the
function declaration file, the clients would have artificial dependencies on
files that they do not need as they would be including things that they
don't need from the functions they don't call.
- *Provide separate header files for declarations and definitions.* To
facilitate adherance to these guidelines, header files should come in pairs:
one for declarations, the other for definitions. These files need to be kept
consistent. This means that library clients should `#include` a declaration
file instead of forward declaring themselves. This means that library
authors should provide both of these header files. The above example
would look like so:
```cpp
#include "datefwd.h"

Date today();
void clearAppointments(Date d);
```

Classes like `Person` that use the 'pimpl idiom' are often called handle
classes. Here's how `Person` might be implemented(aka the .cpp file):
```cpp
#include "Person.h"
#include "PersonImpl.h"

Person::Person(const std::string& name, const Date& birthday,
               const Address& addr)
:pImpl(new PersonImpl(name, birthday, addr))
{}

std::string Person::name() const
{
    return pImpl->name();
}
```

Now lets consider the alternative. Interface classes. An interface class
for `Person` might look like this:
```cpp
class Person {
public:
    virtual ~Person();

    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
    virtual std::string address() const = 0;
}
```

==Clients of this class must program in terms of Person pointers and
references as it is impossible to instantiate classes containing pure virtual
functions.== It is however, possible to instantiate classes *derived* from
`Person`. Like handle classes, clients of the interface class do not have to
recompile unless the interface is modified.

Clients of interface classes need a way to create new objects. This is
typically done using 'factory functions' that play the role of the
constructor. They return pointers (preferably smart) to dynamically
allocated objects that support the Interface classes interface. Such
functions are often declared static inside the interface class:
```cpp
class Person {
public:
    ...
    static std::shared_ptr<Person>
        create(const std::string& name,
               const Date& birthday,
               const Address& addr);
    ...
};

// to use the factory function
std::string name;
Date dob;
Address addr;

std::shared_ptr<Person> pp(Person::create(name, dob, addr));
```

At some point the Person interface class will need to be 'implemented'.
That might look like this:
```cpp
class RealPerson: public Person {
public:
    RealPerson(const std::string& name, const Date& dob,
               const Address& addr)
    : theName(name), theBirthdate(dob), theAddress(addr)
    {}
    virtual ~RealPerson() {}
    virtual std::string name() const;
    virtual std::string birthDate() const;
    virtual std::string address() const;
private:
    std::string theName;
    Date theBirthDate;
    Address theAddress;
};

...
shared_ptr<Person> Person::create(const std::string& name,
                                  const Date& birthday,
                                  cosnt Address& addr)
{
    return std::shared_ptr<Person>(new RealPerson(name,dob
                                                  addr));
}
```

This demonstrates one of two most common mechanisms for implementing
an Interface class: inherit its interface specification, then implement the
function from the interface. A second way is using multiple inheritance
(item 40).

==So what are the costs for all this magic?== Some speed at runtime, plus
some additional memory per object. For handle classes, you have to go
through one layer of indirection to get to the object's data per access.
The space required for this extra implementation pointer increases the
memory required as well. Finally, since the object is dynamically
allocated, there are associated costs with that.

For interface classes, every function call is virtual, so you pay the cost
of an indirect jump each time you make a function call. All objects derived
from an interface class also requires a virtual table pointer.

Finally, for both, they don't really benefit from `inline` functions.

Do not dismiss ==handle classes== and ==interface classes== due to these costs
though. Use them evolutionarily. Use them during development and then
replace with concrete classes for production use if you can prove the
speed/size benefit outweighs the coupling.

> [!abstract] Summary  
> - Main idea to reduce compilation dependencies is to rely on
> declarations as much as possible. The two approaches are =='handle
> classes'== and =='interface classes'==.
> - Library header files should exist in full and declaration-only forms
> even if templates are involved.

## Ch6: Inheritance and Object-Oriented Design  

## Ch7: Templates and generic programming  

## Ch8: Customising `new` and `delete`  

## Ch9: Miscellany  


