# Customzing Python

A list (n=1) of step-by-step of tutorials of Python customizing. Each focuses on understanding Cpython internals.

## Pipe operator
The goal is to add an operator to perfom bash-like chain calls:
```python
>>> [1,2] |> map(lambda x:x*2) |> list()

[2, 4]
```

[Check the tutorial](/pipe.md)

---
Pavel Kolchanov, 2024