# C Vector Library
A simple vector library for C. This library's vectors work in a similar manner to C++ vectors: they can store any type, their elements can be accessed via the `[]` operator, and elements may be added or removed with simple library calls.

You can easily create a vector like so:

```c
int* num_vec = vector_create();
```

You can store different data types in vectors by simply modifying the pointer type:

```c
// vector of chars
char* char_vec = vector_create();

// vector of integers
int* int_vec = vector_create();

// you can even store structs!
struct foo* foo_vec = vector_create();
```

A vector's elements can be accessed using the `[]` operator:

```c
char* char_vec = vector_create();

// add an item
vector_add(&char_vec, 'a');

// print the item from the vector
printf("first item: %c\n", char_vec[0]);
```

Vectors require that their pointer type corresponds to the contents of the vector (e.g. `int*` for a vector containing `int`s); otherwise, you have to explicitly cast the vector pointer to the correct type when accessing elements or making library calls:

```c
// recommended vector use
int* foo = vector_create();

vector_add(&foo, 5);

// reassignment
foo[0] = 3;


// generic vector (not recommended)
vector bar = vector_create();

vector_add(&(int*)bar, 3);

((int*)bar)[0] = 5; // we have to cast to an int* so we can properly access vector elements
```
Note: the `vector` type is just an alias for `void*` defined in `vec.h`.

# How It Works

These vectors can be manipulated in a similar manner to C++ vectors; their elements can be accessed via the `[]` operator and elements may be added or removed through the use of simple library calls.

This works because these vectors are stored directly alongside special header in memory, which keeps track of the vector's size and capacity:

    +--------+-------------+
    | Header | Vector data |
    +--------+-------------+
             |
             `-> Pointer returned to the user.

This design was inspired by anitrez's [Simple Dynamic Strings](https://github.com/antirez/sds/).

This library uses the preprocessor to perform compile-time type checks. The type checks are done using C23's `typeof` operator, which was actually implemented in some compilers before C23 (such as GCC and Clang). [Older versions of MSVC](https://learn.microsoft.com/en-us/cpp/c-language/typeof-c?view=msvc-170#requirements) do not support the `typeof` operator. See [Missing typeof Reference Sheet](#missing-typeof-reference-sheet) for info about vector usage when `typeof` is not present.

If you're using this library in a C++ project, and `decltype` is supported, a macro substitute for `typeof` will automatically be applied.

# Usage

Just because these vectors can be accessed and modified like regular arrays doesn't mean they should be treated the same in all cases.

Because of the hidden header data, these vectors can only be properly freed by calling `vector_free(vec)`.

Since the header data is stored in the same location as the the vector's elements, the entire vector might be moved to a new location when its size is changed. This means the library calls that modify the vector's capacity need to take the address of the vector pointer, e.g `&vec`, so it can be reassigned. This obviously means that you can't have copies of the same vector pointer in multiple locations (unless the vector's capacity is constant), but this issue can be easily remedied by indirectly referencing the vector, either by referencing some kind of structure that contains the vector pointer, or by referencing the vector pointer itself.

Here is an example using a function that reassigns the vector pointer:

```c
int* baz = vector_create();

// adds the number "5" to the vector
vector_add(&baz, 5); // takes the address of the `int*` in case the pointer needs to be changed
```

This is another similarity to *Simple Dynamic Strings*, which has library calls that are a little more clear in their reassignment, e.g. `s = sdscat(s,"foo")`, but this functionality would break the `vector_add` and `vector_insert` macros, which already expand to an assignment expression. If you're using a compiler that supports the `typeof` operator, the vector macros will perform compile-time checks that will produce an error when the vector pointer is used incorrectly.

Some functions never move a vector to a new location, so they just take a regular vector pointer as an argument:

```c
int* age_vec = vector_create();

// get the vector size
int num_ages = vector_size(age_vec); // just pass the `int*` without taking its address
```

If the vector parameter for a function or macro is named `vec`, you don't need to take the address, but if the parameter is named `vec_addr`, then you do need to take it.

# Best Practices

Because of the differences between regular arrays and vectors, it's probably a good idea to try to distinguish them from one another.

One way to do this is to create an alias for various vector types using `typedef`:

```c
typedef float* vec_float; // vector alias for float

vec_float qux = vector_create();
```

The *recommended* way to differentiate between vectors and arrays is to simply name them differently, for example, an array of eggs could be named `eggs` while a vector of eggs could be named `egg_vec`.

# What About Structures?

If you have a vector storing some kind of structure, you can't initialize new elements of the vector like you would a variable, e.g. `{ a, b, c }`. To get around this, there's a set of special macros that allow for more control over element initialization.

Here is an example:

```c
typedef struct { ... } foo;

foo* foo_vec = vector_create();

// the lifetime of temp is not guaranteed to be long; don't use this pointer after initialization
foo* temp = vector_add_dst(&foo_vec);
temp->a = 1;
temp->b = 2;
temp->c = 3;
temp = NULL; // stop using temp now that the element is initialized
```

# Reference Sheet

Below is a cheat sheet for this library's functions and macros.

Some functions and macros take a normal vector argument, e.g. `vec`, while others can change the vector's memory location and therefore require the address of the vector pointer, e.g. `&vec`. You should get a compile-time error if you use a vector pointer incorrectly. Parameter names will also indicate whether or not to take the address of the vector pointer.

| Action                                  | Code                                       | Changes vector address? |
|-----------------------------------------|--------------------------------------------|-------------------------|
| create a vector                         | `type* vec = vector_create();`             | N/A                     |
| free a vector                           | `vector_free(vec);`                        | N/A                     |
| add `item` to the vector `vec`          | `vector_add(&vec, item);`                  | yes                     |
| insert `item` into `vec` at index `9`   | `vector_insert(&vec, 9, item)`             | yes                     |
| erase `4` items from `vec` at index `3` | `vector_erase(vec, 3, 4);`                 | no (moves elements)     |
| remove item at index `3` from `vec`     | `vector_remove(vec, 3);`                   | no (moves elements)     |
| get the number of items in `vec`        | `int size = vector_size(vec);`             | no                      |
| get the storage capacity of `vec`       | `int capacity = vector_get_capacity(vec);` | no                      |
| add `item` to the vector `vec`          | `type* temp = vector_add_dst(&vec);`       | yes                     |
| insert `item` into `vec` at index `9`   | `type* temp = vector_insert_dst(&vec, 9);` | yes                     |
| reserve space for 255 items in `vec`    | `vector_reserve(&vec, 255);`               | yes                     |
| make a copy of `vec`                    | `type* vec_copy = vector_copy(vec);`       | no                      |

# Convenience Iterator and Access Macros

Several macros are implemented to make iteration and element access more convenient

### `vector_front(vec)`

Access the first element in the vector by value (`vec[0]`).

Will cause an out-of-bounds access on an empty vector.

### `vector_back(vec)`

Access the last element in the vector by value (`vec[vector_size(vec) - 1]`).

Will cause an out-of-bounds access on an empty vector.

### `vector_begin(vec)`

Address of first element in vector (`vec`).

### `vector_end(vec)`

Address of element "past the end" of the vector. (`vec + vector_size(vec)`).

### `vector_foreach(*T, vec)`

Iterate over the vector using the provided item pointer.

```c
int *int_vec = vector_create();

for (size_t i = 0; i < 100; i += 1) {
    vector_add(&int_vec, i);
}

int sum = 0;
vector_foreach(int *ip, int_vec) {
    sum += *ip;
}

printf ("The sum of items is %d", sum);
```

# Missing typeof Reference Sheet

Because some compilers don't support the `typeof` operator, which is used for static type checks in some of this library's macros, you have to use a slightly different set of macros. Unfortunately, this also means some errors will be missed at compile time, so if you're getting runtime errors, make sure you are properly using `vec` and `&vec` for their corresponding calls.

| Action                                  | Code                                             | Changes vector address? |
|-----------------------------------------|--------------------------------------------------|-------------------------|
| add `item` to the vector `vec`          | `vector_add(&vec, type) = item;`                 | yes                     |
| insert `item` into `vec` at index `9`   | `vector_insert(&vec, type, 9) = item;`           | yes                     |
| erase `4` items from `vec` at index `3` | `vector_erase(vec, type, 3, 4);`                 | no (moves elements)     |
| remove item at index `3` from `vec`     | `vector_remove(vec, type, 3);`                   | no (moves elements)     |
| add `item` to the vector `vec`          | `type* temp = vector_add_dst(&vec, type);`       | yes                     |
| insert `item` into `vec` at index `9`   | `type* temp = vector_insert_dst(&vec, type, 9);` | yes                     |
