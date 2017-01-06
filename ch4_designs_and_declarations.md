# Ch4: Designs and Declarations

## Item 18: Make interfaces easy to use correctly and hrad to use incorrectly

```c++
class Date {
public:
    Date(int month, int day, int year);
};

// May be misused
Date d(30, 3, 1995);

// can be prevented by introdution of new types
struct Day {
    explicit Day(int d) : val(d) {}
    int val;
};
...
class Date {
public:
    Date(const Month& m, const Day& d, const Year& y);
};
Date(Month(3), Day(30), Year(1995));
```

Restrict the values

```c++
class Month {
public:
    static Month Jan() { return Month(1); }
    static Month Feb() { return Month(2); }
    ...
private:
    explicit Month(int m);
};
Date d(Month::Mar(), Day(30), Year(1995));
```

Have your types behave consistently with the built-in types.

    if (a*b = c)

size() function in STL containers.

To avoid resource leak:
```c++
Investment* createInvestment();

std::tr1::shard_ptr<Investment> createInvestment();  // force clients to store the return value in a tr1::shared_ptr
```

declare deleter in shared_ptr

```c++
// create a null shared_ptr with getRidOfInvestment as its deleter
std::tr1::shared_ptr<Investment> pInv(static_cast<Investment*>(0), getRidOfInvestment); 
```

Cross-DLL problem: an object is created using new in one dynamically linked library but is deleted in a different DLL.

shared_ptr prevents this problem.

### Things to Remember
* Good interfaces are easy to use correctly and hard to use incorrectly. You should 
strive for these characteristics in all your interfaces.
* Ways to facilitate correct use include consistency in interfaces and behavioral
compatibility with built-in type.
* Ways to prevent errors include creating new types, restricting operations on types, constraining object values,
and eliminating client resource management reponsiblities.
* tr1::shared_ptr supports custom deleters. This prevents the cross-DLL problem, can be used to automatically 
unlock mutexes (see Item 14), etc.

## Item 19: Treat class design as type design

* How should objects of your new type be created and destroyed?
* How should object initialization differ from object assignment?
* What does it mean for objects of your new type to be passes by value?
* What are the restrictions on legal values for your new type?
* Does your new type fit into an inheritance graph?
* What kind of type conversions are allowed for your new type?
* What operators and functions make sense for the new type?
* What standard functions should be disallowed?
* Who should have access to the members of your new type?
* What is the undeclared interface of your new type?
* How general is your new type?
* Is a new type really what you need?

### Things to Remember
* Class design is type design. Before defining a new type, be sure to consider all the issued discussed in this Item.

## Item 20: Prefer pass-by-reference-to-const to pass-by-value

Less copy and avoid slicing problem. 

### Things to Remember
* Prefer pass-by-reference-to-const over pass-by-value. It's typically more efficient and it avoids the slicing problem.
* The rule doesn't apply to built-in types and STL iterator and function object types. For them, 
pass-by-value is usually appropriate.

## Item 21: Don't try to return a reference when you must return an object

If you need to return an object, just return an object.

On-the-stack: Return an reference to local object is bad. Undefined behavior.

On-the-heap: Return an reference to an object on the heap is bad. Resource leak possible.

Return a reference to static local object is bad. Only one copy...

### Things to Remember
* Never return a pointer or reference to a local stack object, a refernce to a heap allocated object, or a pointer
or reference to a local static object if there is a chance that more than one such object will be needed. (Item 4
provides an example of a design where returning a reference to a local static is resonable, at least in single-threaded
environment.)

## Item 22: Declare data members private

Syntactic consistency: everything is a function so no need to remeber whether to use parentheses

Fine-grained control

```c++
class AccessLevels {
public:
    int getReadOnly() const { return readOnly; }

    void setReadWrite(int value) { readWrite = value; }
    int getReadWrite() const { return readWrite; }

    void setWriteOnly(int value) { writeOnly = value; }
private:
    int noAccess; 
    int readOnly;
    int readWrite;
    int writeOnly;
};
```
Allow different implementations under the same interface
```c++
class SpeedDataCollection {
public:
    void addValue(int speed);   // add new data
    double averageSoFar() const;  // return average speed
};
```

### Things to Remember
* Declare data members private. It gives clients syntactically uniform access to data, 
affords find-graned access control, allows invariants to be enforced, and offers class authors implementation flexibility.
* protected is no more encapsulated than public.

## Item 23: Prefer non-member non-friend functions to member functions
```c++
class WebBrowser {
public:
    void clearCache();
    void clearHistory();
    void removeCookies();
};

// Which is better?
// member function
class WebBrowser {
public:
    ...
    void clearEverything();
};

// non-member non-friend function
void clearBrowser(WebBrowser& wb) {
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```
### Encapsulation

Encapsulation affords us the flexibility to change things in a way that affects only a limited number of clients.

The more functions that can access it, the less encapsulated the data.

Using the non-member non-friend function doesn't increase the number of functions that can access the private parts of the class.

Two things: 1. non-friend. 2. Can be member of another class.

We could make clear Browser a static member function of some utility class.

```c++
namespace WebBrowserStuff {
    class WebBrowser { ... };
    void clearBrowser(WebBrowser& wb);
}
```

They can't offer any functionality a WebBrowser client couldn't already get in some other way.

### Partition function functionality

Separate the functionality into different different header files. Like std, separate in vector, algorithm and so on.

Can be easily extend by adding other convenient functions.

```c++
// header webbrowser.h
namespace WebBrowserStuff {
    class WebBrowser {...}  // "core" related functionality, e.g. non-member functions almost all clients need
}
// header webbrowserbookmarks.h
namespace WebBrowserStuff {  // bookmark-related convenience functions
}
// header webbrowsercookies.h
namespace WebBrowserStuff {  // cookie-related convenience functions
}
```

### Things to Remember
* Prefer non-member non-friend functions to member functions. Doing so increases encapsulation, packaging flexibility, and functional extensibility.

## Item 24: Declare non-member functions when type conversions should apply to all parameters
```c++
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    int numerator() const;
    int denominator() const;
private:
    ...
};
class Rational {
public:
    const Rational operator*(const Rational& rhs) const;
};
Rational oneHalf(1,2);
result = oneHalf * 2;  // fine (with non-explicit ctor)
result = 2 * oneHalf;  // error (even with non-explicit ctor)

// We need this 
const Rational operator*(const Rational& lhs, const Rational& rhs) {   // avoid friend if possible
    return Rational(lhs.numerator()* rhs.numerator(), lhs.denominator()*rhs.denominator());
}
```
### Things to Remember
* If you need type conversions on all parameters to a funciton(including the one pointed to by the this pointer),
the function must be a non-member.

## Item 25: Consider support for a non-throwing swap

Default swap:
```c++
namespace std {
    template<typename T>
    void swap(T& a, T& b) {
        T tmp = a;
        a = b;
        b = tmp;
    }
}
```
Suppose you want to have your own version of swap.
```c++
class Widget {
public:
    Widget(const Widget& rhs);
    Widget& operator=(cosnt Widget& rhs) {
        ...
        *pImpl = (rhs.pImpl);  // deep copy the real data 
        ...
    }
private:
    WidgetImpl* pImpl;  // ptr to object with Widget's data
};

namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b) {
        swap(a.pImpl, b.pImpl);   // Just swap the ptrs. But cannot compile since pImpl is private
    }
}
```
The most common way to do this (consistent with STL containers) is to provide both public swap member function
and specializations of std::swap that call these member functions.
```c++
class Widget {
public:
    void swap(Widget& other) {
        using std::swap;  // The need for this declaration is explained later
        swap(pImpl, other.pImpl);
    }
    ...
};

namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b) {
        a.swap(b);    // call their swap member function
    }
}
```
Then, what about template class?
```c++
template<typename T>
class WidgetImpl {...};
template<typename T>
class Widget {...};

namespace std {
    template<typename T>
    void swap<Widget<T>>(Widget<T>& a, Widget<T>& b) {  // error!
        a.swap(b);
    }
}
```
Though c++ allows partial specialization of class templates, it doesn't allow it for function
templates. So the above code won't compile.

When you want to "Partially specialize" a function template, the usual approach is to simply add
and overload:
```c++
namespace std {
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b) {
        a.swap(b);
    }
};
```
However, it's not good to add overloading function into std namespace.

So, what to do?

Add it into the same namespace as your class:
```c++
namespace WidgetStuff {
    template<typename T>
    class Widget {...};

    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b) {
        a.swap(b);
    }
}
```
According to the name lookup rules, C++ will find the Widget-specific version in WidgetStuff.
Which is exactly what we want.

Suppose you're writing a function template where you need to swap the values of two objects.
```c++
template<typename T>
void doSomething(T& obj1, T& obj2) {
    using std::swap;  // make std::swap available
    swap(obj1, obj2);
}
```
The name lookup rules ensure that this will find any T-specific swap at global scope or in the same
namespace as the type T. If no T-specific swap exists, compiler will use swap in std,
thanks to the using declaration that makes std::swap visable in this function.

Don't do this:
```c++
std::swap(obj1, obj2);  // force compiler to consider only the swap in std
```

### Things to Remember
* Provide a swap member function when std::swap would be inefficient for your type.
Make suer your swap doesn't throw exceptions.
* If you offer a member swap, also offer a non-member swap that calls the member.
For classes (not templates), specialize std::swap, too.
* When calling swap, employ a using declaration for std::swap, then call swap without namespace qualification.
* It's fine to totally specialize std templates for user-defined types, but never try to add something
completely new to std.
