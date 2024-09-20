---
layout: post
title:  "rvalue, lvalue and std::forward"
---

Today I came across a problem of using `std::forward`, which I didn't have any idea. After some research, now I know why should we use `std::forward`. It all started with the **rvalue** concept.

## rvalue/lvalue
In C++ the terms **lvalue** and **rvalue** are used to describe different categories of expressions based on how they can be used. Let start with a example code as follow:

```cpp
int a = 10;     // 'a' is an lvalue, 10 is rvalue 
a = 20;         // 'a' appears on the left side, and it has an identifiable memory location
int b = 20;     // '20' is an rvalue (a temporary value)
int c = a + b;  // The result of 'a + b' is an rvalue, because it's a temporary result
```

From above example, we can see that `lvalue` is indeed `left value`, of course `rvalue` stands for `right value`. For `lvalue` a.k.a `left value` doesn't mean it only can appear on the left side of argument. Why?. Then we need to fully understand what is defintion of an `lvalue`
1. **lvalue**
- An **lvalue** refers to an object that **occupies some identifiable location in memory** (i.e., it has an address).
- It can appear on both the **left-hand** and **right-hand** sides of an assignment.
- lvalues represent **persistent objects** whose lifetime extends beyond a single expression.
- **lvalue reference** was already used before C++11 (it's just call reference like reference vs pointer) e.g `const Class& object` is a parameter in Copy Constructor
2. **rvalue**
- An **rvalue** refers to **temporary data** or **expressions** that **do not occupy a persistent memory location**. They typically exist only within a single expression.
- An **rvalue** can only appear on the **right-hand** side of an assignment. It represents **data** rather than a specific object in memory.
- **rvalue reference** was not introduced before C++11

## Move semantics
In C++11, the concept of **rvalue references** was introduced to allow functions to accept rvalues by reference, enabling move semantics using `std::move` & `std::forward` and optimization.
```cpp
int&& rref = 10;  // 'rref' is an rvalue reference, binds to the rvalue '10'
```
This is useful for **move constructors** and **move assignment** operators, where temporary objects can be transferred (moved) without copying.
However **move sematic** `std::move` & `std::forward` does not really move something, but it's a function templates perform **casts**. It start sound very complicated. Go ahead read the example and you will understand it more.

**Question**: Why do we need this **move constructors** or **move assignment** ?

Here I just make comparisons between **copy constructors** abd **move constructors** to illustrate the underlying reason. The same is applied for **move/copy assignments**. 

## Copy Constructor
Let's see an example of **Copy Construtor**
```cpp
#include <iostream>
#include <cstring> // for memcpy

class MyClass {
private:
    int* data;
    int size;
public:
    // Default Constructor
    MyClass() : data(nullptr), size(0){
        std::cout << "Default Constructor with size 0" << std::endl;
    }
    // Constructor with size
    MyClass(int s) : size(s) {
        data = new int[size];
        std::cout << "Constructor with size: " << size << std::endl;
        for (int i = 0; i < size; i++) {
            data[i] = i; // initialize value of data
        }
    }
    // Copy Constructor with deep copy
    MyClass(const MyClass& other) : size(other.size) {
        data = new int[size];                               // Allocate new memory
        std::memcpy(data, other.data, size * sizeof(int));  // Perform deep copy from other
        std::cout << "Copy Constructor: Deep Copy\n";
    }
    // Print out data function
    void print(std::string prefix = "") {
        // printout address of data 
        std::cout << prefix << " data " << std::hex << data << ": "; 
        // printout data
        for (int i = 0; i < size; i++) {
            std::cout << data[i] <<" ";
        }
        std::cout << std::endl;
    }
    // Deconstructor
    ~MyClass() {
        delete[] data;
    }
};


int main() {
    MyClass obj1(10);  // Create an array of size 10
    // Create a copy using the copy constructor
    MyClass obj2 = obj1;  // Deep copy happens here
    obj2.print("obj2");
    obj1.print("obj1");
    return 0;
}
```
The output of program is
```
Constructor with size: 10
Copy Constructor: Deep Copy
obj2 data 0x600003048300: 0 1 2 3 4 5 6 7 8 9
obj1 data 0x6000030482d0: 0 1 2 3 4 5 6 7 8 9
```
Here the copy constructor has performed the deep copy of one array to another, by allocating new memory for `obj2.data` and copy all the data from `obj1.data` to `obj2.data`

### Where lvalue reference is used
Here, `const MyClass&` is an **lvalue reference**, which means that the copy constructor is designed to take an **existing object** (an lvalue) as its argument and make a copy of that object.

## Move Constructor
For **Move Constructor**, it does not copy the data of source object but use the move the data from source object to the newly created object. Leaving source object in some valid state (but not usable i.e there's no data left since it was moved). To make the class use Move Constructor we need to use `std::move`.
The following code is added for Move Constructor
```cpp
MyClass {...
    // Move Constructor
    MyClass(MyClass&& other) noexcept : data(other.data), size(other.size) {
        other.data = nullptr;  // Transfer ownership, leave 'other' in valid state
        other.size = 0;
        std::cout << "Move Constructor: Shallow Copy\n";
    }
int main (
    ...
    // Perform a move using the move constructor
    MyClass obj3 = std::move(obj1);
    obj3.print();
    obj2.print();
    obj1.print();
```
The argument passed in **Move Constructor** is **rvalue reference** `MyClass&& other`. 
To make a **Move Constructor** we use std::move like this `MyClass obj3 = std::move(obj1);`. Output of program:
```
Move Constructor: Shallow Copy
obj3 data 0x6000030482d0: 0 1 2 3 4 5 6 7 8 9
obj2 data 0x600003048300: 0 1 2 3 4 5 6 7 8 9
obj1 data 0x0:
```
We can see that obj3 has taken all data from obj1, now obj1 has no data.

### Where rvalue reference is used with move sematics `std::move`?

- **Rvalue Reference (`T&&`)** in this example is **MyClass&& other**: Rvalues refer to **temporary objects** that are about to be destroyed. They don’t have a permanent memory address and can be “moved” instead of copied.
  - In the example, the expression `std::move(obj1)` **casts `obj1` to an rvalue reference**, allowing the move constructor to be called.
  - **Move semantics** are typically used when working with **temporary objects**, such as:
    - Returning objects from functions.
    - Passing by value in a function.
    - Working with containers like `std::vector`, where moving objects instead of copying them improves performance.

You can skip [Summary of Comparison between Move Constructor and Copy Constructors](##move-constructor-vs-copy-constructor-summary) and move to [std::forward](##stdforward)

## Move Constructor vs Copy Constructor Summary
From above example: you can see that **lvalue reference** (existing object) is use with Copy Constructor, while **rvalue reference** (temporary object) is used with Move Constructor

Here's a table that briefly compares the **Move Constructor** and the **Copy Constructor** in C++:

| **Feature**               | **Move Constructor**                                           | **Copy Constructor**                                      |
|---------------------------|---------------------------------------------------------------|-----------------------------------------------------------|
| **Purpose**                | Transfers ownership of resources from one object to another.   | Creates a deep copy of the original object's resources.    |
| **Syntax**                 | `MyClass(MyClass&& other)`                                    | `MyClass(const MyClass& other)`                           |
| **Resource Handling**      | Moves resources (shallow copy), leaving the source in a valid but empty state. | Duplicates resources (deep copy), leading to independent objects. |
| **Performance**            | Fast, as it avoids resource duplication (e.g., large memory allocations). | Slower due to copying of all resources, especially for large objects. |
| **When It’s Called**       | When an object is initialized from an **rvalue** (temporary object). | When an object is initialized from an **lvalue** (existing object). |
| **Use Cases**              | Ideal for temporary objects, large objects like containers (`std::vector`, `std::string`) where moving avoids costly copying. | Used when copying is necessary, such as when making an independent copy of the object. |
| **Pros**                   | - Much faster for large objects.<br>- Avoids memory duplication.<br>- Enables optimization with move semantics. | - Ensures that two objects have independent copies of the resources.<br>- No concerns about leaving the source object in a valid but empty state. |
| **Cons**                   | - Leaves the source object in a valid but unspecified state (e.g., `nullptr`).<br>- More complex to implement. | - Can be inefficient for large data structures.<br>- Resource duplication can lead to higher memory usage and slower performance. |

## std::forward
It's quite long right? 
Now you understand **lvalue reference** (reference to existing object) and **rvalue reference** (reference to temporary object). `std::move` casts its argument into **rvalue reference** making Move Construtor called (it may do more but I just want to talk about a concrete case, you need to grasp the underlying idea is to move object by move constructor, and since it's reference to temporay object, moving its data aways doesn't affect anything but save time without having to perform some deep copy).
`std::forward` is quite similar to `std::move` it casts its arguments to rvalue only under condition. 
### Universal Reference
`T&& a` is the syntax you see above for a rvalue reference. However this `T&&` is either **rvalue reference** or **lvalue reference**. C++ is 😭😭😭 I know. Then it's call universal reference.
So `std::forward` was introduced to preserve value category (rvalue reference or lvalue reference)
Example: Difference between `std::move` and `std::forward`:
```cpp
template <typename T>
void wrapper_move(T&& arg) {
    print(std::move(arg));  // Always turns arg into an r-value
}

template <typename T>
void wrapper_forward(T&& arg) {
    print(std::forward<T>(arg));  // Preserves the original value category
}
```

Now let's applied it to our previous example. We want to support `std::cout<<obj` i.e `operator<<` overloading for `MyClass` obj. Add follow code to exsiting code of `MyClass`.
```cpp
class MyClass {
    ...
public:
    // Friend function to overload operator<<
    // argument: T&& obj is an universal reference 
    template <typename T>
    friend std::ostream& operator<<(std::ostream& os, T&& obj) {
        return print(os, std::forward<T>(obj));  // Forward the object based on its value category
    }
private: 
    // Print function for lvalue reference
    static std::ostream& print(std::ostream& os, const MyClass& obj) {
        os << "Lvalue reference: ";
        for (int i = 0; i < obj.size; i++) {
            std::cout<< obj.data[i] <<" ";
        }
        std::cout << std::endl;
        return os;
    }

    // Print function for rvalue reference
    static std::ostream& print(std::ostream& os, MyClass&& obj) {
        os << "Rvalue reference: ";
        for (int i = 0; i < obj.size; i++) {
            std::cout<< obj.data[i] <<" ";
        }
        std::cout << std::endl;
        return os;
    }
}
```
And add follow to `main()` for testing
```cpp
    // Printout using std::ostream operator<< overloading
    std::cout<< obj3; //lvalue reference
    std::cout<< MyClass(10); // rvalue reference
```
Out of program will be:
```
Lvalue reference: 0 1 2 3 4 5 6 7 8 9 
Constructor with size: 10
Rvalue reference: 0 1 2 3 4 5 6 7 8 9
```

Now you're clear with the use of `std::forward` as its name: **forward** rvalue reference as rvalue reference, forward lvalue reference as lvalue reference because T&& can refer to both rvalue & lvalue reference. 
**Notice:** this  overloading `operator<<` is not recommended with rvalue reference argument. But here I just want to make the example. Think about it in depth. (Your HW).

<details>

<summary><b>Full Source Code</b></summary>
<div markdown="1">

```cpp
#include <iostream>
#include <cstring> // for memcpy

class MyClass {
private:
    int* data;
    int size;
public:
    // Default Constructor
    MyClass() : data(nullptr), size(0){
        std::cout << "Default Constructor with size 0" << std::endl;
    }
    // Constructor with size
    MyClass(int s) : size(s) {
        data = new int[size];
        std::cout << "Constructor with size: " << std::dec << size << std::endl;
        for (int i = 0; i < size; i++) {
            data[i] = i; // initialize value of data
        }
    }
    // Copy Constructor with deep copy
    MyClass(const MyClass& other) : size(other.size) {
        data = new int[size];                               // Allocate new memory
        std::memcpy(data, other.data, size * sizeof(int));  // Perform deep copy from other
        std::cout << "Copy Constructor: Deep Copy\n";
    }
    // Move Constructor
    MyClass(MyClass&& other) noexcept : data(other.data), size(other.size) {
        other.data = nullptr;  // Transfer ownership, leave 'other' in valid state
        other.size = 0;
        std::cout << "Move Constructor: Shallow Copy\n";
    }
    // Friend function to overload operator<<
    template <typename T>
    friend std::ostream& operator<<(std::ostream& os, T&& obj) {
        return print(os, std::forward<T>(obj));  // Forward the object based on its value category
    }
    // Print out data function
    void print(std::string prefix = "") {
        // printout address of data 
        std::cout << prefix << " data " << std::hex << data << ": "; 
        // printout data
        for (int i = 0; i < size; i++) {
            std::cout << data[i] <<" ";
        }
        std::cout << std::endl;
    }
    // Deconstructor
    ~MyClass() {
        delete[] data;
    }
private: 
    // Print function for lvalue reference
    static std::ostream& print(std::ostream& os, const MyClass& obj) {
        os << "Lvalue reference: ";
        for (int i = 0; i < obj.size; i++) {
            std::cout<< obj.data[i] <<" ";
        }
        std::cout << std::endl;
        return os;
    }

    // Print function for rvalue reference
    static std::ostream& print(std::ostream& os, MyClass&& obj) {
        os << "Rvalue reference: ";
        for (int i = 0; i < obj.size; i++) {
            std::cout<< obj.data[i] <<" ";
        }
        std::cout << std::endl;
        return os;
    }
};


int main() {
    MyClass obj1(10);  // Create an array of size 10

    // Create a copy using the copy constructor
    MyClass obj2 = obj1;  // Deep copy happens here
    obj2.print("obj2");
    obj1.print("obj1");

    // Perform a move using the move constructor
    MyClass obj3 = std::move(obj1);
    obj3.print("obj3");
    obj2.print("obj2");
    obj1.print("obj1");

    // Printout using std::ostream operator<< overloading
    std::cout<< obj3; //lvalue reference
    std::cout<< MyClass(10); // rvalue reference

    return 0;
}

```
</div>
</details>


## Conclusion
In this post I have shared about rvalue and lvalue reference, with examples used in **Copy Constructor** and **Move Constructor**. Move sematics `std::move` && `std::forward` to cast argument to **rvalue reference** and forward **universal reference** to correct category. 

Hope you enjoy it!

