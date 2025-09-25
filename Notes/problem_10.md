[Walk Faster! <<](./problem_9.md) | [**Home**](../README.md) | [>> But I want a vector of chars](./problem_11.md)

# Problem 10: Now you've gone too far
## **2025-09-24**

```C++
Vector v;
v.push_back(4);
v.push_back(5);

v[2];   // Out of bounds! (undefined behaviour) - may or may not crash
```

Can we make this safer?

Adding bounds checks to operator`[]` may be needlessly expensive.

Could have a second, safer fetch operator (optional for user):

```C++
class Vector {
    ...
    public:
        int &at (size_t i) {
            if (i < n) return theVector[i];
            // else what?
        }
};
```

`v.at(2)` still wrong - what should happen?
- Return any `int`, looks like non-error
- Returning a non-`int`, not type correct
- Crash the program - can we do better? Don't return. Don't crash

**Solution:** Raise an `exception`

```C++
class range_error {};

class vector {
    ...
    public:
        int &at(size_t i) {
            if (i < n) return theVector[i];
            else throw range_error{};   // Construct an object of range_error & "throw" it
        } 
};
```

## **2025-09-25**
- Client's options
1. Do nothing
    ```C++
    vector v;
    v.at(0); // The exception will crash the program
    ```
1. Catch it
    ```C++
    try {
        vector v;
        v.at(0);
    } catch (range_error &r) {  // r is the thrown object
    // usually catch by ref, save the cost of copying.
        // Do something
    }
    ```
1. Let your caller catch it
    ```C++
    int f() {
        vector v;
        v.at(0);
    }
    int g() {
        try{
            f();
        } catch(range_error &r) {
            // Do something
        }
    }
    ```
    - Exception will propagate through the call chain until a handler is found.
    - Called **unwinding** the stack
    - If no handler is found, program aborts (`std::terminate` gets called)
    - Control resumes after the catch block (problem code is not retried)

What happens if a constructor throws an exception?
- Object is considered **partially constructed**
- Destructor will not run on partially constructed object

Ex.

```C++ 

class C {...};

class D {
    C a;
    C b;
    int *c;

    public:
        D() {
            c = new int[100];
            ...
            if (...) throw something {};  // (*)
        }

        ~D() {
            delete[] c;
        }
};

D d;
```

- At (\*) the `D` object is not fully constructed, so `~D()` will not run on d
- But `a` and `b` are fully constructed so their destructors will run
- So if a constructor wants to throw, it must clean itself

```C++
class D {
    ...
    public:
        D() {
            c = new int[100];
            // ...
            if (...) {
                delete[] c;
                throw something {};
            }
        }
    }
} 
```

What happens if a destructor throws? 
> Trouble
- By default, program aborts immediately
- `std::terminate`
- If you really want a throwing destructor, tag it with `noexcept(false)` 

```C++
class myexception{};

class C {
    ...
    public:
        ~C(_) noexcept(false) {
            throw myexception{};
        }
};
```
And we have
```C++
void h() {
    C c1;
}

void g() {
    C c2;
    h();
};

void f() {
    try {
        g();
    } catch (myException &ex) {
        ...
    }
}
```

1. Destructor for `c1` throws at the end of `h`
1. Unwind through `g`
1. Destructor for `c2` throws as we leave `g`
    - No handler yet
    - Now two unhandled exceptions
    - Immediate termination guaranteed, `std::terminate` is called

**Never** let destructors throw, swallow the exception if necessary

Also note that you can throw _any value_, not just objects

---
[Walk Faster! <<](./problem_9.md) | [**Home**](../README.md) | [>> But I want a vector of chars](./problem_11.md)
