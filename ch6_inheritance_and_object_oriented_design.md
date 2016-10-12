# Ch6: Inheritance and Object-Oriented Design

## Item 32: Make sure public inheritance models "is-a"

Public inheritance means "is-a".

Sometimes it may be difficult. Penguin is a bird but penguin cannot fly.

```c++
class Bird {
public:
    virtual void fly();  // birds can fly
    ...
};
class Penguin: public Bird {  // Penguins are birds
};

// 1
class Bird {  // no fly function
};
class FlyingBird {
public:
    virtual void fly();
    ...
};
class Penguin: public Bird {  // no fly function
};

// 2
void error(const std::string& msg);
class Penguin: public Bird {
    virtual void fly() { error("Attempt to make a penguin fly!"); }
};
```
Better to rejects the fly during compilation than detect than at runtime.

Another example is square is rectangle, but rectangle can have different height and width.

### Things to Remember
* Public inheritance means "is-a". Everything that applies to base classes must also apply to derived class, 
because every derived class object is a base class object.

## Item 33: Avoid hiding inherited names

Derived class name will hide all the same name in the base class.

To make that name available, Two options:
1. `using` statement
2. Explicitly forward to base class

### Things to Remember
* Names in derived classes hide names in base classes. Under public inheritance, this is never desirable.
* To make hidden name visible again, employing using declarations or forwarding functions.

## Item 34: Differentiate between inheritance of interface and inheritance of implementation
```c++
class Shape {
public:
    virtual void draw() const = 0;
    virtual void error(const std::string& msg);
    int objectID() const;
};
class Retangle: public Shape {};
```

* Member function interfaces are always inherited. As explained in Item 32, public inheritance means is-a, so anything
that is true of a base class must be true of its derived classes. Hence, if a function applies to a class, it must also apply
to its derived classes.

* The purpose of declaring a pure virtual function is to have derived classes inherit a function interface only.
* The purpose of declaring a simple virutal function is to have derived classes inherit a function interface
as well as a default implementation.

When adding a new derived class, maybe forget to customize the virtual function....
```c++
class Airport {};
class Airplane {
public:
    virtual void fly(const Airport& dest) {
        ...
    }
};
class ModelA: public Airplane {...};
class ModelB: public Airplane {...};
```
What about we add ModelC but need a customized fly and forget to redefine the fly function?
```c++
class Airplane {
public:
    virtual void fly(const Airport& dest) = 0;
protected:
    void defaultFly(const Airport& dest) {
        // default code for flying...
    }
};
class ModelA: public Airplane {
public:
    virtual void fly(const Airport& dest) {
        defaultFly(dest);
    }
};
class ModelB: public Airplane {
public:
    virtual void fly(const Airport& dest) {
        defaultFly(dest);
    }
};
class ModelC: public Airplane {
public:
    virtual void fly(const Airport& dest) {
        // It's own version here
    }
};
```
Some people object to the idea of having separate functions for providing interface and default inplementation,
such as fly and defaultfly.

It pollutes the class namespace.

Other solution?

Pure virtual functions must be redeclared in concrete derived classes, but they may also have implementations of
their own.
```c++
class Airplane {
public:
    virtual void fly(cosnt Airport& dest) = 0;
};
void Airplane::fly(const Airport& dest) {
    // default fly ...
}

class ModelA: public Airplane {
public:
    virtual void fly(const Airport& dest) {
        Airplane::fly(dest);
    }
}
class ModelB: public Airplane {
public:
    virtual void fly(const Airport& dest) {
        Airplane::fly(dest);
    }
}
class ModelC: public Airplane {
public:
    virtual void fly(const Airport& dest) {
        // it's own version
    }
}
```
* The purpose of declaring a non-virtual function is to have derived classes inherit a function interface as well as
a mandatory implementation.

### Things to Remember
* Inheritance of interface is different from inheritance of implementation. Under public inheritance, derived classes
always inherit base class interfaces.
* Pure virtual functions specify inheritance of interface only.
* Simple (inpure) virtual functions specify inheritance of interface plus inheritance of a default implementation.
* Non-virtual functions specify inheritance of interface plus inheritance of a mandatory implementation.

## Item 35: Consider alternatives to virtual functions
```c++
class GameCharacter {
public:
    virtual int healthValue() const;  // return character's health rating; derived classes may redefine this
};
```
### The Template Method Pattern via the Non-virtual interface Idiom

A school of thought that argues that virtual functions should almost always be private.

non-virtual interface (NVI) idiom

```c++
class GameCharacter {
public:
    int healthValue() const {
        ...  // do before stuff
        int retVal = doHealthValue();
        ...  // do after stuff
        return retVal;
    }
private:
    virtual int doHealthValue() const {  // derived class may redefine this
        ...
    }

};
```
Can do before stuff and do after stuff.

C++ allow derived classes redefine private inherited virtual functions.

###  The Strategy Pattern via Function Pointers
```c++
class GameCharacter; // forward declaration

int defaultHealthCalc(const GameCharacter& gc);
class GameCharacter {
public:
    typedef int (*HealthCalcFunc)(const GameCharacter&);
    explicit GameCharacter(HealthCalcFunc hcf = defaultHealthCalc):healthFunc(hcf){}

    int healthValue() const {
        return healthFunc(*this);
    }
private:
    HealthCalcFunc healthFunc;
};

class EvilBadGuy: public GameCharacter {
public:
    explicit EvilBadGuy(HealthCalcFunc hcf = defaultHealthCalc):GameCharacter(hcf) {}
};

int loseHealthQuickly(cosnt GameCharacter&);
int loseHealthSlowly(cosnt GameCharacter&);

EvilBadGuy ebg1(loseHealthQuickly);
EvilBadGuy ebg1(loseHealthSlowly);
```

1. Different instances of the same character type can have different health calculation functions.
2. The function can be changed at runtime.

###  The Strategy Pattern via std::function

Similar...

### The classic Strategy Pattern
```c++
class GameCharacter;
class HealthCalcFunc {
public:
    virtual int calc(const HealthCalcFunc& gc) const {}
};
```
HealthCalcFunc defaultHealthCalc;

class GameCharacter {
public:
    explicit GameCharacter(HealthCalcFunc* phcf = &defaultHealthCalc):pHealthCalc(phcf){}
    int healthValue() const {
        return pHealthCalc->calc(*this);
    }
private:
    HealthCalcFunc* pHealthCalc;
};

### Things to Remember
* Alternatives to virtual functions include the NVI idiom and various forms of the strategy desgin pattern.
The NVI idiom is itself example of the Template Method Design Pattern.
* A disadvantage of moving functionality from a member function to a function outside the class is that the
non-member function lacks access to the class's non-public members.
* std::function objects act like generalized function pointers. Such objects support all callable entities
compatible with a given target signature.

## Item 36: Never redefine an inherited non-virtual function

```c++
class B{};
class D: public B{};

D x;
B* pB = &x;
pB->mf();
D* pD = &x;
pD->mf();  // these two mf should be the same
```

* Everything that applies to B objects also applies to D objects, because every D object is-a B object.

### Things to Remember
* Never redefine an inherited non-virtual function

## Item 37: Never redefine a function's inheritance default parameter value
### Things to Remember

* Never redefine an inherited defalut parameter value, because default parameter values are statically bound,
while virtual functions - the only functions you should overriding - are dynamically bound.

## Item 38: Model "has-a" or "implemented-in-terms-of" through composistion
```c++
class Person {
public:
    ...
private:
    std::string name;   // composed object, has-a
    Address address;
    ...
};
```
```c++
template<class T>
class Set {
public:
    void insert(const T& item);
    ...
private:
    std::list<T> rep;  // representation, implemented in terms of
};
```
### Things to Remember
* Composition has meanings completely different from that of public inheritance.
* In the application domain, composition means has-a. In the implementation domain, it means is-implemented-in-terms-of.

## Item 39: Use private inheritance judiciously
```c++
class Timer {
public:
    explict Timer(int tickFrequency);
    virtual void onTick() const;  // automatically called for each tick
};
```
It's not true that Widget is-a Timer. Widget clients shouldn'e be able to call onTick on a Widget, because it's not 
part of the conceptual Widget interface.

We thus inherit privately:
```c++
class Widget: private Timer {
private:
    virtual void onTick() const;
};
```
Private inheritance isn't strictly necessary.
```c++
class Widget {
private:
    class WidgetTimer: public Timer {
    public:
        virtual void onTick() const;
        ...
    };
    WidgetTimer timer;
};
```
Two advantages:
1. You might want to design widget to allow for derived classes, but prevent dervied classes from redefining onTick.
2. Minimize Widget's compilation dependencies.

The edge case is edgy indeed: it applies only when you're dealing with a class that has no data in it.
```c++
class Empty {};
class HoldsAnInt {
private:
    int x;
    Empty e;  // shoud require no memory
};
class HoldsAnInt: private Empty {
private:
    int x;
};
```
Empty base optimization (EBO)

### Things to Remember
* Private inheritance means is-implemented-in-terms-of. It's usually inferior to composition, but it makes sense
when a derived classs need to redefine inherited virtual functions.
* Unlike composition, private inheritance can enable the empty base optimization. This can be important for library
developers who strive to minimize object sizes.

## Item 40: Use multiple inheritance judiciously

In a diamond shape inheritance hierarchy, the default is to perform replication for base class data.

If you want only one copy, you need to use virtual base class. To do that, you have all classes that immediately
inherit from it use virtual inheritance.

### Things to Remember
* Multiple inheritance is more complex than single inheritance. It can lead to new ambiguity issues and to the need for 
virtual inheritance.
* Virtual inheritance imposes costs in size, speed, and complexitiy of initialization and assignment. It's most 
practical when virtual base classes have no data.
* Multiple inheritance does have legitimate uses. One scenario involves combining public inheritance from an Interface
class with private inheritace from a class that helps with implementation.
