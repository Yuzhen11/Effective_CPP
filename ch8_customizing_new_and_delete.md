# Ch8: Customizing new and delete

```c++
T* p = new T;
// The process is loosely equivalent to the following
void* addr = T::operator new(sizeof(T));
T* p = new (addr) T;
```
First obtain memory then constructs the object in that memory.

## Item 49: Understand the behavior of the new-handler

Before operator new throws an exception in response to an unsatisfiable request for memory, it calls a client-specifiable
error-handling function called a new-handler.

```c++
namespace std {
typedef void(*new_handler)();
new_handler set_new_handler(new_handler p) throw();
}
```
The parameter is a pointer to the function operator new should call if it can't allocate the requested memory.
The return value is a pointer to the function in effect for that purpose before set_new_handler was called.

You use set_new_handler like this:
```c++
void outOfMem() {
    std::cerr << "Unable to satisfy request for memory\n";
    std::abort();
}
int main() {
    std::set_new_handler(outOfMem);
    int* pBigDataArray = new int[100000000L];
}
```
When operator new is unable to fulfill a memory request, it calls the new-handler function repeatedly until it can find 
enough memory.

A well-designed new-handler function must do one of the following:
* Make more momery available
* Install a different new-handler
* Deinstall the new-handler
* Throw an exception
* Not return

Supposes you want to handle memory allocaiton failures for the Widget class.

```c++
class Widget {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void operator new(std::size_t size) throw(std::bad_alloc);
private:
    static std::new_handler currentHandler;
};
std::new_handler Widget::currentHandler = 0;

// similar to standard version
std::new_handler Widget::set_new_handler(std::new_handler p) throw() {
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}

// The Widget's operator new will do:
// 1. install new-handler
// 2. call the global operator new to perform the actual memory allocation
// 3. if fails, restore the global new-handler automatically

// RAII resource-handling class
class NewHandlerHolder {
public:
    explicit NewHandlerHolder(std::new_handler nh):handler(nh) {}  // acquire current new-handler
    ~NewHandlerHolder() { std::set_new_handler(handle); }  // release it
private:
    std::new_handler handler;  // remember it
    NewHandlerHolder(const NewHandlerHolder&);   // prevent copying
    NewHandlerHolder& operator=(const NewHandlerHolder&);
};

// new
void Widget::operator new(std::size_t size) throw(std::bad_alloc) {
    // this is very important, install the new one and store the old one
    NewHandlerHolder h(std::set_new_handler(currentHandler));  // install handler
    return ::operator new(size);  // allocate or throw
}

void outOfMem();
Widget::set_new_handler(outOfMem);  // set
Widget* pw1 = new Widget;  // if memory allocation fails, call outOfMem
std::string* ps = new std::string;  // if memory allocation fails, call the global new-handling function
Widget::set_new_handler(0);  // set
Widget* pw2 = new Widget;  // if memory allocation fails, throw an exception
```

How to resuse it? To create a "mixin-style" base class.
A base class that's designed to allow derived classes to inherit a single specific capability.

Turn the base class into a template, so that you get a different copy of the class data for each inheriting class.

```c++
template<typename T>
class NewHandlerSupport {
public:
    static std::new_handler set_new_handler(std::new_handler p) throw();
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    ...
private:
    static std::new_handler currentHandler;
};
template<typename T>
std::new_handler NewHandlerSupport<T>::set_new_handler(std::new_handler p) throw() {
    std::new_handler oldHandler = currentHandler;
    currentHandler = p;
    return oldHandler;
}
template<typename T>
void NewHandlerSupport<T>::operator new(std::size_t size) throw(std::bad_alloc) {
    // this is very important, install the new one and store the old one
    NewHandlerHolder h(std::set_new_handler(currentHandler));  // install handler
    return ::operator new(size);  // allocate or throw
}
template<typename T>
std::new_handler NewHandlerSupport<T>::currentHandler = 0;

class Widget: public NewHandlerSupport<Widget> {
    ...
};
```
Widget inherits from NewHandlerSupport<Widget> may look strange. We need a copy of NewHandlerSupport for each 
class that inherits from NewHandlerSupport.

Called curiously recurring template pattern(CRTP) or "Do It For Me".

nothrow new may throw because it may contain other throwable new.
```c++
Widget* pw = new (std::nothrow) Widget;
```

### Things to Remember
* set_new_handler allows you to specify a funciton to be called when memory allocation requests cannot be satisfied.
* Nothrow new is of limited utility, because it applies only to memory allocaiton; subsequent constructor calls may still throw exceptions.

## Item 50: Understand when it makes sense to replace new and delete

Why would anybody want to replace the compiler-provided versions of operator new or operator delete in the first place?
* To detect usage errors
    operator new keeps a list of allocated addresses and operator delete removes addresses from the list
* To improve efficiency
    optimized for your case
* To collect usage statistics

More:
* To reduce the space overhead of default memory management
* To compensate for suboptimal alignment in the default allocator
* To cluster related objects near one another
* To obtain unconventional behavior
        
### Things to Remember
* There are many valid reasons for writing custom versions of new and delete, including improving performance, 
debugging heap usage errors and collecting usage information.

## Item 51: Adhere to convention when writing new and delete

pseudocode for a non-member operator new:
```c++
void* operator new(std::size_t size) throw(std::bad_alloc) {
    using namespace std;
    if (size == 0) {
        size = 1;
    }
    while (true) {
        // attempt to allocate size bytes
        if (the allocation was successful)
            return (a pointer to the memory);

        // allocation was unsuccessful; find out what the 
        // current new-handling functions is
        new_handler globalHandler = set_new_handler(0);
        set_new_handler(globalHandler);

        if (globalHandler) (*globalHandler)();
        else throw std::bad_alloc();
    }
}
```
Handle requests for zero bytes.

Using set_new_handler to get the funciton.

An infinite loop to allocate memory.

operator new in a base class will be called to allocate memory for an object of a derived class.
```c++
class Base {
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc);
    ...
};
class Derived: public Base {   // Derived doesn't declare operator new
};
Derived* p = new Derived;  // calls Base::operator new!
```
Slough off calls requesting the "wrong amount of memory to the standard operator new".
```c++
void* Base::operator new(std::size_t size) throw(std::bad_alloc) {
    if (size != sizeof(Base))  // if size is wrong, have standard operator new handle the reqeust
        return ::operator new(size);
}
```
### Things to Remember
* operator new should contain an infinite loop trying to allocate memory, should call the new-handler if it can't 
satisfy a memory request, and should handle requests for zero bytes. Class-specific versions should handle
requests for larger blocks than expected.
* operator delete should do nothing if passed a pointer that is null. Class-specific versions should handle 
blocks that are larger than expected.

## Item 52: Write placement delete if you write placement new

```c++
Widget* pw = new Widget;
```
Two functions are called: one to operator new to allocate memory, a second to Widget's default constuctor.

Suppose the first call succeeds, but the second call results in an exception, runtime system needs to handle.

The normal operator new corresponds to normal operator delete:
```c++
void* operator new(std::size_t) throw(std::bad_alloc);
void operator delete(void* rawMemory) throw();

void operator delete(void* rawMemory, std::size_t size) throw(); // typical normal signature at class scope
```

When an operator new function takes extra parameters (other than the mandatory size_t argument), that function
is known as a placement version of new.

A particular useful placement new is the one that takes a pointer specifying where an object should be constructed.
```c++
void* operator new(std::size_t, void* pMemory) throw();
```
This new is used inside std::vector to create objects in the vector's unused capacity.

The rule is simple: if an operator new with extra parameters isn't matched by an operator delete
with the same extra parameters, no operator delete will be called if a memory allocation by the new
needs to be undone.

```c++
class Widget {
public:
    static void* operator new(std::size_t size, std::ostream& logStream) throw(std::bad_alloc);
    static void operator delete(void* pMemory) throw();
    static void operator delete(void* pMemory, std::ostream& logStream) throw();
};
Widget* pw = new (std::cerr) Widget;  // no leak this time

delete pw;  // invokes the normal operator delete
```
Placement delete called only if an exception arises from a constructor call that's coupled
to a call to a placement new.
Applying delete to a pointer never yields a call to a placement version of delete.

Be careful to avoid having class-specific news hide other news, including the normal version.

```c++
class Base {
public:
    // this new hides the normal global forms
    static void* operator new(std::size_t size, std::ostream& logStream) throw(std::bad_alloc);
};
Base* pb = new Base; // error! the normal form of operator new is hidden
Base* pb = new (std::cerr) base;  // fine, call Base's placement new

// Similarly, operator news in derived classes hide both global and inherited
versions of operator new

class Derived: public Base {
public:
    static void* operator new(std::size_t size) throw(std::bad_alloc); // redeclares the normal form of new
    ...
};
Derived* pd = new (std::clog) Derived;   // error! Base's placement new is hidden
Derived* pd = new Derived;
```

Item 33 discusses this kind of name hiding in considerable details.

C++ offers the following forms of operator new at global scope:
```c++
void operator new(std::size_t) throw(std::bad_alloc) // normal new
void operator new(std::size_t, void*) throw() // placement new
void operator new(std::size_t, const std::nothrow_t&) throw() // nothrow new
```

If you declare any operator news in a class, you'll hide all these standand forms.
Be sure to make them available as well as operator delete.

An easy way to do this is to create a base class containing all the normal forms of new and delete:
```c++
class StandardNewDeleteForms {
public:
    // normal new/delete
    static void operator new(std::size_t) throw(std::bad_alloc)
    { return ::operator new(size); }
    static void operator delete(void* pMemory) throw()
    { ::operator delete(pMemory); }

    // placement new/delete
    static void operator new(std::size_t, void* ptr) throw()
    { return ::operator new(size, ptr); }
    static void operator delete(void* pMemory, void* ptr) throw()
    { ::operator delete(pMemory, ptr); }

    // nothrow new/delete
    static void operator new(std::size_t, const std::nothrow_t& nt) throw()
    { return ::operator new(size, nt); }
    static void operator delete(void* pMemory, const std::nothrow_t&) throw()
    { ::operator delete(pMemory); }
};
class Widget: public StandardNewDeleteForms {
public:
    using StandardNewDeleteForms::operator new;  // make those forms visible
    using StandardNewDeleteForms::operator delete;

    static void* operator new(std::size_t size, std::ostream& logStream) throw(std::bad_alloc);  // add custom placement new
    static void operator delete(void* pMemory, std::ostream& logStream) throw(); // add corresponding placement delete
};
```

### Things to Remember
* When you write a placement version of operator new, be sure to write the corresponding
placement version of operator delete. If you don't, your program may experience subtle,
intermittent memory leaks.
* When you declare placement versions of new and delete, be sure not to unintentionally hide the 
normal versions of those functions.
