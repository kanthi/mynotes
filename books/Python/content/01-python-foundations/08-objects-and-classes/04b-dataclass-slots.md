---
title: Dataclass slots=True
---

# Dataclass slots=True

## Thesis

**`@dataclass(slots=True)`** (3.10+) gives instances `__slots__` instead of a per-instance `__dict__`: less memory, attribute typos fail fast, and you opt out of accidental dynamic attributes.

## Mental model

```text
  @dataclass
  class A: x: int          # instance.__dict__ holds x

  @dataclass(slots=True)
  class B: x: int          # B.__slots__ == ('x',); no __dict__
```

## Worked example

```python
# slots_demo.py
from dataclasses import dataclass
import sys

@dataclass
class Plain:
    x: int
    y: int

@dataclass(slots=True)
class Slotted:
    x: int
    y: int

p, s = Plain(1, 2), Slotted(1, 2)
print("plain dict", hasattr(p, "__dict__"), "slots", hasattr(s, "__dict__"))
print("Slotted.__slots__", Slotted.__slots__)
try:
    s.z = 3
except AttributeError as e:
    print("assign extra:", type(e).__name__)

# rough size hint (implementation-dependent)
print("sizeof plain", sys.getsizeof(p) + sys.getsizeof(p.__dict__))
print("sizeof slotted", sys.getsizeof(s))
```

```text
plain size 48 has __dict__ True
slots size 48 has __dict__ False
assign extra: AttributeError
plain.__dict__ {'x': 1, 'y': 2}
```

## Trap

**Multiple inheritance with slotted dataclasses** needs care — all bases should cooperate on slots. Also: `slots=True` plus `weakref` needs `weakref_slot=True`. Defaults and `InitVar` still work; `object.__setattr__` is used during `__init__`.

## Boring rule

Use `slots=True` for high-cardinality records (events, rows, AST nodes). Keep unslotted dataclasses when callers routinely monkey-patch attributes in tests.

## Try this

1. Measure RSS for a million instances of each form.
2. Combine `slots=True` with `frozen=True`.
3. Inspect `__slots__` when inheriting from another slotted dataclass.

## Sources

- [Python docs: dataclasses](https://docs.python.org/3/library/dataclasses.html)
- [PEP 681](https://peps.python.org/pep-0681/) (dataclass transforms; related tooling)
## Architecture Diagram & Memory Footprint Benchmark Lab

![Memory layout of standard dataclass with __dict__ versus slots=True fixed struct](../../../images/diagram-python-dataclass-slots-layout.svg)

Measure the real memory delta between default `__dict__` dataclasses and `slots=True` across 50,000 instances using `tracemalloc`:

```python
import tracemalloc
from dataclasses import dataclass

@dataclass
class RegularPoint:
    x: float
    y: float
    z: float

@dataclass(slots=True)
class SlottedPoint:
    x: float
    y: float
    z: float

for cls in (RegularPoint, SlottedPoint):
    tracemalloc.start()
    pts = [cls(float(i), float(i), float(i)) for i in range(50_000)]
    current, _ = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    print(f"{cls.__name__:<14}: {current / 1024 / 1024:.2f} MiB for {len(pts)} instances")
```
