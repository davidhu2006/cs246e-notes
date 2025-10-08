[Memory management is hard <<](./problem_15.md) | [**Home**](../README.md) | [>> The middle](./problem_17.md)

# Problem 16: Is vector exception safe?
## **2025-10-07**

Consider:

```C++
template<typename T> class vector {
    size_t n, cap;
    T *theVector;
    public:
        vector(size_t n, const T &x): 
            n{n}, cap{n}, 
            theVector{static_cast<T*>(operator new(n * sizeof(T)))} {
                for (size_t i = 0; i < n; ++i)
                    // copy constructor for T, could throw - then what?
                    new(theVector + i) T(x);
            }
};
```
- `malloc` vs `operator new`: `malloc` returns a `nullptr` if it fails, `operator new` fails it would throw an exception, 
but then it means we wouldn't have done anything, nothing has been allocated (strong guarantee), no problem.
- Other place we can get an exception is in the copy ctor of `T`.
- Partially constructed vector - destructor will not run
    - Broken invariant, does not contain `n` valid objects

**Fix:**
```C++
template<typename T> class vector {
    size_t n, cap;
    T *theVector;
public:
    vector(size_t n, const T &x): 
        n{0}, cap{n}, 
        theVector{static_cast<T*>(operator new(n * sizeof(T)))} {
            try {
                while(this->n < cap) {
                    new(theVector + this->n) T(x);
                    ++this->n;
                }
            } catch(...) {  // ... supresses all type checking (accept whatever)
                while(this->n >0){
                    theVector[this->n].~T();
                    --this->n;
                }
                operator delete(theVector);
                throw;  // rethrow the exception, we don't need to refer to the exception to throw it
            }
        }
};
``` 
- `...` in the `catch` block is literally `...` and not Brad wants us to fill in the blank. 
- In languages like Java, there is a base class for all errors, but this is not the case for C++ as it can throw literally anything. 
- One way we can *possibly* catch all is to catch `std::exception`, but again not everything follows this.
- Be careful about throwing `std::string`, because it actually throws `char*` and not `string`.


Abstract the filling part into its own function (since the function is quite big now):
```C++
template<typename T> void uninitialized_fill(T *start, T *finish, const T &x) {
    T *p;
    try {
        for (p = start; p != finish; ++p) {
            new(p) T(x);
        }
    } catch(...) {
        for (T *q = p; q != start; --q) (q-1)->~T();
        throw;
    }
}
```
This is an all or nothing function (strong guarantee).


```C++
template<typename T> class vector {
    size_t n, cap;
    T *theVector;
    public:
        vector(size_t n, const T &x): n{n}, cap{n}, theVector{static_cast<T*>(operator new(n * sizeof(T)))} {
            try {
                uninitialized_fill(theVector, theVector + n, x);
            } catch (...) {
                operator delete(theVector);
                throw;
            }
        }
};
```
- Vector is now responsible for 2 things: Creation and destruction of the array, and managing the content of the array.

Can clean this up using RAII on the array:

```C++
// Just to manage the array, nothing to do with the content, just the array itself
template<typename T> struct vector_base {
    size_t cap;
    T *v;
    vector_base(size_t n): 
        cap{n}, v{static_cast<T*>(operator new(cap * sizeof(T)))} {}
    ~vector_base() { operator delete(v); }
};

template<typename T> class vector {
    vector_base<T> vb;  // Cleaned up implicitly when vector is destroyed
    size_t n;
    public:
        vector(size_t n, const T &x): vb{n} , n{n}{
            uninitialized_fill(vb.v, vb.v + vb.cap, x); // strong-guarantee
        }
        void clear(){
            while(n) pop_back();
        }
        ~vector() {
            clear();
        }
};
```  
- We have made the filling ctor exception safe.

### **Copy Constructor:**
```C++
vector(const vector &other): vb{other.n}, n{other.n} {
    uninitialized_copy(vb.v, vb.v+vb.cap, other.begin());   // Similar to uninitialized_fill, details, exercise
}
```

Assignment: copy & swap is exception-safe as long as swap is no-throw

Push_back:
```C++
void push_back(const T& x) {
    increaseCap();
    new(vb.v + n) T(x); // If this throws, have the same vector
    ++ n; // Don't increment n before you know the construction succeded
}
``` 

What if `increaseCap` throws? 

```C++
void increaseCap() {
    if (vb.n == vb.cap) {
        vector_base vb2 {2 * vb.cap};   // RAII, anything goes wrong we're good
        uninitialized_copy(vb2.v, vb.v + n, vb.v);   // Strong guarantee
        clear();
        std::swap(vb, vb2); // might be no throw
    }
}
```

**Note:** only `try` blocks are in `uninitialized_copy` and `uninitialized_fill`.

But we have an efficiency issue - copying from old array to the new one knowing that the old array is going to be destroyed. Moving would be better.
- But moving does not preserve the old array, so if an exception is thrown during moving, our vector is corrupted
- Therefore, we can only move if we are sure that the move operation is nothrow.

```C++
void increaseCap() {
    if (n == vb.cap) {
        vector_base vb2 {2 * vb.cap};
        uninitialized_copy_or_move(vb2.v, vb2.v + n, vb.v);
        clear();
        std::swap(vb, vb2); 
    }
}

template<typename T>
void uninitialized_copy_or_move(T *start, T *finish, T *source) {
    T *p;
    try {
        for (p = start; p != finish; ++p, ++source) {
            new(p) T{std::move_if_noexcept(*p)};
        }
    } catch(...) {//will nevery happen if T has a non-throwing move constructor
        for(;p!=start;--p) (source+(p-start-1)->~T();
    }
}
```

`std::move_if_noexcept(x)` produces `std::move(x)` if `x` has a non-throwing move constructor, produces `x` otherwise.

But how should the compiler know if `T`'s move constructor is non-throwing? You have to tell it:

```C++
class C {
    public:
        C(C &&other) noexcept;
        ...
}
```

In general: moves and swaps should be non-throwing. Declare them so - will allow more optimized code to run.

Any function you are sure will never throw or propagate an exception, you should declare `noexcept`.

## **2025-10-08**

**Q:** Is `std:swap` `noexcept`?

```C++
template<typename T> void swap(T &a, T &b) {
    T c (std::move(a))
    a = std::move(b);
    b = std::move(c);
}
```

**Answer:** Only if T has a `noexcept` move constructor and a `noexcept` move assignment. How do we specify this?

```C++
template<typname T> void swap(T &a, T &b) 
    noexcept(std::is_nothrow_move_constructible<T>::value &&
             std::is_nothrow_move_assignable<T>::value) {
    ...
}
``` 
**Note:** `noexcept` = `noexcept(true)`

---
[Memory management is hard <<](./problem_15.md) | [**Home**](../README.md) | [>> The middle](./problem_17.md)