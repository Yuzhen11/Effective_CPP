# Ch2: Constrcutors, Destructors, and Assignment Operators

## Item 5: Know what functions C++ silently writes and calls

```c++
class Emtpy {};
// Essentially the same
class Empty {
public:
    Empty() {...}   // default constructor
    Empty(const Empty& rhs) {...}  // copy constructor
    ~Empty() {...}  // destructor
    
    Empty& operator=(const Empty& rhs) {...}  // copy assignment operator
};

Empty e1;  // default constructor
Empty e2(e1);  // copy constructor
e2 = e1;   // copy assignment operator
```
The generated destructor is non-virtual unless it's for a class inheriting from a base class that itself declares a 
virtual destructor.

The copy constructor and the copy assignment operator compiler generated simply copy each non-static data memnber.

If a class has members like reference or const, the copy assignment operator won't be generated.

C++ refuses to compile the code:
```c++
template<class T>
class NamedObject {
public:
    NamedObject(std::string& name, const T& value);
private:
    std::string& nameValue;
    const T objectValue;
};

std::string newDog("Persephone");
std::string oldDog("Satch");
NamedObject<int> p(newDog, 2);
NamedObject<int> s(oldDog, 2);
p = s;  // what should happen to the data members in p?
```

### Things to Remember

* Compilers may implicitly generate a class's default constructor, copy constructor, copy assignemnt operator, adn destructor.

## Item 6: Explicitly disallow the use of compiler generated functions you do not want

Ways to prevent object to be copied.

```c++
class HomeForSale {...};
HomeForSale h1;
HomeForSale h2;

HomeForSale h3(h1);  // should not compile
h1 = h2;  // should not compile
```

Solution 1: Make it private and no definition
```c++
class HomeForSale {
public:
  ....
private:
    HomeForSale(const HomeForSale&);  // declare only
    HomeForSale& operator=(const HomeForSale&);
};
```
Compiler will prevent the copy intention and linker will complain if you try to invoke them in a member or a friend function.

Solutoin2: Inherit from uncopyable baseclass

```c++
class Uncopyable {
protected:
  Uncopyable() {}  // allow constructoin and destuction of derived objects....
  ~Uncopyable() {}
private:
  Uncopyable(const Uncopyable&);   // but prevent copying
  Uncopyable& operator=(const Uncopyable&);
};

class HomeForSale:private Uncopyable {
};
```
The compiler-generated copy operations will try to call their base class couterparts, and those calls will be rejected, becasue
the copying operations are private in the base class.

In C++11, we may use delete functoin.

### Things to Remember
* To disallow functoinality automatically provided by compilers, declare the corresponding member functions private and give
no implementations. Using base class like Uncopyable is one way to do this.

## Item 7: Declare destructors virtual in polymorphic base classes

Otherwise, the derived part of the object won't be cleared correctly. 

However, you shouldn't always to that, for non-polymorphc class without virtual functions, declare destructors non-virtual make the 
object small. virtual pointer will cost space. 

Shouldn't inherit from STL containers.

Abstract classes with pure virtual destructor. 

```c++
class AWOV {  // AWOV = abstract w/o virtuals
public:
  virtual ~AWOV() = 0;
};
AWOV::~AWOV() {}  // definition

```

### Things to Remember
* Polymorphic base classes should declare virtual destructors. If a class has any virutal functions, it should have a virtual destructor.
* Classes not designed to be base casses or not designed to be used polymorphically should not declare virtual destructors.

## Item 8: Prevent exceptions from leaving your destructors

C++ doesn't prohibit destructors from emitting exceptions, but it certainly discourages the practice. 

```c++
class Widget {
public:
    ...
    ~Widget() { ... }
};
void doSomething() {
    std::vector<Widget> v;
}
```
Possible that two simultaneously active exceptions throw. 

C++ does not liek destructors that emit exceptions!

```c++
class DBConnection {
public:
    ...
    static DBConnection create();
    void close()'   // close connection; throw an exception if closing fails
};
class DBConn {
public:
    ~DBConn() {
        db.close();
    }
private:
    DBConnection db;
};
```
What if the destructor throws exceptions?

here are two primary ways to avoid the trouble.

1. Terminate the program if close throws
```c++
DBConn::~DBConn() {
    try {db.close();}
    catch(...) {
        make log entry that the call to close failed
        std::abort();
    }
}
```
2. Swallow the exception
```c++
DBConn::~DBConn() {
    try {db.close();}
    catch(...) {
        make log entry that the call to close failed
    }
}
```

Neither of these approaches is especially appealing. The problem with both is that the program has no way to react
to the condition that led to close throwing an exception in the first place.

A better strategy is to design DBConn's interface so that its clients have an opportunity to react to problems that may arise.

Could offer a close function itself. thus giveing clients a chance to handle exceptions arising from that operation.

It could also keep track of wheter its DBConnection had been closed, closing it itself in the destructor if not. That would prevent leaking. 

If the call to close were to fail in DBConnection destructor, we'd be back to terminating or swallowing:

```c++
class DBConn {
public:
    void close() {
        db.close();
        closed = true;
    }
    ~DBConn() {
        if (!closed) {
            try {db.close();}
            catch(...) {
                make log entry that call to close failed
                ...
            }
        }
    }
private:
    DBConnection db;
    bool closed;
};
```

Telling cleints to call close themselves doesn't impose a burden on them; it gives them an opportunity to deal with
errors they would otherwise have no chance to react to .

### Things to Remember
* Destructors should never emit exceptions. If functions called in a destructor may throw, the destructor should catch any exceptions,
then swallow them or terminate the program.

* If class clients need to be able to react to exceptions thrown during an operation, the class should provide a regular (i.e., 
non-destructor) function that performs the operation.

## Item 9: Never call virtual functions during construcion or destruction

During base class construction, the derived part is not constructed, so virtual function calls will call the base class one. 

During base class construction of a derived class object, the type of the object is that of the base class. 

The same reasoning applies during destruction.

Solutions:

1. Use non-virtual function to init. 

2. We can pass derived class information by passing arguments up to base class. 

### Things to Remember
* Don't call virutal functions during construction ro destruction, because such calls will never go to a more derived class than that of 
the currently executing constructor or destructor.

## Item 10: Have assignment operators return a reference to \*this

```c++
int x,y,z;
x = y = z = 15;
x = (y = (x = 15));  // the same
```

```c++
class Widget {
public:
    Widget& operator+=(const Widget& rhs) {
        ...
        return *this;
    }
    Widget& operator=(int rhs) {
        ...
        return *this;
    }
};
```

### Things to Remember
* Have assignment operators return a reference to \*this

## Item 11: Handle assignment to self in operator=

```c++
class Widget {...};
Widget w;
...
w = w;

a[i] = a[j];  // potential assignment to self
*px = *py;  // potential assignment to self
```

If you try to manage resources yourself, you can fall into the trap of accidentally releasing a resource before you're done using it. 

```c++
class Bitmap { ... };
class Widget {
    ...
private:
    Bitmap* pb;
};
```

Here's an unsafe implementation:
```c++
Widget& Widget::operator=(const Widget& rhs) {
    delete pb;
    pb = new Bitmap(*rhs.ph);
    return *this;
}
```

Traditional way to prevent this error:

```c++
Widget& Widget::operator=(const Widget& rhs) {
    if (this == &rhs) return *this;
    delete pb;
    pb = new Bitmap(*rhs.ph);
    return *this;
}
```
This version still has exception trouble. What if new yield an exception, the Widget will end up holding a pointer to a deleted Bitmap.

Happily, making operator= exeption-safe typically redners it self-assignment-safe too.

A careful ordering of statements can yield exception-safe.

```c++
Widget& Widget::operator=(const Widget& rhs) {
    Bitmap* pOrig = pb;  // remember the original pb
    pb = new Bitmap(*rhs.pb);
    delete pOrig;  // delete the original pb
    return *this;
}
```
Now, if `new Bitmap` throws an excpetion, pb remains unchanged. This code handle self-assignement too but with less efficient(delete and new).

copy and swap 
```c++
class Widget {
    void swap(Widget& rhs);
};
Widget& Widget::operator=(const Widget& rhs) {
    Widget temp(rhs);
    swap(temp);
    return *this;
}
// A variation, sacrifices clarity at the altar of cleverness. 
Widget& Widget::operator=(Widget rhs) {
    swap(rhs);
    return *this;
}
```
### Things to Remember
* Make sure operator= is well-behaved when an object is assigned to itself. Technqiues include comparing addresses of 
source and target objects, careful statement ordering, and cpoy-and-swap.
* Make suret that any function operating on more than one object behaves correctly if two or more of the objects are the same. 

## Item 12: Copy all parts of an object
1. Copy all members.
2. Don't forget the base class
```c++
class PriorityCustomer: public Customer {
public:
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator=(const PriorityCustomer& rhs);
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs) 
: Customer(rhs),  // invoke base class copy ctor
  priority(rhs.priority) {}
PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs) {
    Customer::operator=(rhs);   // assign base class parts
    priority = rhs.priority;
    return *this;
}
```

### Things to Remember
* Copying functions should be sure to copy all of an objects data members and all of its base class parts. 
* Don't try to implement one of the copying functions in terms of the other. Instead, put common functionaliy in a third 
function that both call.
