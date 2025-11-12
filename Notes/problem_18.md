[The middle <<](./problem_17.md) | [**Home**](../README.md) | [>> Heterogeneous Data](./problem_19.md)

# Problem 18 - A case study in Strings
## **2025-10-09**


So if a string is a sequence of chars, can we just use vector<char> when we want a string?

Ans: Yes, in principle. But there might be some reasons not to...

Primarily, vector functions as a container of items that my not have much to do with each others.
But in a string, it's the elements taken together as a group that make a string what it is.

E.g. Strings are more likely than vectors to 
- be appended to one another
- be compared lexicographically
- be searched for a substring
- have a substring extracted

All of these are about characters in the sequence in conjunction with their neighbors.
So, a string would likely be expected to support a difference interface from vector.

There are different kinds of characters
There is char(8 bits), but there are also wider character types, so string should be a template parameterized by character type.
For simplicity, we'll stick with char and not write a template.

No matter what the character type is, they all have one thing in common - they're a basic type, not a class. 
So no constructors/destructors, no placement new needed.

Simple implementation:
```C++
class stirng{
    size_t length, cap;
    char *theString;
    ...
};
```
**Question:** Do we still need null termination?
**Answer:** No, but also, kind of. We have a length field now, to tell us how long the string is.
**Advantages:**
1. length is O(1)
2. There can now be `\0`'s in the middle of a string
**But:** We still need to be able to interface with functions expecting char*.


```C++
class string{
    ...
    public:
        ...
        const char *c.str(); //Returns an equal C-style string, O(1) time
        ...
}
```
For this to work, the char* must be null-terminated. So we still need `theString[length]=='\0'` even though we don't need this condition to determine length.

**Note:** if our string does contain `'\0`'s(not as terminators), the char* returned by c.str will be interpreted as ending at the first `'\0'`

**Comparison(lexicographically)** 
C:strcmp - direct comparison was comparison at pointers

Since string is a class, we can define `operator<`, `operator<=`, etc. for strings.
Instead of `if(strcmp(s1, s2)<0)...`, we can write `if(s1<s2)...`
But `strcmp` had one advantage over `operator<`, etc.: it is 3-valued.
So we can write:
```C++
char *s1, *s2;
auto result=strcmp(s1, s2);//1 linear scan
if(result<0){...}
else if(result==0){...}
else {...}
```
vs.
```C++
string s1, s2;
if(s1<s2){...}//1 linear scan
else if(s1==s2){...}//2 linear scans. Mostly the same operation done twice
else {...}
```
If we want to compete with C, we need a 3-valued comparison operation: `operator<=>`
The standard provides a class `std::strong_ordering` and constants `std::strong_ordering{less, equal, greater}` that we can use as the result of comparison.
These constants compare, respectively, as {`<`, `==`, `>`} 0

By default, `<=>` does lexicographical comparison for free, if you ask for it:
```C++
class Vec{
    int x, y;
    public:
        std::strong_ordering operator<=>(const Vec &other)const = default;
}
```
Or we could write it ourselves:
```C++
class Vec{
    ...
    std::strong_ordering operator<=>(const Vec &other)const {
        auto n=x<=>other.x;
        return (n==0)? y<=>other.y:n;//equivalent to default
    }
};
```
But for string, the default will do pointer comparison on `theString`, so we need to write our own:
```C++
class string{
    ...
    std::strong_ordering operator<=>(const string &other)const {
        for(size_t i=0;i<min(length, other.length);++i){
            if(theString[i]!=other.theString[i])
                return theString[i]<=>other.theString[i];
        }
        return length<=>other.length;
    }  
};
```
This also gets us `s1<s2` for free, also `<=`, `>`, `>=`. `s1<s2` is rewritten by the compiler as `(s1<=>s2)<0` etc.

There's a small problem with (in)equality.
- `<=>` always does a linear scan, even if the strings have different lengths
- But strings of different lengths cannot be equal

**Solution:** write a specialized `operator==` (exercise)
- compare lengths first
- then do a linear scan

C++ will use this for both `==` and `!=`.

**Optimizing the representation**
One thing we know about `chars` vs `T`'s - they're small(one byte). A lot of strings are small too.

`string s="a"`
`
| 1 | length
|10?| cap
| -> | a | \0 | | theString
`
pointer deref -> pointer locality
- seems wasteful when the payoff is one character.

What if short strings were represented differently?

Multiple representations for an objects... let's talk about unions.
For types `T1`, `T2`, 
```C++
Union U{
    T1 t1;
    T2 t2;//like a structure, but has one of the fields t1, t2(U has space to hold whichever is larger)
}
```
**Danger:**
```C++
T1 x;
U u;
u.t1=x;
T2 y=u.t2;//Undefined Behaviour
```
## **2025-10-21**
Unions must be used consistently - must remember which field of a union is active
Typically - discriminator variable, or the union appears inside a struct with a discriminator field

Traditional vector/string layout:
```C++
class string{
    size_t n;
    size_t cap;
    char *theString;//short string optimization, use this space as an array of chars, if n is small
    //The size n can also be discriminator
}
```
Then,
```C++
class string{
    struct S{
        size_t cap;
        char *theString;
    };
    size_t n;//if n>=sizeof(s), use S; else use arr
    union{
        S s;
        char arr[sizeof(s)];
    }
}
```
Assuming `size_t` is 8 bytes, pointers are 8 bytes. SSO can hold 15 bytes(+1 for null=16)
But(trick):What if we store the size n last, instead of first? (Note: Assumes big-endian architecture)

If the string is short - the first 7 bytes of n will be 0. The first byte of n could function as the null terminator.
Then we can hold 16 characters+null terminator

**Extracting Substrings**
-`string::substr` creates a new string - new heap allocation (Or SSO)
-Can we do better? What if we know the substring will not be mutated?

Consider:
```C++
class string_view{
    const char *start;
    size_t n;
};
```
A non-owning slice & a string
Points to a position in an existing `string` or `char*`, or `vector<char>` plus a length(to encode the end)
- No allocation
- Smaller than string 
  - Pass-by-value is reasonable
- Copy operators are trivial
  - no big 5

What could we do with this? What methods could we give `string_view`?
- iteration 
  - begin/end
  - range-based loops
    - Note: Null termination is not guaranteed, so range_based looping is appropriate
- extracting further substrings
  - remove prefix
    - modify the endpoints
  - remove suffix
    - modify the endpoints
  - substr
    - new `string_view`






# Problem 20 - Abstraction over containers
## **2025-10-21**


Recall: `map` from Racket

```scheme
(map f (list a b c)) -> (list (f a) (f b) (f c))
```

May want to do the same with vectors:

```
+---+---+---+---+     +----+----+----+----+
| a | b | c | d | ->  |f(a)|f(b)|f(c)|f(d)|
+---+---+---+---+     +----+----+----+----+

      source                  target
```

Assume target has enough space to hold as much of source as we want to send.

```C++
template<typename T1, typename T2>

void transform(const vector<T1> &source, vector<T2> &target, T2 (*f)(T1)) {
    // reminds you of trampoline :)
    auto it = target.begin();
    for (auto &x: source) {
        *it = f(x);
        ++it;
    }
}
```
This is OK, but:
- What if we want only part of the source?
- What if we want to send the source to the middle of the target, not the beginning?

More general: use iterators

```C++
// This does not compile for C++14, but it would in C++20
template <typename T1, typename T2> void transform(vector<T1>::iterator start, 
    vector<T1>::iterator finish, vector <T2>::iterator target, T2 (*f)(T1)) {
    while (start != finish) {
        *target = f(*start);
        ++start;
        ++target;
    }
}
```

Ok, but:
- What if I want to transform a list, I'll write the same code again
- What if I want to transform a list to a vector, or vice versa

Solution: Make the types stand for the iterators, not the container elements. But then how do we indicate the type for `f`?

```C++
template<typename InIter, typename OutIter, typename Fn>
void transform(InIter start, InIter finish, OutIter target, Fn f) {
    while (start != finish) {
        *target = f(*start);
        ++start;
        ++target;
    }
}
```
- Works over vector iterators, list iterators, or any kinds of iterators.

InIter/OutIter can be any types that support `++`, `*`, `!=`, including ordinary raw pointers.

C++ will instantiate a template function with any type that has the operations being used by the function.

`Fn` can be any type that supports **function application**.

```C++
class Plus {
        int n;
    public:
        Plus(int n): n{n} {}
        int operator() (int m) { return n + m; } // function application operator
};

Plus p{5};

std::cout << p(7);  // 12
```
- Here is an object that is acting like a function, we call them **function object**
- Many people would call them *functor*, but Brad tends to avoid calling them like that, because that word has too many meanings in math and cs already, we don't want to overload it, it doesn't need another one

But more interesting, we can do something like

```C++
transform(v.begin(), v.end(), w.begin(), Plus{1});
```
- This is cheap and quick and dirty of getting things done
  
Now we have an arbitrary plus one function for any types


<u>Note</u>: 
- One of the advantages of writing function objects instead of functions and then having classes that are callable, is that you can do things more easily with classes than functions, in particular, unlike functions, classes maintain **states**. 
- The other way you can really maintain states in a function between multiple function calls is with a static variable, which is really tricky to use.
- Function objects **maintain** states. They are really powerful things.

OR we can do something like this in case you don't like objects:
```C++
transform(v.begin(), v.end(), w.begin(), [](int n) { return n + 1 });
                                         // ^ "lambda"
```
Lambda anatomy:
```
            param list
               |
lambda: [...](...) mutable? noexcept? { ... }
          |                              |
        capture list                    body
```
Semantics: 
```C++
void f(T1 a, T2 b) {
    [a, &b](int x) { body }(arg)
}
```
In the C++ context, lambdas are really just objects. This would be translated to:
```C++
void f (T1 a, T2 b) {
    class ??? {     // ??? - anonymous class we can't access the name, would be a random thing compiler made up
            T1 a; // any variables external to the lambda that you want to access here, list them in the capturing list
            T2 &b;
        public:
            ???(T1 a, T2 &b): a{a}, b{b} {}
            auto operator()(int x) const {
                body;
            }
    }
};

// Invocation would look like this
???{a, b}.operator() (...);
```

If the lambda is declared mutable, then `operator()` is not const.
- Capture list - provides access to selected vars in the enclosing scope.



# Problem 21 - What's better than one iterator?
## **2025-10-22**

Two iterators!
Consider `vector<T>::erase`
- takes an iterator to the item
- removes the item
- shuffles the following items backward
- returns an iterator to the point of erasure

O(n) cost for shuffling: Fine.

But - What if you need to erase several consecutive items?
- Call `erase` repeatedly? Pay that O(n) cost multiple times
- Faster - Shuffle the items up k positions(in one step each) where k is the # of items being erased

For this reason, we should make a `range` version of erase
```C++
iterator vector<T>::erase(iterator first, iterator last)
```
Erases items in `[first, last)`, only pays the linear cost once

So maybe iterator isn't the right level of abstraction
- Maybe the right abstraction encapsulates a pair of iterators - a `range`

Consider: composing multiple functions on some input
e.g. `filter+transform`(say take all the odd numbers and square them)
```C++
auto odd=[](int n){return n%2!=0;};
auto sqr=[](int n){return n*n;};

vector v{...};//contains something
vector<int> w(v.size()), x(v.size());
copy-if(v.begin(), v.end(), w.begin(), odd);//copy-if by import<algorithm>
transform(w.begin, w.end(), x.begin(), sqr);
```

2 problems
1. The calls don't compose well. Needs 2 separate function calls, no way to chain them
2. Needed to create a vector for intermediate storage

These functions return an iterator to the beginning of the output range

What if instead we get a pair of iterators to the beginning and end of the output range
```C++
template<typename Iter> class range{
  Iter start, finish;
  public:
    Iter begin(){return start;}
    Iter end(){return finish;}
};
```

And what if, instead, `copy-if` and `transform` took a range, rather than a pair of iterators

```C++
transform(copy-if(v, odd), sqr);//a range is anything with begin() and end() producing iterator types. Vector is a range.
```
Functions are now composable, what about intermediate storage?

Who says we need it now? A range only needs to look like it's sitting on top of a container.
Instead: have the range fetch items on demand.

`filter` - on fetch, iterate through the range until you find an item that satisfies the predicate and return it.
`transform` - on fetch, fetch an item x from the range below and return `f(x)`

These range objects are called `views` and they have on-demand fetching, no intermediate storage.
These exist in the C++20 std libraries:
```C++
import <ranges>;
vector v{1,2,3,4,5,6,7,8,9,10};
auto x=std::ranges::views::transform(std::ranges::views::filter(x, odd), sqr);
```

It gets better: `filter`, `transform` take a second form
`filter(pred), transform(f)` (Just supply the function, not the range)
Become callable objects, parameterized by a range R.

e.g. `filter(pred)(R)`, `transform(f)(R)`,
`transform(sqr)(filter(odd)(R))`

Then `operator|` is defined so that `R|A` is mapped to `A(R)`. So `B(A(R))=B(R|A)=R|A|B`.
So we can write `auto x=v|filter(odd)|transform(sqr)`, just like a Bash pipeline!

---
[The middle <<](./problem_17.md) | [**Home**](../README.md) | [>> Heterogeneous Data](./problem_19.md)
