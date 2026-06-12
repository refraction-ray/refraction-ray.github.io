---
layout: post
title: "PyTrees Are Different in JAX, PyTorch and TensorFlow"
date: 2026-06-12
excerpt: "Learn the subtle differences between PyTree semantics across frameworks"
tags: [python, machine learning]
comments: true
---


* toc
{:toc}

## Introduction

PyTrees look deceptively simple. You flatten a nested Python object into leaves, keep a structure descriptor, and later rebuild or map over the same shape. That abstraction is powerful enough to carry optimizer states, model parameters, batched inputs, gradients, and sharding annotations. It is also just ambiguous enough that three major frameworks implement three subtly different languages under the same idea.

This note compares JAX `jax.tree_util`, PyTorch `torch.utils._pytree`, and TensorFlow `tf.nest`. I tested the behavior in two environments: an older stack with JAX 0.4.35, PyTorch 2.2.2, TensorFlow 2.20.0, and a newer stack with JAX 0.10.0, PyTorch 2.12.0, TensorFlow 2.21.0. Most flatten/unflatten semantics were stable across these versions. The main version-sensitive result is PyTorch: `_pytree.tree_map` in 2.2.2 accepts only one pytree, while 2.12.0 supports multiple pytrees and behaves much closer to JAX prefix-style mapping.

The short version: JAX treats pytrees as a transformation language, PyTorch is converging toward that model in `torch.func`, and TensorFlow exposes a broader nested-structure utility through `tf.nest`. Those differences show up exactly where backend-agnostic libraries usually hurt: `None`, dictionary order, custom containers, `tree_map`, autodiff, and vectorization.

## The Shape Of The APIs

The three APIs have the same surface story but not the same contract.

```python
from jax import tree_util as jtu
from torch.utils import _pytree as tpu
import tensorflow as tf

leaves, treedef = jtu.tree_flatten(tree)
tree = jtu.tree_unflatten(treedef, leaves)
tree = jtu.tree_map(f, *trees)

leaves, spec = tpu.tree_flatten(tree)
tree = tpu.tree_unflatten(leaves, spec)
tree = tpu.tree_map(f, tree)          # PyTorch 2.2.2
tree = tpu.tree_map(f, *trees)        # PyTorch 2.12.0

leaves = tf.nest.flatten(tree)
tree = tf.nest.pack_sequence_as(structure, leaves)
tree = tf.nest.map_structure(f, *structures)
```

Flattening means "which objects are leaves?" Unflattening means "what metadata is needed to reconstruct the original container?" Mapping means "what does it mean for several structures to match?" Those three questions are where the frameworks diverge.

JAX calls its structure descriptor a `PyTreeDef`, so `treedef` is the conventional variable name. PyTorch calls the analogous descriptor a `TreeSpec`, so examples and internals often name it `spec`. Conceptually they play the same role: they describe the container skeleton and the metadata needed to rebuild it from a flat leaf list. TensorFlow's `tf.nest` does not return a separate treedef object from `flatten`; instead, `pack_sequence_as` takes an existing nested `structure` as the template.

There is also a small argument-order trap. JAX unflattens as `tree_unflatten(treedef, leaves)`, while PyTorch unflattens as `tree_unflatten(leaves, spec)`. TensorFlow's equivalent is `pack_sequence_as(structure, leaves)`. 

## A Compact Map Of The Differences

| Case | JAX | PyTorch `_pytree` | TensorFlow `tf.nest` |
|---|---|---|---|
| Scalar | Leaf | Leaf | Leaf |
| `None` | Empty pytree, 0 leaves | Leaf | Leaf |
| `list`, `tuple` | Containers | Containers | Containers |
| namedtuple | Container, type-strict | Container, type-strict | Container, type-strict |
| plain `dict` order | Sorted keys | Insertion order | Sorted-key leaf order |
| `OrderedDict` order | Insertion order | Insertion order | Sorted-key leaf order |
| `defaultdict` order | Sorted keys | Insertion order | Sorted-key leaf order |
| `defaultdict.default_factory` | Preserved | Preserved | Preserved |
| custom `dict` subclass | Leaf unless registered | Leaf unless registered | Container |
| custom `list`/`tuple` subclass | Leaf unless registered | Leaf unless registered | Container |
| dataclass instance | Leaf unless registered | Leaf unless registered | Leaf by default |
| multi-arg `tree_map` | Supported, prefix semantics | PyTorch 2.2.2: not supported; PyTorch 2.12.0: supported with prefix semantics | Supported, strict same structure |
| unflatten arity mismatch | Raises `ValueError` | Raises `ValueError` | Raises `ValueError` |

The rest of the note explains why these rows matter.

## `None`: A Ghost Node In JAX, A Leaf Elsewhere

The cleanest way to feel the philosophical split is `None`. In JAX, `None` is not a value to map over. It is a zero-leaf structural marker.

```python
jtu.tree_flatten(None)
# leaves: []
# treedef: PyTreeDef(None)

jtu.tree_map(lambda x: ("mapped", x), None)
# None
```

In PyTorch and TensorFlow, `None` is a leaf.

```python
tpu.tree_flatten(None)
# [None]

tf.nest.flatten(None)
# [None]

tpu.tree_map(lambda x: ("mapped", x), None)
# ("mapped", None)

tf.nest.map_structure(lambda x: ("mapped", x), None)
# ("mapped", None)
```

The nested case makes the difference visible:

```python
tree = [1, None, 3]

jtu.tree_flatten(tree)[0]
# [1, 3]

tpu.tree_flatten(tree)[0]
# [1, None, 3]

tf.nest.flatten(tree)
# [1, None, 3]
```

If `None` means "optional value absent", JAX treats it structurally. If `None` means "a value in my tree", PyTorch and TensorFlow are closer to that intuition.

## Dictionaries: The Same Keys, Different Time Arrows

Plain `dict` is a container everywhere, but the traversal order differs. JAX sorts keys, PyTorch follows insertion order, and TensorFlow assigns leaves by sorted keys while preserving the original mapping order when rebuilding.

```python
tree = {"b": 2, "a": 1}

jtu.tree_flatten(tree)[0]
# [1, 2]   # a, then b

tpu.tree_flatten(tree)[0]
# [2, 1]   # b, then a

tf.nest.flatten(tree)
# [1, 2]   # a, then b
```

Replacing the leaves with `[10, 20]` shows the reconstruction contract:

```python
# JAX
{"a": 10, "b": 20}

# PyTorch
{"b": 10, "a": 20}

# TensorFlow
{"b": 20, "a": 10}
```

TensorFlow's result is the surprising one on first read. It maps values according to sorted keys, but prints in the original insertion order. The object order and the leaf assignment order are not the same concept.

Mixed incomparable key types are another consequence of sorting. JAX and TensorFlow fail on `{1: "one", "2": "two"}` because `1 < "2"` is not defined. PyTorch does not sort and therefore flattens this case in insertion order.

```python
jtu.tree_flatten({1: "one", "2": "two"})
# ValueError: Comparator raised exception while sorting pytree dictionary keys.

tf.nest.flatten({1: "one", "2": "two"})
# TypeError: '<' not supported between instances of 'str' and 'int'

tpu.tree_flatten({1: "one", "2": "two"})[0]
# ["one", "two"]
```

## Ordered Containers Are Not Just Dicts With Better Manners

`OrderedDict` has explicit order metadata, and JAX treats that metadata as part of the tree structure. PyTorch uses insertion order too. TensorFlow again uses sorted-key leaf assignment.

```python
from collections import OrderedDict

tree = OrderedDict([("b", 2), ("a", 1)])

jtu.tree_flatten(tree)[0]
# [2, 1]

tpu.tree_flatten(tree)[0]
# [2, 1]

tf.nest.flatten(tree)
# [1, 2]
```

All three preserve the `OrderedDict` type when rebuilding, but TensorFlow assigns replacement leaves by sorted key:

```python
tf.nest.pack_sequence_as(OrderedDict([("b", 2), ("a", 1)]), [10, 20])
# OrderedDict([("b", 20), ("a", 10)])
```

Multi-argument mapping reveals another difference. JAX rejects two `OrderedDict`s with the same keys but different order because the custom node metadata differs.

```python
a = OrderedDict([("b", 2), ("a", 1)])
b = OrderedDict([("a", 10), ("b", 20)])

jtu.tree_map(lambda x, y: (x, y), a, b)
# ValueError: Mismatch custom node data: ('b', 'a') != ('a', 'b')
```

TensorFlow accepts this and pairs by key while preserving the first structure's order. PyTorch 2.12.0 also accepts it and returns the same visible result:

```python
OrderedDict([("b", (2, 20)), ("a", (1, 10))])
```

## `defaultdict`: Losing The Type Changes Behavior

`defaultdict` is not a decorative subclass. It carries a `default_factory`, which changes lookup behavior.

```python
from collections import defaultdict

counter = defaultdict(int)
counter["missing"]
# 0

plain = {}
plain["missing"]
# KeyError
```

All three frameworks preserve the `default_factory`, but they disagree about leaf order just as with dictionaries.

```python
tree = defaultdict(int, {"b": 2, "a": 1})

jtu.tree_flatten(tree)[0]
# [1, 2]

tpu.tree_flatten(tree)[0]
# [2, 1]

tf.nest.flatten(tree)
# [1, 2]
```

Rebuilding with `[10, 20]` gives:

```python
# JAX
defaultdict(int, {"a": 10, "b": 20})

# PyTorch
defaultdict(int, {"b": 10, "a": 20})

# TensorFlow
defaultdict(int, {"b": 20, "a": 10})
```

This matters for any pure Python fallback. If it flattens a `defaultdict` as a mapping but reconstructs a plain `dict`, it is wrong, not merely imprecise.

## Custom Containers: Either Register Them Or Treat Them As Leaves

JAX and PyTorch are conservative about arbitrary subclasses. TensorFlow is more eager to recurse into sequence and mapping subclasses.

```python
class MyDict(dict):
    pass

class MyList(list):
    pass

class MyTuple(tuple):
    pass
```

JAX and PyTorch treat these as leaves unless explicitly registered:

```python
jtu.tree_flatten(MyDict({"b": 2, "a": 1}))[0]
# [MyDict({"b": 2, "a": 1})]

tpu.tree_flatten(MyList([1, 2]))[0]
# [MyList([1, 2])]
```

TensorFlow traverses them:

```python
tf.nest.flatten(MyDict({"b": 2, "a": 1}))
# [1, 2]

tf.nest.flatten(MyList([1, 2]))
# [1, 2]

tf.nest.flatten(MyTuple((1, 2)))
# [1, 2]
```

Namedtuple is the standard exception. All three frameworks recognize it as a structural container and preserve its type. They are also strict about namedtuple type matching: `Point(1, 2)` is not the same structure as `(1, 2)` or `RGB(1, 2)`.

## The Real Trap: `tree_map` Does Not Always Mean Same-Structure Map

JAX `tree_map` uses the first argument as the reference structure. Later arguments are flattened "up to" that structure. If the first tree has a leaf, the corresponding value in a later tree may be an entire subtree.

```python
jtu.tree_map(lambda x, y: (x, y), [1, 2], [[3], {"x": 4}])
# [(1, [3]), (2, {"x": 4})]
```

The first tree says: "I am a list of two leaves." Therefore the second tree only needs to be a list of two objects. Those objects are passed whole to the function.

The scalar case is even clearer:

```python
jtu.tree_map(lambda x, y: (x, y), 1, [2, 3])
# (1, [2, 3])

jtu.tree_map(lambda x, y: (x, y), [1, 2], 3)
# ValueError: Expected list, got 3.
```

PyTorch 2.12.0 behaves similarly:

```python
tpu.tree_map(lambda x, y: (x, y), [1, 2], [[3], {"x": 4}])
# [(1, [3]), (2, {"x": 4})]

tpu.tree_map(lambda x, y: (x, y), 1, [2, 3])
# (1, [2, 3])

tpu.tree_map(lambda x, y: (x, y), [1, 2], 3)
# ValueError: Node type mismatch; expected <class 'list'>, but got <class 'int'>.
```

PyTorch 2.2.2 did not support this multi-pytree call through `_pytree.tree_map`. TensorFlow supports multiple structures, but it requires strict structural equality:

```python
tf.nest.map_structure(lambda x, y: (x, y), [1, 2], [[3], {"x": 4}])
# ValueError: structures do not have the same nested structure
```

## Transform APIs: PyTree Support Is Not Just Flattening

Tree semantics matter most when they meet transforms. Here the frameworks differ again.

JAX transformations are natively pytree-based. `grad` accepts nested inputs and returns gradients with the same structure:

```python
import jax
import jax.numpy as jnp

def f(params):
    return params["x"] ** 2 + params["y"][0] ** 3

params = {"x": jnp.array(2.0), "y": [jnp.array(3.0)]}
jax.grad(f)(params)
# {"x": Array(4., dtype=float32), "y": [Array(27., dtype=float32)]}
```

JAX `vmap` accepts nested pytree inputs too:

```python
def g(params):
    return params["x"] + params["y"][0]

batched = {"x": jnp.arange(3.0), "y": [jnp.arange(3.0) + 10]}
jax.vmap(g)(batched)
# Array([10., 12., 14.], dtype=float32)
```

Because `None` is a zero-leaf node in JAX, it can sit inside a vmapped input without becoming a batched argument:

```python
def h(params):
    return params["x"]

jax.vmap(h)({"x": jnp.arange(3.0), "y": [None]})
# Array([0., 1., 2.], dtype=float32)
```

Classic PyTorch autograd is different. `torch.autograd.grad` expects tensors or gradient edges as `inputs`, not an arbitrary nested pytree:

```python
x = torch.tensor(2.0, requires_grad=True)
y = torch.tensor(3.0, requires_grad=True)
loss = x ** 2 + y ** 3

torch.autograd.grad(loss, (x, y))
# (tensor(4.), tensor(27.))

nested = {"x": x, "y": [y]}
torch.autograd.grad(loss, nested)
# RuntimeError: all inputs have to be Tensors or GradientEdges, but got str
```

The newer `torch.func` stack does understand nested pytree-like parameter structures:

```python
from torch.func import grad, vmap

def f(params):
    return params["x"] ** 2 + params["y"][0] ** 3

params = {"x": torch.tensor(2.0), "y": [torch.tensor(3.0)]}
grad(f)(params)
# {"x": tensor(4.), "y": [tensor(27.)]}

def g(params):
    return params["x"] + params["y"][0]

batched = {"x": torch.arange(3.0), "y": [torch.arange(3.0) + 10]}
vmap(g)(batched)
# tensor([10., 12., 14.])
```

TensorFlow's transform support follows `tf.nest`. `GradientTape.gradient` accepts nested sources and returns gradients in the same structure:

```python
x = tf.Variable(2.0)
y = tf.Variable(3.0)
nested = {"x": x, "y": [y]}

with tf.GradientTape() as tape:
    loss = nested["x"] ** 2 + nested["y"][0] ** 3

tape.gradient(loss, nested)
# {"x": tf.Tensor(4.0), "y": [tf.Tensor(27.0)]}
```

`tf.vectorized_map` also accepts nested input structures:

```python
def g(params):
    return params["x"] + params["y"][0]

batched = {"x": tf.range(3.0), "y": [tf.range(3.0) + 10]}
tf.vectorized_map(g, batched)
# tf.Tensor([10. 12. 14.], shape=(3,), dtype=float32)
```

`tf.function` accepts nested structures as ordinary function arguments:

```python
@tf.function
def f(params):
    return params["x"] ** 2 + params["y"][0] ** 3

f({"x": tf.constant(2.0), "y": [tf.constant(3.0)]})
# tf.Tensor(31.0, shape=(), dtype=float32)
```

The right summary is more specific: JAX transforms are pytree-native; PyTorch classic autograd is not, while `torch.func` is; TensorFlow transform APIs accept nested structures.

## Closing

PyTrees are a small abstraction with a long tail. Simple examples make every framework look compatible; real optimizer states, optional values, ordered mappings, custom containers, and transform APIs expose the differences quickly.
