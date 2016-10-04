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

## Item 19: Thread class design as type design

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
