# Ch1: Accustoming yourself to C++

## Item 1: View C++ as a federation languages

Today's C++ is a multiparadigm programming language, one supporting a combination of
procedural, object-oriented, functoinal, generic, and metaprogramming features.

Four sublanguages:

1. C

2. Object-Oriented C++

3. Template C++

4. The STL

It's a federation of four sublanguage, each with its own conventions.

Don't be surprise that you need to switch from one sublanguage to another.

### Things to Remember

* rules for effective C++ programming vary, depending on the part of C++ you
are using.

## Item 2: Prefer consts, enums, and inlines to #defines.

Prefer compiler to preprocessor. #define will be handled be preprocessor.

```c++
#define ASPECT_RATIO 1.653
```

The name may not get entered into the symbol table. Debug is hard!

prefer using const

```c++
const double AspectRatio = 1.653;
```

Two sepecial cases:

1. pointer

```c++
const char* const authorName = "Scott Meyers"; // const pointer to const char

// May prefer using std::string
const std::string authorName("Scott Meyers");
```

2. class-specific constants

Take it a member and make it static.

```c++
class GamePlayer {
private:
    static const int NumTurns = 5;  // constant declaration
    int scores[NumTurns];
};
```

For integral type (e.g., integers, chars, bools), as long as you don't take their address, 
you can declare them and use them without providing
a definition. 

If you really want, no initial value is permitted at the point of definition.
```c++
const int GamePlayer::NumTurns;  // definition of NumTurns
``

Older compiler may not accept the syntax above and above syntax only works for
integral types.

```c++
class CostEstimate {
private:
    static const double FudgeFactor;  // declaration, in hpp
};

const double CostEstimate::FudgeFactor = 1.35;  // definition,in cpp
``

But what if you want to know the value during compilatoin of the class (e.g. Define 
the length of an array)?  Use "the enum hack".

```c++
class GamePlayer {
private:
    enum {NumTurns = 5};
    int scores[NumTurns];
};
```

The enum hack is worth konwing about for several resons.

1. In some ways more like a #define than a const does. No legal to take address, no storage.

2. Lots of code employs it.

What about using #define to implement functions?

    #define CALL_WITH_MAX(a,b) f((a)>(b) ? (a):(b))

Remember to parenthesize all the arguments in the macro body.

And werid things may still happen.

```c++
int a = 5, b = 0;
CALL_WITH_MAX(++a, b);  // a is incremented twice
CALL_WITH_MAX(++a, b+10);  // a is incremented once
```
The number of times that a is incremented before calling f depends on what it is being compared with!

inline may be a better choice.

```c++
template<typename T>
inline void callWithMax(const T& a, const T& b) {
    f(a > b ? a : b);
}
```

### Things to Remember

* For simple constants, prefer const objects or enums to #defines.

* For function-like marcos, prefer inline functions to #defines.


## Item 3: Use const whenever possible

const keyword is remarkably versatile.

For pointers, you can specify whether the pointer itself is const, the data it points to is const, both, or neither:

```c++
char greeting[] = "Hello";
char* p = greeting;  // non-const pointer, non-const data
const char* p = greeting;  // non-const pointer, const data
char* const p = greeting;  // const pointer, non-const data
const char* const p = greeting;  // const pointer, const data
```

Declaring an iterator const is like declaring a pointer const (T * const pointer), the iterator isn't allow
to point to something different, but the thing it points to may be modified.
```c++
const std::vector<int>::iterator iter = vec.begin();
*iter = 10; // ok
++iter;  // error

std::vector<int>::const_iterator cIter = vec.begin();
*cIter = 10;  // error
++iter;  // fine
```

Return value can be const. Help to reduce the incidence of client errors without giving up safety or efficiency.

```c++
class Rational { ... };
const Rational operator*(const Rational& lhs, const Rational& rhs);

Rational a, b, c;
(a*b) = c; // error
if (a*b = c) ...    // good to prevent this error, can be identified during compilation
```
We can easily prevent the above mistakes. What we want is `if (a*b == c)`

We can also have const member functions
```c++
class TextBlock {
public:
    const char& operator[](std::size_t position) const 
    { return text[position]; }
    char& operator[](std::size_t position)
    { return text[position]; }
private:
    std::string text;
};

TextBlock tb("Hello");
std::cout << tb[0];  // call non-const operator[]
const TextBlock ctb("World");
std::cout << ctb[0];  // call the const one

tb[0] = 'x';  // ok
ctb[0] = 'x';  // error
```

Will be met when we pass parameter using reference-to-const.

Bitwise const and logical const

Compilers enforce bitwise const, but we need logical const!

```c++
class CTextBlock {
public:
    char& operator[](std::size_t position) const  // inappropriate but bitwise const 
    { return pText[position]; }
};
```

Really want to modify value in const member function? mutable!

Avoid Duplicatoin in const and Non-const Member Functions
```c++
class TextBlock {
public:
    const char& operator[](std::size_t position) const 
    { return text[position]; }
    char& operator[](std::size_t position)
    {
        return const_cast<char&>(static_cast<const TextBlock&>(*this)[position]);
    }
private:
    std::string text;
};
```
But I don't this is a good idea...


### Things to Remember

* Declaring something const helps compilers detect usage errors. const can be applied to objects
at any cope, to function parameters and return types, and to member functions as a whole.

* Copmilers enforce bitwise constness, but you should program using conceptual constness.

* When const and non-const member functions have essentially identical implementations,
code duplicatoin can be avoided by having the non-const version call the const version.


## Item 4: Make sure that objects are initialized before they're used.

In general, if you're in the C part of C++ and initialization would probably incur a runtime cost, initialization is not
guaranteed to take place. If you cross into the non-C part of C++, things somethins change.

The best way to deal with this is to always initialize your objects before you use them.

Prefer initialization list... 

A static object is one that exists from the time it's constructed until the end of the program. Included are global objects, objects
defined at namespace scope, objects declared static inside classes, objects declared static inside functions, and object declared static at file scope.

Static objects inside functions are known as local static objects (Because they're local to a function), and the other kinds of static
objects are known as non-local static objects.

Static objects are automatically destroyed when the program exists.

A translation unit is the source code giving rise to a single object file. It's basically a single source file, plus all of its #include files.

The problem is the relative order of iniitalization of non-local static objects defined in different translatoin units is undefined.

```c++
class FileSystem {
public:
    std::size_t numDisks() const;
};
extern FileSystem tfs;

class Directory {
public:
    Directory(params);
}
Directroy::Directory(params) {
    std::size_t disks = tfs.numDisks();  // use the tfs object
}

Directory tempDir(params);  // Another non-local static object, tfs may be initialized!!!
```

The problem is obvious, it's possible that tempDir is initialized before tfs. 

Use local static object!
```c++
class FileSystem {...};
FileSystem& tfs() {
    static FileSystem fs;
    return fs;
}
class Directory {...};
Directory::Directory(params) {
    std::size_t disks = tfs().numDisks();
}
Directory& tempDir() {
    static Directory td;
    return td;
}
```

Objects will be initialized when you first call the functions.

### Things to Remember
* Manually initialize objects of built-in type, because C++ only sometimes initilizes them itself.
* In a constructor, prefer use of the member initialization list to assignment inside the body of the constructor.
List data members in the initialization list in the same order they're declared in the class.
* Avoid initialization order problems across translatoin units by replacing non-local static objects with local static objects.
