# Customzing Python

A series of step-by-step of tutorials of CPython customizing. Each focuses on understanding CPython internals, and highlights different parts of the interpreter. 

## Pipe operator
We will  add an `|>` operator to the CPython interpreter to perform bash-like chain calls.

```python
>>> [1,2] |> map(lambda x:x*2) |> list()

[2, 4]
```

The tutorial covers python grammar, parsing and and compilation to bytecode. 

**[Read the tutorial](/pipe.md)**

## Tail call optimization
The goal is to implement tail-call optimization in the python inteprtetier. 
```python
>>> def u(i):
        if i == 0:
            return 'a'
        return u(i-1)

>>> u(10000)
'a'
```
The tutorial covers creation a new bytecode, flowgraph optimization, and evaluation of Python programs.

**[Read the tutorial](/tail.md)**

---
Pavel Kolchanov, 2024
