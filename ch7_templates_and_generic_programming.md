# Ch5: Templates and Generic Programming

## Item 41: Understand implicit interfaces and compile-time polymorphism

OOP: explicit interfaces and runtime polymorphism.

Template: implicit interfaces and compile-time polymorphism.

### Things to Remember
* Both classes and templates support interfaces and polymorphism.
* For classes, interfaces are explicity and centered on function signatures. Polymorphism occurs at runtime through virtual functions.
* For template parameters, interfaces are implicit and based on valid expressions.
Polymorphism occurs during compilation through template instantiation and function overloading resolution.

## Item 42: Understand the two meanings of typename

```c++
template<class T> class Widget;
template<typename T> class Widget;
```
They are the same. May prefer typename.

Nested dependent name

```c++
template<typename C>
void print2nd(const C& container) {
    if (container.size() >= 2) {
        typename C::cosnt_iterator iter(container.begin());   // typename here
        ++iter;
        int value = *iter;
        std::cout << value;
    }
}

```

```c++
C::const_iterator* x;
```
You may assume that it's a C::const_iterator pointer to x, but const_iterator may be a static data member of C, then
the \* means multiplication.

So the Rule is "typename must precede nested dependent type names".

If the parser encounters a nested dependent name in a template, it assumes that the name is not a type unless you tell it otherwise.

The exception is: 
typename is not needed in a list of base classes or as a base class identifier.
```c++
template<typename T>
class Derived: public Base<T>::Nested {   // typename not allowed
public:
    explicit Derived(int x):Base<T>::Nested(x){  // typename not allowed
        typenamebase<T>::Nested temp; // typename required
    }
};
```

Be familiar with "typdef typename".

```c++
typedef typename std::iterator_traits<IterT>::value_type value_type;
value_type temp(*iter);
```

### Things to Remember
* When declaring template parameters, class and typename are interchangeable.
* Use typename to identify nested dependent type names, except in base class lists or as a base class identifier
in a member initialization list.

## Item 43: Know how to access names in templatized base classes

```c++
class CompanyA {
public:
    void sendClearText(cosnt std::string& msg);
    void sendEncrypted(cosnt std::string& msg);
    ...
};
class CompanyB {
public:
    void sendClearText(cosnt std::string& msg);
    void sendEncrypted(cosnt std::string& msg);
    ...
};

class MsgInfo {...};

template<typename Company>
class MsgSender {
public:
    ...
    void sendClear(const MsgInfo& info) {
        std::string msg;
        // create msg from info
        Company c;
        c.sendClearText(msg);
    }
};

// A derived class
template<typename Company>
class LoggingMsgSender: public MsgSender<Company> {
public:
    void sendClearMsg(const MsgInfo& info) {  // good to have a different name with baseclass, Item 33, 36
        // write "before sending" info to the log
        sendClear(info);  // won't compile
        // write "after sending" info to the log
    }
};
```

The problem is that when compilers encounter the definition for the class template LoggingMsgSender, they don't know 
what class it inherits from. Sure, it's MsgSender<Company>, but Company is template parameter, one that won't be
known until later.

```c++
class CompanyZ {  // offers no sendClearText function
public:
    void sendEncrypted(const std::string& msg);
};

// total template specialization
template<>
class MsgSender<CompanyZ> {
public:
    void sendSecret(const MsgInfo& info);
};
```
That's why C++ rejects the call: it recognizes that base class templates may be specialized and that such specializations
may not offer the same interface as the general template.

Three ways:
```c++
// 1. this->
void sendClearMsg(const MsgInfo& info) {
    this->sendClear(info);  // ok, assumes that sendClear will be inherited
}

// 2. using
using MsgSender<Company>::sendClear;  // tell compilers to assuem that sendClear is in the base class 
void sendClearMsg(const MsgInfo& info) {
    sendClear(info);
}

// 3. explicitlly specify, (However, turn of the virtual binding behavior)
void sendClearMsg(const MsgInfo& info) {
    MsgSender<Company>::sendClear(info);
}
```
Fundamentally, the issue is whether compiller will diagnose invalid references to base class member sooner (when derived
class template definitions are parsed) or later (When those templates are instantiated with specific template arguments).
C++'s policy is to prefer early diagnoses, and that's why it assumes it knows nothing about the contents of base classes when thoses 
classes are instantiated from templates.

### Things to Remember
* In derived class templates, refer to names in base class templates via a "this->" prefix, via using declaration, 
or via an explicit base class qualification.

## Item 44: factor parameter-independent code out of templates

Using templates can lead to code bloat.

```c++
template<typename T, std::size_t n>  // n*n matrics of objects of type T
class SquareMatrix {
public:
    ...
    void invert();
};

SquareMatrix<double, 5> sm1;
smq.invert();
SquareMatrix<double, 10> sm2;
sm2.invert();
```
Two copies of invert will be instantiated here. A classic wayfor template-induced code bloat to arise.

```c++
template<typename T>
class SquareMatrixBase {
protected:
    ...
    void invert(std::size_t matrixSize);
};
template<typename T, std::size_t n>
class SquareMatrix: private SquareMatrixBase<T> {  // private inheritance, not is-a relationship, baseclass only to facilitate the derived classes' implementation
private:
    using SquareMatrix<T>::invert;  // avoid hiding base version of invert; Item 33
public:
    ...
    void invert() {this->invert(n);}  // make inline call to base class version; Item 43
};
```
The versions of invert with the matrix sizes hardwired into them are likely to generate better code than the 
shared version where the size is passed as a function parameter or is stored in the object.

### Things to Remember
* Template generate multiple classes and functions, so any template code not dependent on a template parameter causes bloat.
* Bloat due to non-type template parameters can often be eliminiated by replacing template parameters with functnio parameters 
or class data members.
* loat due to type parameters can be reduced by sharing implementations for instantiation types with identical binary representations.

## ITem 45: Use member function templates to accept "all compatible types"
Pointer allows implicit conversions, what about class template?
```c++
class Top {};
class Middle: public Top {};
class Bottom: public Middle {};
Top* pt1 = new Middle;
Top* pt2 = new Bottom;
const Top* pct2 = pt1;

template<typename T>
class SmartPtr {
    explict SmartPtr(T* realPtr);
    ...
};
SmartPtr<Top> pt1 = SmartPtr<Middle>(new Middle);  // convert SmartPtr<Middle> to SmartPtr<Top>
...
```
Of course, we don't want to write all the overload manually.

We need template constructor.

```c++
template<typename T>
class SmartPtr {
public:
    template<typename U>
    SmartPtr(const SmartPtr<U>& other)
    :heldPtr(other.get()) {}  // will compile only there is an implicit conversion from a U* to T*
private:
    T* heldPtr;
};
```
The utility of member function templates isn't limited to constructors. Another common role for them
is in support for assignment.

```c++
template<class T>
class shared_ptr {
public:
    template<class Y>
    explicit shared_ptr(Y* p);
    template<class Y>
    shared_ptr(shared_ptr<Y> const& r);
    template<class Y>
    explicit shared_ptr(weak_ptr<Y> const& r);
    template<class Y>
    explicit shared_ptr(auto_ptr<Y>& r);

    template<class Y>
    shared_ptr& operator=(shared_ptr<Y> const& r);
    template<class Y>
    shared_ptr& operator=(auto_ptr<Y>& r);
};
```
Member function templates don't change the rules of the language: If a copy ctors is needed and you don't declare one, 
one will be generated fr you automatically.
```c++
template<class T>
class shared_ptr {
public:
    shared_ptr(shared_ptr const& r);
    template<class Y>
    shared_ptr(shared_ptr<Y> const& r);

    shared_ptr& operator=(shared_ptr const& r);
    template<class Y>
    shared_ptr& operator=(shared_ptr<Y> const& r);
};
```

### Things to Remember
* User member function templates to generate functions that accept all compatible types.
* If you declare member templates for generalized copy construction or generalized assignment, you'll still need to 
declare the normal copy constructor and copy assignment operator, too.

## Item 46: Declare non-member functions inside templates when type conversions are desired

```c++
template<typename T>
class Rational {
public:
    Rational(const T& numerator = 0, const T& denominator = 1);
    const T numerator() const;
    const T denominator() const;
};
template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs)
{}
Rational<int> oneHalf(1,2);
Rational<int> result = oneHalf*2;  // won't compile
```
Related to Item 24.

Compilers don't know which function we want to call.

Implicit type conversion functions are never considered during template argument deuction. Such conversions are used
during function class, but before you can call a function, you have to know which functions exist.

Now we're in the template part of C++!

So ?

A friend declaration in a template class can refer to a specific function.

T is always known at the time the class Rational<T> is instantiated.
```c++
template<typename T>
class Rational {
public:
    ...
    friend Rational operator*(const Ration& lhs, const Rational& rhs);

    // The same, T can be omitted
    friend Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs);
};
template<typename T>
const Rational<T> operator*(const Rational<T>& lhs, const Rational<T>& rhs)
{}
```

Won't link!

It's a non-template function declared inside a class template. So you won't be able to define this function
as a template.

```c++
template<typename T>
class Rational {
public:
    ...
    friend Rational operator*(const Rational& lhs, const Rational& rhs) {  // define it inside
        return Rational....
    }
};
```

Ths use of friendship has nothing to do with a need to access non-public parts of the class.

However, it's implicity inline. Avoid this?

```c++
template<typename T> class Rational;
template<typename T>
const Rational<T> doMultiply(const Rational<T>& lhs, const Rational<T>& rhs);

template<typename T>
class Rational {
public:
    ...
    friend Rational operator*(const Rational& lhs, const Rational& rhs) {
        return doMultiply(lhs, rhs);
    }
};

template<typename T>
const Rational<T> doMultiply(const Rational<T>& lhs, const Rational<T>& rhs) {
    ...
}
```
### Things to Remember
* When writing a class template that offers functions related to the template that support implicity type
conversions on all parameters, define those functions as friends inside the class template.

## Item 47: Use traits classes for information about types

```c++
struct input_iterator_tag {};
struct output_iterator_tag {};
struct forward_iterator_tag: public input_iterator_tag {};
struct bidirectional_iterator_tag: public forward_iterator_tag {};
struct random_access_iterator_tag: public bidirectional_iterator_tag {};
```

What we want to do is implement advance like this:
```c++
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d) {
    if (iter is a random access iterator) {
        iter += d;
    }
    else {
        if (d >=0) {while(d--) ++iter;}
        else {while(d++) --iter;}
    }
}
```

```c++
template<typename IterT>
struct iterator_traits;
```
The way iterator_traits works is that for each type IterT, a typedef named iterator_category is declared in the struct
iterator_traits<IterT>.

```c++
template<...>  // template params elided
class deque {
public:
    class iterator {
    public:
        typedef random_access_iterator_tag iterator_category;
        ...
    };
    ...
};

template<...>  // template params elided
class list{
public:
    class iterator {
    public:
        typedef bidirectional_iterator_tag iterator_category;
        ...
    };
    ...
};

template<typename iterT>
struct iterator_traits {
    typedef typename IterT::iterator_category iterator_category;
    ...
};
```
Support pointers
```c++
template<typename iterT>
struct iterator_traits<IterT*> {  // Partial template specialization
    typedef typename IterT::iterator_category iterator_category;
    ...
};
```
We know how to design and implement a traits class:
* Identify some information about types you'd like to make available (e.g. for iterators, their iterator category).
* Choose a name to identify that information(e.g., iterator_category)
* Provide a template and set of specializations(e.g., iterator_traits) that contain the information for the types you
want to support.
```c++
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d) {
    if (typeid(typename std::iteator_traits<IterT>::iterator_category) == typeid(std::random_access_iterator_tag)) {
    }
}
```
Cannot compile and if we can do it in compilation why we need to do it at runtime?

Overload!

```c++
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, Dist d, std::random_access_iterator_tag) {  // use this impl for random access iterators
    iter += d;
}
template<typename IterT, typename DistT>
void doAdvance(IterT& iter, Dist d, std::bidirectional_iterator_tag) {  // use this impl for bidirectional iterators
    if (d >=0) {while(d--) ++iter;}
    else {while(d++) --iter;}
}
...
template<typename IterT, typename DistT>
void advance(IterT7 iter, DistT d) {
    doAdvance(iter, d, typename std::iterator_traits<IterT>::iterator_category());
}

```
Summary of how to use traits class:
* Create a set of overloaded "worker" functions or function templates that differ in a traits parameters.
Implement each function in accord with the traits information passed.
* Create a "master" function or function template that calls the workers, passing information provided by a traits class.

In iterator_traits, there are iterator_category, value_type, char_traits, numeric_limits...

### Things to Remember
* Traits classes make information about types available during compilation. They're implemented using tmeplates
and template specializations.
* In conjunction with overloading, traits classes make it possible to perform compile-time if...else tests on type.

## Item 48: Be aware of template metaprogramming
```c++
template<typename IterT, typename DistT>
void advance(IterT& iter, DistT d) {
    if (typeid(typename std::iteator_traits<IterT>::iterator_category) == typeid(std::random_access_iterator_tag)) {
        iter += d;   // error!
    }
    else {
        if (d >=0) {while(d--) ++iter;}
        else {while(d++) --iter;}
    }
}
```
We'll never try to execute the +=, but
Compiler are obliged to make sure that all source code is valid.

TMP
```c++
template<unsigned n>
struct Factorial {
    enum { value = n*Factorial<n-1>::value; }
};
template<>
struct Factorial<0> {
    enum { value = 1;}
};

int main() {
    std::cout << Factorial<5>::value << std::endl;
    std::cout << Factorial<10>::value << std::endl;
}
```
Why TMP is worth knowing about?
* Ensuring dimensional unit correctness
* Optimizing matrix operations
* Generating custom design patern implementations
### Things to Remember
* Template metaprogramming can shift work from runtime to compile-time, thus enabling earlier error detection and higher
runtime performance.
* TMP can be used to generate custom code based on combinations of policy choices, and it can also be used to avoid generating
code inappropriate for particular types.
