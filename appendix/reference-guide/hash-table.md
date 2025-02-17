# Hash Table

**Pintos provides a hash table data structure in `lib/kernel/hash.c`.**

* To use it you will need to include its header file, `lib/kernel/hash.h`, with `#include <hash.h>`.
* **No code provided with Pintos uses the hash table**, which means that you are free to use it as is, modify its implementation for your own purposes, or ignore it, as you wish.
* **Most implementations of the virtual memory project use a hash table to translate pages to frames.** You may find other uses for hash tables as well.

## Data Types

### Basic Element

A hash table is represented by **`struct hash`**.

* <mark style="color:blue;">**Type: struct hash**</mark>
  * **Represents an entire hash table.**
  * The actual members of `struct hash` are "**opaque**." That is, code that uses a hash table should not access `struct hash` members directly, nor should it need to. Instead, use hash table functions and macros.
  * **The hash table operates on elements of type `struct hash_elem`.**
* <mark style="color:blue;">**Type: struct hash\_elem**</mark>
  * **Embed a `struct hash_elem` member in the structure you want to include in a hash table.**
  * Like `struct hash`, `struct hash_elem` is **opaque**. All functions for operating on hash table elements actually take and return pointers to `struct hash_elem`, not pointers to your hash table's real element type.

**You will often need to obtain a `struct hash_elem` given a real element of the hash table, and vice versa.** Given a real element of the hash table, you may use the `&` operator to obtain a pointer to its `struct hash_elem`. Use the **`hash_entry()`** macro to go the other direction.

* <mark style="color:blue;">**Macro: type \*hash\_entry (struct hash\_elem \*elem, type, member)**</mark>
  * **Returns a pointer to the structure that \_elem**\_**, a pointer to a `struct hash_elem`, is embedded within.**
  * You must provide _type_, the name of the structure that elem is inside, and _member_, the name of the _member_ in _type_ that _elem_ points to.
  * For example, suppose `h` is a `struct hash_elem *` variable that points to a `struct thread` member (of type `struct hash_elem`) named `h_elem`. Then, `hash_entry(h, struct thread, h_elem)` yields the address of the `struct thread` that `h` points within.

See section [Hash Table Example](hash-table.md#hash-table-example), for an example.

### Hash Functions

**Each hash table element must contain a key, that is, data that identifies and distinguishes elements, which must be&#x20;**_**unique**_ **among elements in the hash table.** (Elements may also contain non-key data that need not be unique.)

* **While an element is in a hash table, its key data must not be changed.** Instead, if need be, remove the element from the hash table, modify its key, then reinsert the element.
* For each hash table, you _**must**_ write two functions that act on keys: **a hash function** and **a comparison function**.

These functions must match the following prototypes:

* <mark style="color:blue;">**Type: unsigned hash\_hash\_func (const struct hash\_elem \*element, void \*aux)**</mark>
  * **Returns a hash of&#x20;**_**element**_**'s data, as a value anywhere in the range of `unsigned int`.** The hash of an _element_ should be a pseudo-random function of the _element_'s key. It must not depend on non-key data in the element or on any non-constant data other than the key. Pintos provides the some functions (See Hash Functions Basis) as a suitable basis for hash functions.
  * See section [Auxiliary Data](hash-table.md#auxiliary-data), for an explanation of _aux_.

<details>

<summary>Hash Functions Basis</summary>

* <mark style="color:blue;">**Function: unsigned hash\_bytes (const void \*buf, size\_t \*size)**</mark>
  * **Returns a hash of the&#x20;**_**size**_**&#x20;bytes starting at&#x20;**_**buf**_**.**
  * The implementation is the general-purpose [Fowler-Noll-Vo hash](http://en.wikipedia.org/wiki/Fowler_Noll_Vo_hash) for 32-bit words.
* <mark style="color:blue;">**Function: unsigned hash\_string (const char \*s**</mark>**)**
  * **Returns a hash of null-terminated string&#x20;**_**s**_**.**
* <mark style="color:blue;">**Function: unsigned hash\_int (int i)**</mark>
  * **Returns a hash of integer&#x20;**_**i**_**.**

**Notes**

* If your key is **a single piece of data** of an appropriate type, it is sensible for your hash function to directly return the output of one of these functions.
* For **multiple pieces of data**, you may wish to **combine the output** of more than one call to them using, e.g., the **^** (exclusive or) operator.
* Finally, you may entirely ignore these functions and **write your own hash function** from scratch, but remember that your goal is to build an operating system kernel, not to design a hash function.

</details>

* <mark style="color:blue;">**Type: bool hash\_less\_func (const struct hash\_elem \*a, const struct hash\_elem \*b, void \*aux)**</mark>
  * **Compares the keys stored in elements&#x20;**_**a**_**&#x20;and&#x20;**_**b**_**.**
  * Returns true if _a_ is less than _b_, false if _a_ is greater than or equal to _b_.
  * **If two elements compare equal, then they must hash to equal values.**
  * See section [Auxiliary Data](hash-table.md#auxiliary-data), for an explanation of aux.
  * See section [Hash Table Example](hash-table.md#hash-table-example), for hash and comparison function examples.

### Hash Action Function

A few functions in this hash table implementation accepts a pointer to a third kind of function (called hash action function) as an argument:

* <mark style="color:blue;">**Type: void hash\_action\_func (struct hash\_elem \*element, void \*aux)**</mark>
  * **Performs some kind of action, chosen by the caller, on&#x20;**_**element**_**.**
  * See section [Auxiliary Data](hash-table.md#auxiliary-data), for an explanation of _aux_.

## Basic Functions

These functions _**create**_, _**destroy**_, and _**inspect**_ hash tables.

* <mark style="color:blue;">**Function: bool hash\_init (struct hash \*hash, hash\_hash\_func \*hash\_func, hash\_less\_func \*less\_func, void \*aux)**</mark>
  * **Initializes&#x20;**_**hash**_**&#x20;as a hash table with&#x20;**_**hash\_func**_**&#x20;as hash function,&#x20;**_**less\_func**_**&#x20;as comparison function, and&#x20;**_**aux**_**&#x20;as auxiliary data.**
  * Returns true if successful, false on failure. `hash_init()` calls `malloc()` and fails if memory cannot be allocated.
  * See section [Auxiliary Data](hash-table.md#auxiliary-data), for an explanation of _aux_, which is most often a null pointer.
* <mark style="color:blue;">**Function: void hash\_clear (struct hash \*hash, hash\_action\_func \*action)**</mark>
  * **Removes all the elements from&#x20;**_**hash**_**, which must have been previously initialized with `hash_init()`.**
  * **If&#x20;**_**action**_ **is non-null, then it is called once for each element in the hash table**, which gives the caller an opportunity to deallocate any memory or other resources used by the element. For example, if the hash table elements are dynamically allocated using `malloc()`, then action could `free()` the element. This is safe because `hash_clear()` will not access the memory in a given hash element after calling _action_ on it. However, **action must not call any function that may modify the hash table**, such as `hash_insert()` or `hash_delete()`.
* <mark style="color:blue;">**Function: void hash\_destroy (struct hash \*hash, hash\_action\_func \*action)**</mark>
  * **If&#x20;**_**action**_ **is non-null, calls it for each element in the** _**hash**_, with the same semantics as a call to `hash_clear()`.
  * **Then, frees the memory held by&#x20;**_**hash**_**.**
  * Afterward, _hash_ must not be passed to any hash table function, absent an intervening call to `hash_init()`.
* <mark style="color:blue;">**Function: size\_t hash\_size (struct hash \*hash)**</mark>
  * **Returns the number of elements currently stored in&#x20;**_**hash**_**.**
* <mark style="color:blue;">**Function: bool hash\_empty (struct hash \*hash)**</mark>
  * Returns true if _hash_ currently contains no elements, false if _hash_ contains at least one element.

## Search Functions

Each of these functions **searches a hash table for an element that compares equal to one provided**. Based on the success of the search, they perform some action, such as inserting a new element into the hash table, or simply return the result of the search.

* <mark style="color:blue;">**Function: struct hash\_elem \*hash\_insert (struct hash \*hash, struct hash\_elem \*element)**</mark>
  * Searches _hash_ for an element equal to _element_.
  * If none is found, inserts _element_ into _hash_ and returns a null pointer. If the table already contains an element equal to _element_, it is returned without modifying _hash_.
* <mark style="color:blue;">**Function: struct hash\_elem \*hash\_replace (struct hash \*hash, struct hash\_elem \*element)**</mark>
  * Inserts _element_ into _hash_. Any element equal to _element_ already in _hash_ is removed. Returns the element removed, or a null pointer if _hash_ did not contain an element equal to _element_.
  * **The caller is responsible for deallocating any resources associated with the returned element, as appropriate.** For example, if the hash table elements are dynamically allocated using `malloc()`, then the caller must `free()` the element after it is no longer needed.

**The element passed to the following functions is only used for hashing and comparison purposes.** It is never actually inserted into the hash table. Thus, **only key data in the element needs to be initialized**, and other data in the element will not be used. It often makes sense to declare an instance of the element type as a local variable, initialize the key data, and then pass the address of its `struct hash_elem` to `hash_find()` or `hash_delete()`. See section [Hash Table Example](hash-table.md#hash-table-example), for an example. (Large structures should not be allocated as local variables. See section [Struct thread](threads.md#struct-thread), for more information.)

* <mark style="color:blue;">**Function: struct hash\_elem \*hash\_find (struct hash \*hash, struct hash\_elem \*element)**</mark>
  * **Searches&#x20;**_**hash**_**&#x20;for an element equal to&#x20;**_**element**_**.**
  * Returns the element found, if any, or a null pointer otherwise.
* <mark style="color:blue;">**Function: struct hash\_elem \*hash\_delete (struct hash \*hash, struct hash\_elem \*element)**</mark>
  * **Searches&#x20;**_**hash**_**&#x20;for an element equal to&#x20;**_**element**_**.**
  * If one is found, it is removed from _hash_ and returned. Otherwise, a null pointer is returned and _hash_ is unchanged.

{% hint style="info" %}
**The caller is responsible for deallocating any resources associated with the returned element, as appropriate.** For example, if the hash table elements are dynamically allocated using `malloc()`, then the caller must `free()` the element after it is no longer needed.
{% endhint %}

## Iteration Functions

**These functions allow iterating through the elements in a hash table.** Two interfaces are supplied. The first requires writing and supplying a _**hash\_action\_func**_ to act on each element (see section [Data Types](hash-table.md#data-types)).

* <mark style="color:blue;">**Function: void hash\_apply (struct hash \*hash, hash\_action\_func \*action)**</mark>
  * **Calls&#x20;**_**action**_**&#x20;once for each element in&#x20;**_**hash**_**,&#x20;**<mark style="color:red;">**in arbitrary order**</mark>**.**
  * _**action**_**&#x20;must not call any function that may modify the hash table**, such as `hash_insert()` or `hash_delete()`.
  * _**action**_**&#x20;must not modify key data in elements**, although it may modify any other data.

The second interface is based on an "**iterator**" data type. Idiomatically, iterators are used as follows:

```c
struct hash_iterator i;

hash_first (&i, h);
while (hash_next (&i))
  {
    struct foo *f = hash_entry (hash_cur (&i), struct foo, elem);
    ...do something with f...
  }
```

* <mark style="color:blue;">**Type: struct hash\_iterator**</mark>
  * **Represents a position within a hash table.**
  * Calling any function that may modify a hash table, such as `hash_insert()` or `hash_delete()`, **invalidates** all iterators within that hash table.
  * Like `struct hash` and `struct hash_elem`, `struct hash_iterator` is **opaque**.
* <mark style="color:blue;">**Function: void hash\_first (struct hash\_iterator \*iterator, struct hash \*hash)**</mark>
  * **Initializes&#x20;**_**iterator**_ **to just before the first element in** **hash.**
* <mark style="color:blue;">**Function: struct hash\_elem \*hash\_next (struct hash\_iterator \*iterator)**</mark>
  * **Advances&#x20;**_**iterator**_ **to the next element in&#x20;**_**hash**_**, and returns that element.** Returns a null pointer if no elements remain.
  * After `hash_next()` returns null for _iterator_, calling it again yields undefined behavior.
* <mark style="color:blue;">**Function: struct hash\_elem \*hash\_cur (struct hash\_iterator \*iterator)**</mark>
  * **Returns the value most recently returned by `hash_next()` for&#x20;**_**iterator**_**.**
  * Yields undefined behavior after `hash_first()` has been called on _iterator_ but before `hash_next()` has been called for the first time.

## Hash Table Example

Suppose you have a structure, called `struct page`, that you want to put into a hash table.

* First, define `struct page` to include a `struct hash_elem` member:

```c
struct page
  {
    struct hash_elem hash_elem; /* Hash table element. */
    void *addr;                 /* Virtual address. */
    /* ...other members... */
  };
```

* We **write a hash function and a comparison function** using _addr_ as the key. A pointer can be hashed based on its bytes, and the `<` operator works fine for comparing pointers:

```c
/* Returns a hash value for page p. */
unsigned
page_hash (const struct hash_elem *p_, void *aux UNUSED)
{
  const struct page *p = hash_entry (p_, struct page, hash_elem);
  return hash_bytes (&p->addr, sizeof p->addr);
}

/* Returns true if page a precedes page b. */
bool
page_less (const struct hash_elem *a_, const struct hash_elem *b_,
           void *aux UNUSED)
{
  const struct page *a = hash_entry (a_, struct page, hash_elem);
  const struct page *b = hash_entry (b_, struct page, hash_elem);

  return a->addr < b->addr;
}
```

(The use of `UNUSED` in these functions' prototypes suppresses a warning that _aux_ is unused. See section [Function and Parameter Attributes](../../getting-started/debug-and-test/debugging.md#function-and-parameter-attributes), for information about `UNUSED`)

* Then, we can **create a hash table** like this:

```c
struct hash pages;

hash_init (&pages, page_hash, page_less, NULL);
```

* Now we can manipulate the hash table we've created. If `p` is a pointer to a `struct page`, we can **insert** it into the hash table with:

```cpp
hash_insert (&pages, &p->hash_elem);
```

If there's a chance that pages might already contain a page with the same _addr_, then we should check `hash_insert()`'s return value.

* To **search** for an element in the hash table, use `hash_find()`. This takes a little setup, because `hash_find()` takes an element to compare against. Here's a function that will find and return a page based on a virtual address, assuming that _pages_ is defined at file scope:

```cpp
/* Returns the page containing the given virtual address,
   or a null pointer if no such page exists. */
struct page *
page_lookup (const void *address)
{
  struct page p;
  struct hash_elem *e;

  p.addr = address;
  e = hash_find (&pages, &p.hash_elem);
  return e != NULL ? hash_entry (e, struct page, hash_elem) : NULL;
}
```

`struct page` is allocated as a local variable here on the assumption that it is fairly small. Large structures should not be allocated as local variables. See section [Struct thread](threads.md#struct-thread), for more information.

* A similar function could **delete** a _page_ by _address_ using `hash_delete()`.

## Auxiliary Data

In simple cases like the example above, there's no need for the _aux_ parameters. In these cases, just **pass a null pointer** to `hash_init()` for _aux_ and ignore the values passed to the hash function and comparison functions. (You'll get a compiler warning if you don't use the _aux_ parameter, but you can turn that off with the `UNUSED` macro, as shown in the example, or you can just ignore it.)

_**aux**_ **is useful when you have some property of the data in the hash table is both constant and needed for hashing or comparison, but not stored in the data items themselves.** For example, if the items in a hash table are fixed-length strings, but the items themselves don't indicate what that fixed length is, you could pass the length as an _aux_ parameter.

## Synchronization

**The hash table does not do any internal synchronization.** **It is the caller's responsibility to synchronize calls to hash table functions.**

* In general, any number of functions that examine but do not modify the hash table, such as `hash_find()` or `hash_next()`, may execute simultaneously.
* However, these functions cannot safely execute at the same time as any function that may modify a given hash table, such as `hash_insert()` or `hash_delete()`, nor may more than one function that can modify a given hash table execute safely at once.

**It is also the caller's responsibility to synchronize access to data in hash table elements.** How to synchronize access to this data depends on how it is designed and organized, as with any other data structure.
