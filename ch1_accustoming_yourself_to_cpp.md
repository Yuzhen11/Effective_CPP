# Ch1: Accustoming yourself to C++

## Item 1: View C++ as a federatoin languages

Today's C++ is a multiparadigm programming language, one supporting a combination of
procedural, object-oriented, functoinal, generic, and metaprogramming features.

Four sublanguages:

1. C

2. Object-Oriented C++

3. Tepmlate C++

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

### Things to Remember

* Declaring something const helps compilers detect usage errors. const can be applied to objects
at any cope, to function parameters and return types, and to member functions as a whole.

* Copmiler enforce bitwise constness, but you should program using conceptual constness.

* When const and non-const member functions have essentially identical implementations,
code duplicatoin can be avoided by having the non-const version call the const version.
