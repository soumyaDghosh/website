+++
title = 'GSOC: Until Midterm, August 2025'
date = 2025-08-18T11:00:00+05:30
images = ['gsoc.png']
keywords = ['OpenPrinting', 'Python', 'PyCups', 'GSOC', '2025']
tags = ['OpenPrinting', 'GSOC', '2025']
+++

![](python-spewing-printers.png)
_Representative and generated_

**So long of GSoC and No blog yet? Why Soumya?**

Yeah, yeah, I know. It’s been a while, and no blog posts yet. But hey, between wrestling with **CFFI + C + Python** and **libcups with its 150+ APIs**, blogging kind of took the back seat. But enough excuses - let’s dig into what’s been cooking.

---

## A little history (with C in it)

Back in the day (almost 15 years ago!), [**Tim Waugh**](https://github.com/twaugh) wrote the first version of `pycups` as a **C extension module for Python**. That worked well, but like all old code, it aged... let’s just say, not like fine wine. After Tim, [**Zdenek**](https://github.com/zdohnal) took over as maintainer, but with multiple projects of OpenPrinting and other projects in the mix, there wasn’t much room to modernize PyCups2 for him.

Fast forward to the release of **libcups3** — OpenPrinting needed a shiny new `pycups`, but the old codebase was a bit too… tangled. It was all C extensions, hard to read, harder to extend, and basically a maintenance nightmare. That’s where I stepped in and pitched a **hybrid solution**:

👉 **Use CFFI (C Foreign Function Interface)** to call libcups APIs **directly** from Python.

👉 Keep the library written mostly in **pure Python**.

Why? Because:

* Python code is easier to read.
* Easier to extend and manage.
* And let’s be honest — no one wants to debug a decade-old C extension unless they absolutely have to.

---

## From Classes to Mixins (and back again)

At first, I thought of a nice, clean architecture: a **base class** with protocol-specific subclasses. For example, all HTTP APIs neatly under one class.

But then reality hit: **CUPS talks to everything over HTTP.** Even IPP (Internet Printing Protocol) APIs need HTTP under the hood. So that design collapsed faster than my sleep schedule during GSoC.

Instead, I took inspiration from `pycups2` and built a single big [**`Connection` class**](https://github.com/soumyaDghosh/pycups/blob/feat-refactor/cups/connection/__init__.py). To keep it sane, I split logic into **Mixin classes**. These Mixins aren’t meant to be used directly — they just inject methods into `Connection`, keeping the big class from becoming a 10,000-line monster.

Now you can do things like:

```python
import cups
conn = cups.Connection()
conn.encryption = HttpEncryption.ALWAYS
```

Here, the `encryption` setter just wraps a C API call via CFFI using the powerful [property decorator](https://www.linkedin.com/posts/soumyadghosh_why-i-fell-in-love-with-property-while-activity-7356164166472118275-ysfk?utm_source=share&utm_medium=member_desktop&rcm=ACoAAEb88BYBvz0xFt2Nk4WyZerJF0FdcaAdvog):

```python
def setEncryption(encryption: HttpEncryption):
    c_encryption = _ffi.cast("http_encryption_t", encryption.value)
    _lib.cupsSetEncryption(c_encryption)
```

Pythonic on the outside, C-powered on the inside. 🚀

---

## Making Enums Less Painful

One of the reasons this feels so much nicer is because I also **simplified enums**.

In the old world, you couldn’t even do:

```python
conn = cups.Connection()
conn.encryption
```

Instead, you’d get:

```
Traceback (most recent call last):
  File "<python-input-2>", line 1, in <module>
    conn.encryption
AttributeError: 'cups.Connection' object has no attribute 'encryption'
```

Why? Because the old C extension didn’t expose things as nice Python objects.

But now, thanks to `IntFlag` enums, you *can* read and write them like proper Python attributes:

```python
conn.encryption
HttpEncryption.IF_REQUESTED
```

And even change them dynamically:

```python
conn.encryption = cups.enums.HttpEncryption.ALWAYS
conn.encryption
<HttpEncryption.ALWAYS: 3>
```

This works because I mapped the old C macros into Python enums like this:

```python
from enum import IntFlag
from cups import _cups

_lib = _cups.lib

class HttpEncryption(IntFlag):
    IF_REQUESTED = _lib.HTTP_ENCRYPTION_IF_REQUESTED
    NEVER = _lib.HTTP_ENCRYPTION_NEVER
    REQUIRED = _lib.HTTP_ENCRYPTION_REQUIRED
    ALWAYS = _lib.HTTP_ENCRYPTION_ALWAYS
```

No more magic numbers (`0`, `1`, `2`, `3`). Instead, you get **readable, type-safe, Pythonic enums**. And since they’re `IntFlag`, they behave just like integers when passed into C, but with the bonus of clarity and bitwise flexibility if you need it.

---

## Meet `cupsDest` – our star data structure

CUPS has plenty of C structures, and in pycups3 I wrapped many of them into Python `dataclass`es. The key one is **`cups_dest_t`**, represented as a Python class named `cupsDest`.

To make things consistent, I created a base class `cupsBaseClass`, which provides two essential methods:

* `cffi_new` → allocates memory in C.
* `cffi_free` → frees memory (though [as per docs,](https://cffi.readthedocs.io/en/stable/using.html#working-with-pointers-structures-and-arrays) CFFI usually handles this anyway).

Each subclass (like `cupsDest`) defines three extra attributes:

* `ffi_name` → the C type name.
* `ffi_free` → the C function to free memory for this type.
* `ffi_value` → the raw `CData` object.

Here’s the Python version of `cups_dest_t`:

```python
@dataclass(repr=False)
class cupsDest(cupsBaseClass):
    @property
    def name(self) -> str:
        return str(_bytes_to_value(self.ffi_value[0].name))

    @property
    def instance(self) -> Optional[str]:
        return _bytes_to_value(self.ffi_value[0].instance)

    @property
    def options(self) -> Dict[str, cupsOption]:
        return cupsOption.from_cffi_list(
            opts=self.ffi_value[0].options, count=self.ffi_value[0].num_options
        )

    @property
    def is_default(self) -> bool:
        return self.ffi_value[0].is_default

    ffi_name = "cups_dest_t"
    ffi_free = "cupsFreeDests"

    @classmethod
    def from_cffi_list(cls, dests: "cupsDest", count: int) -> "Dict[str, cupsDest]":
        results: Dict[str, cupsDest] = {}
        for i in range(count):
            new_dest: Any = dests.ffi_value[0][i]
            results[str(_bytes_to_value(new_dest.name))] = cupsDest.from_cffi(
                dest=new_dest
            )

        return results

    @classmethod
    def to_cffi_list(cls, dests: "Dict[str, cupsDest]") -> Any:
        count = len(dests)
        c_dests = _ffi.new(f"cups_dest_t[{count}]")

        for i, (_, dest) in enumerate(dests.items()):
            c_dests[i].name = _ffi.new("char[]", dest.name.encode("utf-8"))
            c_dests[i].instance = _ffi.new(
                "char[]",
                dest.instance.encode("utf-8") if dest.instance is not None else b"",
            )
            c_dests[i].is_default = dest.is_default
            c_dests[i].num_options = len(dest.options)
            c_dests[i].options = cupsOption.to_cffi_list(dest.options)

        return c_dests
```

This class makes it super simple to move between Python and C worlds. For example, helper methods like `from_cffi_list` and `to_cffi_list` make it possible to convert lists of printers back and forth.

And since it’s a `dataclass`, I don’t need to write boilerplate `__init__`s — less code, more sanity.

---

## Wrapping it up

So yeah, between **CFFI wrappers**, **Python dataclasses**, and **enum simplifications**, we’ve got a design that’s:

* Easier to understand,
* Easier to extend,
* And much less scary than raw C extensions.

Of course, this is just scratching the surface. There are *lots* of other details — structures, enums, mixins, helpers — and things can get really complicated if you let them.

But if I had to sum it up: these three pillars —

1. **Connection + Mixins**
2. **Dataclass wrappers for C structures**
3. **Simplified enums**

— are the heart of the new pycups3 architecture.

And yes, the work is still ongoing. So stay tuned… or better yet, stay caffeinated. ☕
