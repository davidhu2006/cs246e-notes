[Now you've gone too far << ](./problem_10.md) | [**Home**](../README.md) | [>> Where do I even start](./problem_12.md) 

# Problem 11: But I want a vector of chars
## **2025-09-25**

Start over? No

Introduce a major abstraction mechanism, **templates**
- Generalize overtypes

#### Vector.cc

```C++
export module vector;
namespace CS246E {
    template<typename T> class Vector {
            size_t n, cap;
            T* theVector;

        public:
            Vector();
            // ...
            void push_back(T n);
            T &operator[](size_t i);
            const T &operator[](size_t) const;

            using iterator = T*;
            using const_iterator = const T*;
            // etc.
    };

    template<typename T> Vector<T>::vector() n{0}, cap{1}, theVector{new T[cap]} {}
    template<typename T> void Vector<T>::push_back(T n) {...}
    /// etc.
}
```

_main.cc_

```C++
int main() {
    CS246E::Vector<int> v;  // Vector of ints
    v.push_back(1);
    ...
    CS246E::Vector<char> w; // Vector of chars
    v.push_back('a');
    ...
}
```

- **Semantics:**
    - The first time the compile encounters `Vector<int>`, it creates a version of the vector code where `int` replaces `T` and compiles that new class
    - Can't do that unless it has all the details about the class
    - So implementation needs to be available, i.e. in the interface file
    - Can also write bodies inline

---
[Now you've gone too far << ](./problem_10.md) | [**Home**](../README.md) | [>> Where do I even start](./problem_12.md) 
