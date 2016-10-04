# Ch3: Resource Management

## Item 13: Use objects to manage resources

```c++
void f() {
    Investment* pInv = createInvestment();  // call factory function
    ...   // something bad may happen
    delete pInv;  // release object
}
```
Put that resource inside an object whose destructor will automatically release the resource when control leaves f.
```c++
void f() {
    std::auto_ptr<Investment> pInv(createInvestment());  // call factory function
}
```
auto_ptrs have an unusual characteristic: copying them sets them null.
```c++
std::auto_ptr<Investment> pInv1(createInvestment);
std::auto_ptr<Investment> pInv2(pInv1);  // pInv2 now points to the object; pInv1 is now null
pInv1 = pInv2;  // now pInv1 points to the object, and pInv2 is null
```
shared_ptr behaves well.
```c++
std::tr1::shared_ptr<Investment> pInv1(createInvestment);
std::tr1::shared_ptr<Investment> pInv2(pInv1);  // both pInv1 and pInv2 now point to the object
pInv1 = pInv2;  // ditto
```

Two critical aspects of using objects to managing resources:

* Resources are acquired and immediately turned over to resource-managing objects.
* Resource-managing objects use their destructors to ensure that resources are released.

### Things to Remember
* To prevent resource leaks, use RAII objects that acquire resource in their constructors and release 
them in their destructors.
* Two commonly useful RAII classes are tr1::shared_ptr and auto_ptr. tr1::shared_ptr is usually the better choice,
because its behavior when copied is intuitive. copying an auto_ptr sets it to null.

## Item 14: Think carefully about copying behavior in resource-managing classes

In previous item, we see that heap-based resources can be managed using shared_ptr. Not all resources are heap-based.

C API:
```c++
void lock(Mutex* pm);   // lock mutex pointed to by pm
void unlock(Mutex* pm);  // unlock the mutex
```
RAII: Resources are acquired during construction and released during destruction.
```c++
class Lock {
public:
    explicit Lock(Mutex* pm): mutexPtr(pm) { lock(mutexPtr); }  // acquire resource
    ~Lock() {unlock(mutexPtr);}  // release resource
private:
    Mutex* mutexPtr;
};

Mutex m;
{
    Lock ml(&m);
}
```
What about the copying behavior?
```c++
Lock ml1(&m);
Lock ml2(&ml1);  // copy ml1 to ml2, what should happen here?
```
* Prohibit copying
```c++
class Lock : private Uncopyable {  // see Item 6
public:
    ...
};
```
* reference-count the underlying resource
Copying the RAII object should increment the count of the number of objects.

deleter should be defined
```c++
class Lock {
public:
    explicit Lock(Mutex* pm): mutexPtr(pm, unlock)  // deleter added
    { lock(mutexPtr.get()); }
private:
    std::shared_ptr<Mutex> mutexPtr;
};
```

* Copying the underlying resource

Make sure a deep copy. For example, STL.

* Transfer ownership of the underlying resource

Example: auto_ptr

### Things to Remember
* Copying an RAII object entails copying the resource it manages, so the copying behavior of the resource determines
the copying behavior of the RAII object.
* Common RAII class copying behaviors are disallowing copying and performing reference counting, but other
behaviors are possible.

## Item 15: Provide access to raw resources in resource-managin classes

Having an explicit get function like shared_ptr

```c++
class Font {
public:
    explicit Font(FontHandle fh):f(fh) {}
    ~Font() { releaseFont(f); }
    FontHandle get() const { return f; }  // explicit conversion function
private:
    FontHandle f;  // the raw font resource
};

changeFontSize(f.get(), newFontSize);
```

Or implicity conversion
```c++
class Font {
public:
    operator FontHandle() const { return f; }  // implicit conversion
};

changeFontSize(f, newFontSize);

Font f1(getFont());
FontHandle f2 = f1;  // oops
```

### Things to Remember
* APIs often require access to raw resources, so each RAII class should offer a way to get at the resource it manages.
* Access may be via explicit conversion or implicit conversion. In general, explicit conversion is safer,
but implicit conversion is more convenient for clients.

## Item 16: Use the same form in corresponding uses of new and delete

### Things to Remember

* If you use [] in a new expression, you must use [] in the corresponding delete expression. If you don't use []
in a new expression, you mustn't use [] in the corresponding delete expression.

## Item 17: Store newed objects in smart pointers in standalone statements

```c++
processWidget(std::shared_ptr<Widget>(new Widget), priority());
```
The order may be:
1. Execute `new Widget`

2. Call priority

3. Call the shared_ptr constructor

Exception may be thrown between 2 and 3.

```c++
std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority());  // this call won't leak
```

### Things to Remember
* Store newed objects in smart pointers in standalone statements. Failure to do this can lead to 
subtle resource leaks when exceptions are thrown.
