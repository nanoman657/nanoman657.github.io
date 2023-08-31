---
layout: post
title: Python Type Checking with Command Line Inputs
tags: [type hints, python, command line]
---
Type hinting in Python allows you to annotate the data type of variables and functions. While type hinting is not required for the dynamically typed language, this feature enables intelligent type suggestions, type checking with mypy, and improved code clarity. In a recent scenario, I encountered the need to type values from command line inputs. This led to the discovery of new features and limitations of Python's type hinting.

### Typing with Command-Line Input

Suppose we have a function that operates on user input from the command line. Consider the example below, where we aim to pluralize an English word like "pie" based on a given quantity:


```bash
python pluralize.py pie 2 # Desired output: "2 pies"
```

The corresponding Python code, which utilizes two `TypeAlias`es to enhance type-checking, is provided below. The `get_args` function returns a tuple of possible values for `SingularWord`. We can simply check to see if the raw input from the command line matches one of our expected possibilities. You can copy the code and enter `pluralize.py pie 2` to see that it will successfully run. However, this approach is not without its limitations.

```python
from argparse import Namespace
from sys import argv
from typing import TypeAlias, Literal, Union, get_args


SingularWord: TypeAlias = Literal["pie", "pastry"]
PluralWord: TypeAlias = Literal["pies", "pastries"]


class PluralizeNamespace(Namespace):
    word: SingularWord
    quantity: int


def pluralize(word: SingularWord, quantity: int) -> str:
    plural_map: dict = dict(zip(get_args(SingularWord), get_args(PluralWord)))

    return f"{quantity} {plural_map[word] if quantity != 1 else word}"


raw_word: Union[SingularWord, str] = argv[1]
raw_num: str = argv[2]

if raw_word in get_args(SingularWord):
    processed_word: SingularWord = raw_word
    processed_num: int = int(raw_num)
    print(pluralize(processed_word, processed_num))
else:
    raise ValueError("Unknown word requested for pluralization")
```

### Using `argparse` to Regulate Types
 When we check the types, an area of improvement is quickly identified. I use mypy to type check my code, and running `mypy pluralize.py` in the terminal yields an error indicating that the type assignment is incompatible:
 
  `error: Incompatible types in assignment (expression has type "Union[Literal['pie', 'pastry'], str]", variable has type "Literal['pie', 'pastry']")  [assignment]`
  
  There are two reasons for this. First, the mypy `reveal_type` function reveals that get_args returns Any. That effectively means we we cannot infer the types of values coming from get_args. Secondly, there is no way to convert `raw_word` into a `SingularWord`, since `SingularWord` is just a type hint, and not a data type evaluated at runtime like `int`.

To overcome this challenge, Python's `argparse` module can be used to regulate proper typing consistently with mypy. We start by defining the command arguments and using `choices` to restrict the user's input to options specified by `SingularWord`.

```python
from argparse import ArgumentParser, Namespace
from typing import get_args

# ... other code omitted for brevity

parser: ArgumentParser = ArgumentParser(description="Converts a word and quantity to plural statement")
parser.add_argument("word", choices=get_args(SingularWord))
parser.add_argument("quantity", type=int)

arguments: Namespace = parser.parse_args()
word_: SingularWord = arguments.word
quantity_: int = arguments.quantity

plural_statement: str = pluralize(word_, quantity_)
print(plural_statement)

```

The above code ensures that only valid word choices are permitted, and is consistent with `mypy`'s standards. Now, you may be wondering why `word_` is typed when it has to be one of the choices we defined. This relates to some of the limitations of python's argument handling.

In short, `arguments` is a `Namespace`, so any values extracted from it will need to be typed. This is even true for arguments where we have specified the type, like `quantity`. If you try using `reveal_type(arguments.quantity)`, mypy will indicate that `arguments.quantity` is actually an `Any` type. With a little more coding, we can restrict this value to the exact type we want.

### Namespace Typing
`parse_args` has an argument called `namespace` that we can use to insert custom namespaces. In this case, our custom namespace will have the exact typing we want.
```python
from argparse import Namespace


singular_word: TypeAlias = Literal["pie", "pastry"]
plural_word: TypeAlias = Literal["pies", "pastries"]


class PluralizeNamespace(Namespace):
    word: SingularWord
    quantity: int
```

We simply need to specify that our custom namespace will be used in lieu of the generic one. 

```python
arguments: Namespace = parser.parse_args(namespace=PluralizeNamespace)
```

Now, if you try to reveal the type of `arguments.quantity` and `arguments.word`, they will be `int` and `singular_word`, which is exactly what we want.

### Conclusion
Typing hinting is an immensely helpful feature of python that powers mypy's ability to statically check the types used in code. The practice becomes more complex when using command line arguments as input to functions, since the inputs are up to the user. `argparse` is an excellent module for constraining the arguments to values usable by your code. When you combine `ArgumentParser` with your own typed custom `Namespace` the entire code can be completely typed.

### Resources
1. [Source code](https://gist.github.com/nanoman657/93abb6c81dbf81e78f7c6847d5c1818a) : 3 files containing the examples used in this post.
	1. `pluralize.py`: Initial attempt
	2. `pluralize_with_argparse.py`: Initial attempt with argparse
	3. `pluralize_namespace_typing.py`: Final version of the code with custom namespace
2. [Python type hinting](https://docs.python.org/3/library/typing.html): Python's official documentation on type hinting
3. [mypy](https://mypy-lang.org/): An optional static type checker for python
4. [argparse](https://docs.python.org/3/library/argparse.html): Python builtin module for parsing command line arguments
