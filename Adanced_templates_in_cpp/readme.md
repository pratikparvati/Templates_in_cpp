# Advanced Templates in Modern C++

We write high-level languages because they make our program more concise and easier to maintain. Most of the low level details are abstracted over by the compiler, and programmers no longer concern himself into these details. But sometimes the programmers need to know more about some particular details than compiler does to write more efficient code. 

Templates enforce the C++ compiler to execute algorithms at compilation time, which gives us more flexibility to write generic program to avoid run-time overhead. This article is an extension to my previous article [Introduction to C++ templates](https://pratikparvati.com/html/blogview.html?id=-Mdn0MES-VLGAnjPraGl&lan=cpp) to give insight on some advanced features added in C++11, C++14 and C++17.

### Dependent names

A **Dependent Name** is any name within a template definition that depends on one or more of the template parameters. When using templates there is a distinction between the point of definition of the template and the point of instantiation (where templates are used). Names that depend on a template don't get bound until the point of instantiation. For instance:

```C++
template <typename T>
struct Base
{
    void baseMethod()
    {
        std::cout << "Base<T>::f()\n";
    }
};

template <typename T>
struct Derived : Base<T>
{
    void derivedMethod()
    {
        std::cout << "Derived<T>::g()\n  ";

        /**
         * ERROR: Dependent Name (as there is no baseMethod() in derived class): Call to baseMethod() depends
         * on Base class template hence compiler is not aware of baseMethod().
         */
        baseMethod();
        f(); // Non dependent Name: hence no issues here
    }

    void f()
    {

    }
};

int main()
{
    Derived<int> d{};
    d.derivedMethod();
    return EXIT_SUCCESS;
}

/* ERROR
test.pp.cpp: In member function 'void Derived<T>::derivedMethod()':
test.pp.cpp:21:20: error: there are no arguments to 'baseMethod' that depend on a template parameter, so a declaration of 'baseMethod' must be available [-fpermissive]
         baseMethod();
                    ^
test.pp.cpp:21:20: note: (if you use '-fpermissive', G++ will accept your code, but allowing the use of an undeclared name is deprecated)
*/
```
The compiler treats the call to `baseMethod()` as a non-dependent name, and must be resolved at the point of template's definition. At this point compiler doesn't know `Base<T>::baseMethod()` as it can be specialized later. A simple fix is to make the compiler understand that the call `baseMethod()` depends on template parameters. Changing the call to `baseMethod()` as `Base<T>::baseMethod()` (or `this->baseMethod()`) would compile the code without errors as the call to function `baseMethod()` is resolved at the point of template's instantiation.

Now, what if the dependent name is a type? 

```C++
template <typename T> 
struct Base 
{
   using value_type = T;
   void baseMethod() 
   {
       std::cout << "Base<T>::f()\n";
   }
};


template <typename T> 
struct Derived : Base<T> 
{
   value_type val = 10; //(1) ERROR: 'value_type' is not declared in the scope
   Base<T>::value_type val = 10; //(2)ERROR: need 'typename' before 'Base<T>::value_type' because  'Base<T>' is a dependent scope
   typename Base<T>value_type val = 10; //(3) Works
   
   void derivedMethod()
   {
       std::cout << "Derived<T>::g()\n  ";     
       Base<T>::baseMethod();
   }
};
```
We already know why `(1)` doesn't work (`value_type` is non-dependent); when we use `typename` in`(3)` we are explicitly telling the compiler that it is a type. This is stated in the C++ standard, section 14.6:
> A name used in a template declaration or definition and that is dependent on a template-parameter is assumed not to name a type unless the applicable name lookup finds a type name or the name is qualified by the keyword `typename`.

### Template Template Parameter
Template Template Parameters enable a template to be parameterized by the name of another template.

```C++
// Example for template template parameter used with class

template <typename T, template <typename, typename> class Cont > // the keyword class is a must before C++17, otherwise typename can also be used
class MyContainer
{
public:
  explicit MyContainer(std::initializer_list<T> inList): data(inList)
  {  
  }
  int getSize() const
  {
    return data.size();
  }

  void printCont()
  {
      for(const auto& d: data)
      {
          std::cout << d << ' ';
      }
      std::cout << '\n';
  }

private:
  Cont<T, std::allocator<T>> data; // the hidden default allocator in STL should be explicitly defined with the container to work with templates.                                                              

};

int main()
{
  MyContainer<int, std::vector> myIntVec{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}; 
  std::cout << "myIntVec.getSize(): " << myIntVec.getSize() << std::endl;
  return EXIT_SUCCESS;
}

/*OUTPUT
myIntVec.getSize(): 10
*/
```

The first parameter `T`, is the name of a type. The second parameter `Cont`, is a template template parameter. It's the name of a class template that has a two `typename` parameter. Note that we didn't give a name to the `typename` parameter of Cont, although we could have:

```C++
template <typename T, template <typename ElementType, typename Allocator> class Cont>
class MyContainer;
```
However, such a name (`ElementType`and `Allocator` above) can serve only as documentation. These names are commonly omitted, but you should feel free to use them where you think they improve readability. For additional convenience, we can employ a default for the template template argument.

```C++
template <typename T, template <typename, typename> class Cont = std::queue>
class MyContainer 
{
    //...
};
//...
MyContainer<int> c1; // use default: Cont is std::queue
MyContainer<std::string, std::list> c2; // Cont is std::list
```

The template template parameter can also be used with Function template

```C++
template <typename T, template <typename, typename> class Cont >
void print_container(Cont<T, std::allocator<T> > container) 
{
    for (const T& v : container)
        std::cout << v << ' ';
    std::cout << '\n';
}

int main()
{
    std::vector<char> v{'c','+','+'}; //initilize vector
    std::list<int> lt(5,10); // initializing a list with 5 elements

    std::cout << "Vector elements: ";
    print_container(v);
    std::cout << "List elements: ";
    print_container(lt);
    return EXIT_SUCCESS;
}

/*OUTPUT
Vector elements: c + + 
List elements: 10 10 10 10 10 
/*
```
#### Passing Container of Containers as C++ Template Parameter

```C++
template <
    template <typename, typename> class Cont1,
    template <typename, typename> class Cont2,
    typename T>
void print_container(Cont1<Cont2<T, std::allocator<int>>, std::allocator<Cont2<T, std::allocator<T>>>> container)
{
    for (const auto &c2 : container)
    {
        for (const auto &v : c2)
        {
            std::cout << v << ' ';
        }
        std::cout << '\n';
    }
    std::cout << '\n';
}

int main()
{
    std::vector<std::vector<int>> vec{{1, 2, 3}, {4, 5, 6}}; //initilize vector

    std::cout << "Vector elements: \n";
    print_container(vec);
    return EXIT_SUCCESS;
}

/*OUTPUT
Vector elements: 
1 2 3 
4 5 6 
*/

/************** Breaking down function argument for better understanding ******************
 * 
 * using T1 = Cont2<T, std::allocator<int> >, then the argument would be
 * 
 * Cont1<T1, std::allocator<T1>> 
 */
```

The above code is complex and explicit way of passing container of containers.

### Forwarding Reference used with Templates
Forwarding Reference allows a template function that accepts a set of arguments to forward these arguments to another function while retaining the lvalue or rvalue nature of the original function arguments. It reduces excessive copying and simplifies code by reducing the need to write overloads to handle lvalues and rvalues separately.

##### The Problem

Let's look at the below example

```C++
template<typename T> 
void outerMethod(T& param)
{ 
    innerMethod(param); 
}
```
The `outerMethod()` accepts an lvalue reference, therefore we can only pass lvalues to it.

```C++
int x = 10;
outerMethod(x); // Works, lvalue is passed as argument

outerMethod(10) // ERROR: passing rvalue to lvalue reference
```
We can fix this by making the `outerMethod()` to accept `const` lvalue reference, then the `innerMethod()` would not be allowed to modify the argument. We would have to overload the `outerMethod()` to handle rvalues

```C++
template<typename T>
void outerMethod(T&& param)
{
    innerFunction(param);
}
```
What if the `innerFunction()` must accept the rvalue reference as an argument. We know that, even if the outer function accepts an rvalue reference when it comes to passing that argument to the inner function it will be seen by the compiler as lvalue and is therefore not allowed. This is where perfect forwarding or Forwarding reference comes to our rescue.

#### Reference Collapsing

Reference collapsing is a set of rules in C++11 to determine the value of `T` of a template function argument.
- Taking the reference of an lvalue reference results in an lvalue reference `T& &` becomes `T&`
- Taking the rvalue reference of an lvalue reference is an lvalue reference `T& &&` becomes `T&`
- Taking the lvalue reference of an rvalue reference is an lvalue reference `T&& &` becomes `T&`
- Taking the rvalue reference of an rvalue reference is an rvalue reference `T&& &&` becomes `T&&`

In any situation where an lvalue reference is involved the compiler will always collapse the type to an lvalue reference, if an rvalue references is involved then the type deduced is an rvalue reference.

##### The solution: Forwarding with `std::forward`
The function `std::forward` is required for solving the perfect forwarding problem with the functions purpose to resolve that awkward rule in which rvalue references are treated as lvalues.

```C++

class MyClass
{
public:
    MyClass(std::string b) : b(b) {} // Copy Constructor
    MyClass(const MyClass &other) : b(b)
    {
        b = other.b;
        std::cout << "Copy Constructor\n";
    }                             
    MyClass(MyClass &&other) // Move Constructor
    {
        b = std::move(other.b);
        std::cout << "Move Constructor\n";
    }

private:
    std::string b;
};
// And a template function
template <typename T>
void OuterFunction(T &&param)
{
    // As per the rule (third and fourth) lvalue evaluates to lvalue and rvalue evaluate to rvalue
    MyClass a(std::forward<T>(param));
}

int main()
{
    // Passing an lvalue
    MyClass a = MyClass("Amar");
    OuterFunction(a); 
    // Passing an rvalue
    OuterFunction(MyClass("Akbar"));
}
/*OUTPUT
Copy Constructor
Move Constructor
*/
```

### Variable Templates
It is possible to define templated variables since C++14. The most important use of Variable Templates is in defining parametrized constants. Let's take an example of a numeric constant, pi i.e,`π`; which needs to be defined for various numeric types (e.g., `int`, `float`, `double`) to handle different precisions.

```C++
template <typename T>
constexpr T pi = T(3.1415926535897932385);
```
Now we can have `pi` constant for different numeric types.

```C++
std::cout << pi<int> << std::endl;
std::cout << pi<double> << std::endl;
```
We can use Variable Templates to compute mathematical calculations at compile time. Here is an interesting piece of code.

```C++
template<size_t T> struct fact;

template<> // Explicit specialization
struct fact<0>
{
    constexpr static auto value = 1;
};

template<size_t T>
struct fact
{
    constexpr static auto value = T * fact<T - 1>::value;   
};

static_assert(fact<0>::value == 1);
static_assert(fact<1>::value == 1);
static_assert(fact<2>::value == 2);
static_assert(fact<3>::value == 6);
static_assert(fact<4>::value == 24);
static_assert(fact<5>::value == 120);
```
The above code evaluates the factorial of numbers at compile time reducing run time overhead to improve performance.

>**_NOTE_:** The `static_assert` throws error during compile time if the factorial of a number doesn't match.

### Template Type Alias
In C++, it is possible to create synonyms that can be used instead of a type name. The syntax for type alias is as follows

```C++
using identifier = type-id 

template<template-params-list> identifier = type-id // to alias templates
```
For example:

```C++
template <typename T> 
using vec_t = std::vector<T, custom_allocator<T>>; 

vec_t<int>           vi; // std::vector<int, custom_allocator<int>>
vec_t<std::string>   vs;  // std::vector<std::string, custom_allocator<std::string>>

template<typename T> 
using ptr = T*;
pointer<double> p = new double;   // double* p = new double;
```

### Variadic Templates

C++11 lets us define variadic templates, taking any amount of parameters, of any type, instead of just a specific number of parameters. For example, you can use the following code to call `f()` for a variable number of arguments of different types:

```C++
template<typename T, typename... Tail>
void f(T head, Tail... tail)
{
    g(head);   //do someting to head
    f(tail...);      //try again with tail
}

void f() { }  //do nothing
```
The key to implementing a variadic template is to note that when you pass a list of arguments  to  it,  you  can  separate  the  first  argument  from  the  rest. After evaluating the function (`g()`) for `head`, the function `f()` is recursively called with the rest of the arguments (`tail`).The ellipses `...` is used to indicate **the rest** of a list. when the tail become empty and we need a separate function to deal with that.

When we call `f()` as `f(1.5, 23, "Amar");`, the `head` (`1.2`) is precessed by `g()` and later call `f(23, "Amar");` with rest parameters, which will recursively call `f("Amar");`, which will call empty function `f()`.

```C++
f(1.5, 23, "Amar") calls --> f(23, "Amar") calls --> f("Amar") calls --> f()
```

#### Variadic Function Template
Here is a basic example :

```C++
template<typename T>
T multiply(const T& arg)
{
  return arg;
}

template<typename T, typename... ARGS> // Function parameter pack
T multiply(const T& arg, const ARGS&... args)
{
  return arg * multiply(args...); // Unpacking the parameter
}

int main()
{
  std::cout << multiply(1, 5u, 6u, 8L);
}

/*OUTPUT
240
*/
```

#### Variadic Class Template

Here is a basic example

```C++
template <typename... T_values>
class Base
{
public
    virtual void f(T_values... values) = 0;
};

class Derived1 : public Base<int, short, double>
{
public:
    void f(int a, short b, double c) override;
};

class Derived2 : public Base<std::string, char>
{
public:
    void f(std::string a, char b) override;
};
```
##### Parameter Packs

The `typename... T_values` is called template **parameter pack**. if you are just templating a function then it's called a **function parameter pack**.

```C++
class MyClass
{
public:
    template <typename... T_values>
    void myMethod(T_values... values);
};
```
##### Expanding the Parameter
The `T_values...` in `myMethod()` signature is **unpacking** the parameter pack in function parameter list. 

#### Recursive variadic class template with partial specialization

Let's look at an example of recursive variadic template to print out the parameter types of a parameter pack.

```C++
template <typename... Args>
struct PrintType;

template <typename First, typename... Args>
struct PrintType<First, Args...>
{
    static std::string name()
    {
        return std::string(typeid(First).name()) + " " + PrintType<Args...>::name();
    }
};

// Need  partial specialization to end the recursion
template <>  // Partial specialization
struct PrintType<>
{
    static std::string name()
    {
        return "";
    }
};

template <typename... Args>
std::string type_name()
{
    return PrintType<Args...>::name();
}

int main()
{
    std::cout << type_name<bool, char, int , double>() << std::endl;
    return 0;
}

/*OUTPUT
b c i d
*/
```

#### Forwarding references used with variadic templates

```C++
template <typename... Args>
void outerMethod(Args&&... args) {
    innerMethod(std::forward<Args>(args)...);
}
```

>**_NOTE_**: Forwarding references can only be used for template parameters

#### Varidic template with template template parameter

```C++
template< typename T>
struct MyStruct
{
private:
  T* cont;
  size_t size;
public:
  MyStruct(std::initializer_list<T> list): cont(new T[list.size()]), size(list.size())
  {
      int i = 0;
    for(auto &l: list)
    {
      *(cont + i++) = l;
    }
  }
  
  void printCont()
  {
    for(int i  = 0; i < size; i++)
    {
      std::cout << *(cont + i) << ' ';
    }
    std::cout << '\n';
  }
};

template< template<typename, typename...> class ContainerType, typename Type, typename... Types>
auto build_container(Type first, Types... args)
{
    ContainerType<Type> c{first, args...};
    return c;
}

int main()
{
    using namespace std::string_literals;
    auto v = build_container<MyStruct> ("Amar"s, "Akbar"s, "Anthony"s, "Arjun"s);
    v.printCont();
}

/*OUTPUT
Amar Akbar Anthony Arjun
*/
```

### Keyword `typename` and `class`
`typename` and `class` are replaceable in most of the cases. However, in C++, sometimes you must use typename.

```C++

struct Entity
{
    using SubType = int;
    //...
};

template <class T>
class MyClass
{
    typename T::SubType type; // keyword typename is used as the identifier before the type.
    // ...
};


// Another example
class ClassA 
{
public:
    class foo 
    {
    };
};
 
template<typename C>
class ClassB : public C::foo // dependent name, typename keyword is a must
{
};

```
We should use `typename` when a dependent type occurs. To Specify Template Template Type we should use `class` keyword (prior to C++17).

```C++
template< template<typename, typename...> class ContainerType, typename Type, typename... Types>
auto build_container(Type first, Types... args)
{
    ContainerType<Type> c{first, args...};
    return c;
}
```

Understanding advanced C++ templates is crucial  if the developer wish to write flexible and robust code. Templates have advantage over code reuse and faster iterative development, which enhances the flexibility of the code. That being said templates has some downsides, it can cause code bloat problems and can also lead to longer compilation time. When error occurs, the error message is very messy and it is not easy to locate the error.




