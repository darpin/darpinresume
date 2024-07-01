# C++ to me

NOTE: this is a first draft with some initial thoughts.

## Introduction

I first learned about C++ during my undergraduate years studying Math and 
Computer Science at FHSU in the early 1990s. 

Was it love at first sight? Maybe. The language treated me right while writing
my Masters project at Kansas State. Most of my years of professional software
development, C++ has treated me right. I've tried to use, abuse, experiment 
with, play with, learn from, and push the boundaries the language provides 
throughout its and my evolution.

Here, I'll be brief ... hopefully, are some of the things that I've learned
and accomplished with C++ over the years.

## First things first

A lot of times the question is asked: "what version of C++ do you use?"

To be absolutely honest, I don't have a good recollection of when features were
added to the language.

So, yeah, what version of C++ was features XYZ added? I probably can't tell you
but I can likely tell you what goodness it adds.

I'm partial to gcc -- it almost always has been the compiler I've used -- 
because it is mature, it gets lots of love from its developers and community, 
and for things like the atomic operations it provides. See: 
[Built-in Functions for Memory Model Aware Atomic Operations](https://gcc.gnu.org/onlinedocs/gcc/_005f_005fatomic-Builtins.html)

Near-lock-less queues may be more robust when using those built-in functions 
over `std::atomic` (which may or may not use a mutex [it is up to the library
author]).

## Almost Always...

Herb Sutter is well-known for his
[*Almost Always Auto*](https://herbsutter.com/2013/08/12/gotw-94-solution-aaa-style-almost-always-auto/)
guideline.

That guideline says something like: prefer `auto something = int{42};` over
`int something = 42;`. This is a good guideline, except when used for function
declarations(!!!).

The *Almost Always* part is more far-reaching and may be useful for other 
constructs.

Consider *Almost Always Const*. 
That is, prefer constant `const auto something = int{42};` when it is known that 
the value of `something` will never change. 

This does give the compiler some chances to do some magic
with that variable's existence. But...

Must be careful with `const` when the value could be `return`'d.

Suppose have (contrived) something like: 

```cpp
    const auto aLongString = std::string{1'000'000'000, 'a'}`
    // some code doing things
    return SomeStruct{
       0,
       3.14,
       aLongString
    };
```

In general, we'd expect the compiler to *move* the string into the constructed
`SomeStruct` on the return. However, since it is declared `const` the compiler
must make a copy.

Moral of the story with the *Almost Always* guidelines, and some feature of the 
C++ language: you've got to think about what you are doing -- the language isn't 
going to hold your hand and do the right thing for you. I'd argue that this adds 
strength and safety to any solution using C++.

There are potentially other *Almost Always* guidelines to consider, too:

* *Almost Always Algorithm* - prefer `std::for_each(...)` to a 'C-style' 
  for-loop. Why? When the container / collection has a specific implementation
  (which it does) leave it up to the container / collection itself to determine 
  the most efficient way to iterate over the data. 
* *Almost Always C++ Casts* - or - *Almost Never C-Style Casts* - prefer
  `auto* something = static_cast<SomeStruct>(aPtr);` to
  `auto* somthing = (SomeStruct*)aPtr;`
  - exceptions
    * documenting unused parameters and variables `(void)input;` and
      `(void)dummyValue;` which could otherwise be overridden ... whole
      other section of thought with that.

# Polymorphism is Your Friend

A lot of times, especially in serialization code, you might see a set of 
free functions or methods on a serialization class for differing types.

Suppose we have these (contrived) structures:

```cpp
struct SubStruct {
    std::string aValue;
}
struct Struct {
    SubStruct aSubStruct;
    SubStruct anotherSubStruct;
}
struct SuperStruct {
    Struct aStruct;
}
```

Further suppose we want to serialize these into something that looks like
a sort-of-like initializer-list. We have a set of free-functions:

```cpp
std::string SubStructToString(const SubStruct& in)
{
    return { 
        "{ aValue = "s + in.aValue + "}"s
    };
}

std::string StructToString(const Struct& in)
{
    return {
        "{"s +
            "{aSubStruct = "s + SubStructToString(in.aSubStruct) + "},"s +
            "{anotherSubStruct"s + SubStructToString(in.anotherSubStruct)s + "}"s +
        "}"s        
    };
}

std::string SuperStructToString(const SuperStruct& in)
{
    return {
        "{"s + 
            "{aStruct = "s + StructToString(in.aStruct) + "}"s +
        "}"s
    };
}
```

While this is perfectly valid code, let's consider something a bit cleaner.

Same structures, with the following free functions:

```cpp
std::string to_string(const SubStruct& in)
{
    return { 
        "{ aValue = "s + in.aValue + "}"s
    };
}

std::string to_string(const Struct& in)
{
    return {
        "{"s +
            "{aSubStruct = "s + SubStructToString(in.aSubStruct) + "},"s +
            "{anotherSubStruct"s + SubStructToString(in.anotherSubStruct)s + "}"s +
        "}"s        
    };
}

std::string to_string(const SuperStruct& in)
{
    return {
        "{"s + 
            "{aStruct = "s + StructToString(in.aStruct) + "}"s +
        "}"s
    };
}
```

Arguably a lot cleaner. Let's make it less arguably.

Consider the following collections:

```cpp
using SubStructVector = std::vector<SubStruct>;
using SubStructArray = std::array<SubStruct>;
```

To serialize those with free functions expecting the types we'd have:

```cpp
std::string SubStructVectorToString(const SubStructVector& collection)
{
    std::string out = "{"s;
    std::transform(std::begin(collection), 
                   std::end(collection),
                   std::back_inserter(out),
                   [&out](const auto& entry)
                   {
                       entry.push_back("{"s + to_string(entry) + "},"s);
                   });
    out += "}";
    return out;                   
}

std::string SubStructArrayToString(const SubStructVector& collection)
{
    std::string out = "{"s;
    std::transform(std::begin(collection), 
                   std::end(collection),
                   std::back_inserter(out),
                   [&out](const auto& entry)
                   {
                       entry.push_back("{"s + to_string(entry) + "},"s);
                   });
    out += "}";
    return out;                   
}
```

You should notice that the bodies of those two functions are identical.

Consider:
```cpp
template <typename COLLECTION>
std::to_string(const COLLECTION& collection)
{
    std::string out = "{"s;
    std::transform(std::begin(collection), 
                   std::end(collection),
                   std::back_inserter(out),
                   [&out](const auto& entry)
                   {
                       entry.push_back("{"s + to_string(entry) + "},"s);
                   });
    out += "}";
    return out;                   
}
```

Hey, look! We have a function that'll convert collections to 
initializer-list-like strings.

```cpp
auto aVector = SubStructVector{};
// populate it
auto anArray = SubStructArray{};
// populate the other one

std::cout << "the vector: " << to_string(aVector) << "\n"
          << "the array: " << to_string(anArray) << std::endl;
```

... and just for comparison:

```cpp
auto aVector = SubStructVector{};
// populate it
auto anArray = SubStructArray{};
// populate the other one

std::cout << "the vector: " << SubStructVectorToString(aVector) << "\n"
          << "the array: " << SubStructArrayToString(anArray) << std::endl;
```

This also points out another relationship.

## Templates are your friend

Template seem like things that are rarely used. However, a template can remove
coupling that could otherwise 'cripple' the testability of a component.

Suppose we have a set of different components, but the interface to those 
components is the say. Just for simplicity let's say they have a `read` and a
`write` function as their entry points.

This is a classic case for inheritance. For example,

```cpp
class Base {
public:
    int read(std::string& buffer) = 0;
    int write(std::string& buffer) = 0;
}

class Database : public Base {
public:
    int read(std::string& buffer) {
        // read from a database
        return 0;
    }
    int write(std::string& buffer) {
        // write to a database
        return 0;
    }
};

class File : public Base
{
public:
    int read(std::string& buffer) {
        // read from a file
        return 0;
    }
    int write(std::string& buffer) {
        // write to a file
        return 0;
    }
};

class Socket : public Base
{
public:
    int read(std::string& buffer) {
        // read from a socket
        return 0;
    }
    int write(std::string& buffer) {
        // write to a socket
        return 0;
    }
};

class UnixDomainSocket : public Base
{
public:
    int read(std::string& buffer) {
        // read from a unix domain socket
        return 0;
    }
    int write(std::string& buffer) {
        // write to a unix domain socket
        return 0;
    }
};
```

The application simply instantiates the proper derived type (e.g., 
`auto place = Socket{};`) then passes that to a function. For example,

```cpp
std::string ReadBook(Base& base)
{
    std::string buffer;
    base.read(buffer);
    return buffer;
}
```

That `ReadBook()` function must know about the `Base` class' interface to
compile.

Over time, the number of reader/writer abstractions could grow to great heights.
The forrest could become huge.

Could we do something with templates to prevent the requirement of being
derived from `Base`? Yes, yes we can.

Consider these pared down (and not syntactically correct) declarations for the 
classes above:

```cpp
class Database
public:
    int read(std::string& buffer) {}
    int write(std::string& buffer) {}
};

class File
{
public:
    int read(std::string& buffer) {}
    int write(std::string& buffer) {}
};

class Socket
{
public:
    int read(std::string& buffer) {}
    int write(std::string& buffer) {}
};

class UnixDomainSocket
{
public:
    int read(std::string& buffer) {}
    int write(std::string& buffer) {}
};
```

Do note that these are standalone classes that do not derive from a common base
class. So how can we provide 'generic' `ReadBook()` functionality. 

Consider the following function:

```cpp
template <typename READER>
std::string ReadBook(READER& reader)
{
    std::string buffer;
    reader.read(buffer);
    return buffer;
}
```

This templated function does not rely on anything other than the parameterized
typename must have a `read()` method declared in it.

```cpp
auto reader = File{"/etc/os-release"};
auto contents = ReadBook(reader);
```

Sometimes the coupling that results from inheritance can cause excessive
dependencies -- at compile, runtime, and testing. Using this template scheme,
the functionality of the parameterized `ReadBook()` function is minimized to
only being 'called' with something having the right method(s). i.e., not forcing
the interface of a base-class.

