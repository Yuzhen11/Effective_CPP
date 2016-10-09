# Ch5: Implementations

## Item 26: Postphone variable definitions as long as possible
```c++
std::string encryptPassword(const std::string& password) {
    using namespace std;

    string encrypted;
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ...
    return encrypted;
}
```
If exception throws, need to pay for construction and destruction.

Better to postphone the definition and use maybe copy-ctor to construct.
```c++
std::string encryptPassword(const std::string& password) {
    using namespace std;

    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ...
    std::string encrypted(password);
    return encrypted;
}
```

```c++
// Approach A
Widget w;
for (int i = 0; i < n; ++ i) {
    w = ...
}

// Approach B
for (int i = 0; i < n; ++ i) {
    Widget w(...);
}
```
Approach A: 1 ctor + 1 dtor + n assginments

Approach B: n ctors + n dtors

It depends.

A is generally more efficient makes the name w visible in a larger scope.

### Things to Remember
* Postphone variable definitions as long as possbile. It increases program clarity and improves program efficiency.

## Item 27: Minimize casting

```c++
(T)expression;  // C-style
T(expression);  // function style
// new style
cosnt_cast<T>(expression);
dynamic_cast<T>(expression);
reinterpret_cast<T>(expression);
static_cast<T>(expression);
```
* const_cast is typically used to cast away the constness of object.
* dynamic_cast is primarily used to perform "safe downcasting", i.e., to determine whether an object is of a particular
type in an inheritance hierarchy. (It's the only cast that cannot be performed using the old-style syntax, may have significant runtime cost)
* reinterpret_cast is intended for low-level casts that yield implementation-dependent results.
* static_cast can be used to force implicit conversions.

New-style casts are much easier to identify in codeand have more narrowly specified purpose.

The only time to use an old-style cast is when calling an explicit ctor to pass an object to a function.
```c++
class Widget {
public:
    explicit Widget(int size);
};
void doSomeWork(const Widget& w);
doSomeWork(Widget(15));  // create Widget from int with function-style cast
doSomeWork(static_cast<Widget>(15));  // with c++ style cast
```

Casts actually do something instead of just telling compilers to treat one type as another.

```c++
int x,y;
double d = static_cast<double>(x)/y;  // The underlying representation of int and double are different

class Base {};
class Derived: public Base {};
Derived d;
Base* pd = &d;   // the two pointer values will not be the same. An offset maybe
```

An interesting thing about casts is that it's easy to write something that looks right but is wrong.

```c++
class Window {
public:
    virtual void onResize() {...}
};
class SpecialWindow: public Window {
public:
    virtual void onResize() {
        static_cast<Window>(*this).onResize();  // doesn't work
    }
};
```
It will create another copy and invoke onResize() on that copy.

The correct way:
```c++
class SpecialWindow: public Window {
public:
    virtual void onResize() {
        Window::onResize();
    }
```

dynamic_cast may perform strcmp many times, so it's costly.

The need for dynamic_cast generally arises becauses you want to perform derived class operations on what you believe
to be a derived class object, but you have only a pointer- or reference-to-base.

To elimilate the need to use dynamic_cast:

1. Use containers that store pointers to derived class directly. So cannot work with different window types.

2. Let it provide virtual functions in the base class.

### Things to Remember
* Avoid casts whenever practical, especially dynamic_casts in performance-sensitive code. If a design requires casting,
try to develop a cast-free alternative.
* When casting is necessary, try to hide it inside a function. Clients can then call the function instead of putting casts
in their own code.
* Prefer C++ style casts to old-style casts. They are easier to see, and they are more specific about what they do.

## Item 28: Avoid return "handles" to object internal

```c++
class Rectangle {
public:
    Point& upperLeft() const { return pData->ulhc; }  // return handles!
    Point& lowerRight() const { return pData->lrhc; }
private:
    std::shared_ptr<RecData> pData;
};

const Rectangle rec(coord1, coord2);
rec.upperLeft().setX(50);
```
Fallout of the limitation of bitwise constness in Item 3.

Should return const reference:
```c++
class Rectangle {
public:
    const Point& upperLeft() const { return pData->ulhc; }
    const Point& lowerRight() const { return pData->lrhc; }
};
```
Dangling problems
```c++
class GUIObject {...};
const Rectangle boundingBox(const GUIObject& obj);  / return a rectangle by value

GUIObject* pgo;
const Point* pUpperLeft = &(boundingBox(*pgo).upperLeft());
```
This is way any function returns a handle to an internal part of the object is dangerous. Even though is return-by-value.

This doesn't mean that you should never have a member function that returns a handle. operator[] is an exception.

### Things to Remember
* Avoid return handles (references, pointers, or iterators) to object internals. It increases encapsulation, helps
cosnt member functions act const, and minimizes the creation of dangling handles.

## Item 29: Strive for exception-safe code

```c++
class PrettyMenu {
public:
    void changeBackground(std::istream& imgSrc);
private:
    Mutex mutex;
    Image* bgImage;
    int imageChanges;
};
void PrettyMenu::changeBackground(std::istream& imgSrc) {
    lock(&mutex);
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
    unlock(&mutex);
}
```
When an exception is thrown, exception-safe functions:

* Leak no resources

    If "new Image(imgSrc)" yields an exception, the call to unlock never get executed.

* Don't allow data structures to become corrupted

    If "new Image(imgSrc)" throws, bgImage is left pointing to a deleted object and imageChanges has been incremented throws, bgImage is left pointing to a deleted object and imageChanges has been incremented throws, bgImage is left pointing to a deleted object and imageChanges has been incremented.

Addressing the resource leak: Use object to manage resources (Item 13, Item 14).

```c++
void PrettyMenu::changeBackground(std::istream& imgSrc) {
    Lock ml(&mutex);

    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
}
```

What about data structure corruption?

Three guarantees:
* The basic guarantee: Promise that if an exception is thrown, everything in the program remains in a valid state.
* The strong guarantee: If an exception is thrown, the state of the program is unchanged.
* the nothrow gurantee: Never throw exceptions.
```c++
int doSomething() throw();  // note empty exception spec
```
This doesn't say that doSomething will never throw an exception; it says that if doSomething throws an exception,
it's a serious error.

Modify changeBackground to offer the strong guarantee:
```c++
class PrettyMenu {
    ...
    std::shared_ptr<Image> bgImage;
    ...
};

void PrettyMenu::changeBackground(std::istream& imgSrc) {
    Lock ml(&mutex);
    bgImage.reset(new Image(imgSrc));
    ++imageChanges;
}
```

copy-and-swap.
```c++
struct PMimpl {
    std::shared_ptr<Image> bgImage;
    int imageChanges;
};
class PrettyMenu {
    ...
private:
    std::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc) {
    using std::swap;  // see Item 25
    Lock ml(&mutex);
    std::shared_ptr<PMImpl> pNew(new PMImpl(*pImpl));  // copy
    pNew->bgImage.reset(new Image(imgSrc))  // modify the copy
    ++pNew->imageChanges;
    swap(pImpl, pNew);  // swap
}
```

```c++
void someFunc() {
    f1();
    f2();
}
```
A function can usually offer a guarantee no stronger than the weakest guarantee of the functions it calls.

### Things to Remember
* Exception-safe functions leak no resources and allow no data structures to become corrupted, even when
exceptions are thrown. Such functions offer the basic, strong, or nothrow guarantees.
* The strong guarantee can often be implemented via copy-and-swap, but the strong guarantee is not 
practical for all functions.
* A function can usually offer a guarantee no stronger than the weakest guarantee of the functions it calls.

## Item 30: Understand the ins and outs of inlining

Inlining may increase the size of your object code.

Inline is a request to compilers.

Implicit inline is to define a function inside a class definition. Also friend functions.

Inline functions must typically be in header files, because most build environments do inlining during compilation.

Template instantiation is independent of inlining. Declare it inline if necessary.

virtual functions cannot be inlined. virtual means "wait until runtime" and inline means "before exectuion, 
replace the call site with the function".

If you program takes the address of an inline function, compilers must typically generate an outlined function body for it.
```c++
inline void f() {...}
void (*pf)() = f;

f();  // this call will be inlined
pf();  // this call won't be
```

ctors and dtors are often worse candidates for inlining. 

ctors need to construct every data members. If an exception is thrown during construction, any parts of the 
object that have already been fully constructed are automatically destroyed.
```c++
Derived::Derived() {
    Base::Base();
    try { dm1.std::string::string(); }
    catch(...) {
        Base::~Base();
        throw;
    }
    try { dm2.std::string::string(); }
    catch(...) {
        dm1.std::string::~string();
        Base::~Base();
        throw;
    }
    ...
}
```
Library designers must evaluate the impact of declaring functions inline. It's impossible to provide binary
upgrades to the client-visible inline functions in a library.

If f is an inline function in a library, clients of the library compile the body of f into their application.

Initially, don't inline anything, or at least limit your inlining to those functions that must be inline or are truly trivial.

### Things to Remember
* Limit most inlining to small, frequently called functions. This facilitates debugging and binary upgradability, 
minimizes potential code bloat, and maximizes the chances of greater program speed.
* Don't declare function templates inline just because they appear in header files.

## Item 31: Minimize compilation dependencies between files

pimpl

```c++
class PersonImpl;
class Date;
class Address;
class Person {
public:
    person(const std::string& name, const Date& birthday, const Address& addr);
    std::string name() const;
    std::string birthDate() const;
    std::string address() const;
private:
    std::shared_ptr<PersonImpl> pImpl; // ptr to implementation
};
```

The key to this separation is replacement of dependencies on definitions with dependencies with declarations.

* Avoid using objects when object references and pointers will do.
* Depend on class declarations instead of class definitions whenever you can
* Provide separate header files for declarations and definitions.

An alternative to the Handle class approach is to use an Interface class.

```c++
class Person {
public:
    virtual ~Person();
    virtual std::string name() const = 0;
    virtual std::string birthDate() const = 0;
    virtual std::string address() const = 0;

    static std::shared_ptr<Person> create(const std::string& name, const Date& birthday, const Address& addr);
};
class ReadPerson: public Person {
public:
    ...
};

// Clients use them like this
std::shared_ptr<Person> pp(Person::create(name, dateOfBirth, address));
pp->name();
```

The two methods have some costs. An indirect pointer and virtual function call. Cannot inline.

### Things to Remember
* The general idea behind minimizing compilation dependenciese is to depend on declarations instead of definitions.
Two approaches based on this idea are handle classes and Interface classes.
* Library header files should exist in full and declaration-only forms. This applies regardless of whether
templates are involved.
