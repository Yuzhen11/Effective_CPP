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

## Item 6: Explicitly disallow the use of compiler generated funcoitns you do not want

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

