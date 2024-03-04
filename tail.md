> [!WARNING]
> I'm new to Python internals, so the tutorial may contain mistakes and clunky solutions.

# Tail call optimization
A tail call happens when a function calls another as its last action, so it has nothing else to do: 
```python
def g():
   return f()
```
Tail call optimization eliminates the need for adding a new stack frame to the call stack. This is useful for writing recursive functions:

```python
def fact(n, acc):
    if n == 1:
        return acc
    return fact(n-1, acc*n)
```

Altrough Guido Van Rossum [considers](https://neopythonic.blogspot.com/2009/04/final-words-on-tail-calls.html ) tail call optimizatoin as unpythonic, it's interesting to implement it for educational purposes. Let's start.

## Plan 
We are going to modify three steps of python inperpreter:

1. Introduce a new `TAIL_CALL` bytecode opeartor to python vm
2. Add an optimization to compiler, that inserts `TAIL_CALL` to code
3. Implement an interpretator for the new bytecode

## Prereq­ui­sites
Clone the CPython repo and checkout to a new branch.
```bash
$ git clone git@github.com:python/cpython.git && cd cpython
$ git checkout tags/v3.12.1 -b tail-call
```

## A new bytecode instruction

#### About bytecodes
Python source code is compiled into bytecode.  Bytecode is a set of insturctions for the python vm. 
For example, check how `f(a, b)` is represented:

```python
>>> import dis
>>> dis.dis("f(a, b)")
  0           0 RESUME                   0

  1           2 PUSH_NULL
              4 LOAD_NAME                0 (f)
              6 LOAD_NAME                1 (a)
              8 LOAD_NAME                2 (b)
             10 PRECALL                  2
             14 CALL                     2
             24 RETURN_VALUE
```

These instructions are telling an interpreter to:

1. Load a function to a value stack using `LOAD_NAME`
2. Load value of `a` to the value stack using `LOAD_NAME`
3. Load value of `b` to the value stack using `LOAD_NAME`
4. Call the function using `CALL` with `2` arguments.


#### `TAIL_CALL` defenition
Let's introduce a new bytecode instruction. The `Python/bytecodes.c` contains defenitions and interpretations of python bytecodes. It's written in a custom syntax. 

Since we're interested in calls, let's introduce an `TAIL_CALL` by blank copy of regular `CALL`

```diff
  macro(CALL) = _SPECIALIZE_CALL + unused/2 + _CALL;
+ macro(TAIL_CALL) =  _SPECIALIZE_CALL + unused/2 + _CALL;
```

Next, run `make regen-cases` to translate the `Python/bytecodes.c` to a propper c code. Let's have a look what is changed.

First, it defined a new bytecode. For example, in the `Include/opcode_ids.h`

```diff
+#define TAIL_CALL                              116
```

Second, it defined how to interpret the new bytecode in the `Python/generated_cases.c.h`. This file contains functions for the interptetiner. 

```diff
+        TARGET(TAIL_CALL) {
+            _Py_CODEUNIT *this_instr = frame->instr_ptr = next_instr;
+            next_instr += 4;
+            INSTRUCTION_STATS(TAIL_CALL);
+            PyObject **args;
+            PyObject *self_or_null;
+            PyObject *callable;
+            PyObject *res;
...
```

#### Importlib
The other important step is to update the imporlib after introducing the new bytecode. Since some Python libraries are frozen and linked to the Python interpreter, it is necessary to freeze them after any bytecode updates.

Change `MAGIC_NUMBER` constant in the `Lib/importlib/_bootstrap_external.py`. This will lead to .pyc files with the old `MAGIC_NUMBER` to be recompiled by the interpreter on import. 

Then, run `make regen-importlib`. 

<!-- todo importlib  -->


## Flowgraph optimization

According to the
[CPython devguide](https://devguide.python.org/internals/compiler/#control-flow-graphs)  control flow graph is an intermediate result of python source code compilation. CFGs are usually one step away from final code output, and are perfect place to perfom a code optimization. 

Look to the to the `Python/flowgraph.c/_PyCfg_OptimizeCodeUnit`. The function updates code graph: removes unused condsts, insterts super instructions, etc.

```c
int
_PyCfg_OptimizeCodeUnit(cfg_builder *g, PyObject *consts, PyObject *const_cache,
                        int nlocals, int nparams, int firstlineno)
{
    ...
    RETURN_IF_ERROR(optimize_cfg(g, consts, const_cache, firstlineno));
    RETURN_IF_ERROR(remove_unused_consts(g->g_entryblock, consts));
    RETURN_IF_ERROR(
        add_checks_for_loads_of_uninitialized_variables(
            g->g_entryblock, nlocals, nparams));
    insert_superinstructions(g);
    ...
}
```

Insert a new optimization `optimize_tail_call`:

```diff
    RETURN_IF_ERROR(
        add_checks_for_loads_of_uninitialized_variables(
            g->g_entryblock, nlocals, nparams));
    insert_superinstructions(g);
+   optimize_tail_call(g);
```

And define it. The goal is to replace CALL→RETURN sequence to TAIL_CALL→RETURN:

```c

static void
optimize_tail_call(cfg_builder *g)
{
    for (basicblock *b = g->g_entryblock; b != NULL; b = b->b_next) {

        for (int i = 0; i < b->b_iused; i++) {
            cfg_instr *inst = &b->b_instr[i];
            int nextop = i+1 < b->b_iused ? b->b_instr[i+1].i_opcode : 0;
            if (inst->i_opcode == CALL && nextop == RETURN_VALUE) {
                    INSTR_SET_OP1(inst, TAIL_CALL, inst->i_oparg);
            }
  
        }
    }
}
```

Let's check if the new optimizaiton is working.

Recompile cpython with `make -j6`

And check the new optimization with `dis` module:

```python
>>> def f():
...    return g():

>>> dis.dis(f)

1           RESUME                   0

2           LOAD_GLOBAL              1 (g + NULL)
            TAIL_CALL                0
            RETURN_VALUE
```

Perfect. Let's move to the final step. 

##  Implement an interpretator for the new bytecode
As previously mentioned, the `Python/bytecodes.c` file contains definitions and interpretations of Python bytecodes. 
Currently, we are using a blank copy of `CALL` interptetier as `TAIL_CALL`. First, let's understand how it works.


#### How `CALL` works 
A bit of terminology. Call frame is a structure that represents a function call's execution context: local variables, function arguments etc.

Value stack is a list of pointers to python objects, that instructions operates. For example, 

The `Python/bytecodes.c/_CALL` manipulaets both structures. In summary, it does three things:
1. Creates a new call frame and pushes it to the call stack
2. Consumes arguments from the current frame's value stack
3. Passes control to the new frame

```c
//  Creates a new call frame and pushes it to the call stack
int code_flags = ((PyCodeObject*)PyFunction_GET_CODE(callable))->co_flags;
PyObject *locals = code_flags & CO_OPTIMIZED ? NULL : Py_NewRef(PyFunction_GET_GLOBALS(callable));
_PyInterpreterFrame *new_frame = _PyEvalFramePushAndInit(
    tstate, (PyFunctionObject *)callable, locals,
    args, total_args, NULL
);

// Consumes arguments from the current frame's value stack
STACK_SHRINK(oparg + 2);

if (new_frame == NULL) {
    GOTO_ERROR(error);
}
// Updates the current frame's return offset
frame->return_offset = (uint16_t)(next_instr - this_instr);
// Passes control to the new frame
DISPATCH_INLINED(new_frame);
```

Simple and easy. Let's move to the next step. 
#### `TAIL_CALL` interptetier
To create a `TAIL_CALL` interptetier we are going to change a few things in the regular `CALL`.

We need to drop the current frame before creating a new call frame. However, because references to arguments are stored in the current dying frame, we need to store them before dropping. And clean them up after creating the new frame.

Let's move to `Python/bytecodes.c/_TAIL_CALL`. 

First, save args. Since CPython uses it's own memory allocator, use `PyMem_Malloc` to allocate memory. 

```diff
// Check if the call can be inlined or not
if (Py_TYPE(callable) == &PyFunction_Type &&
    tstate->interp->eval_frame == NULL &&
    ((PyFunctionObject *)callable)->vectorcall == _PyFunction_Vectorcall)
{
+ PyObject **newargs = PyMem_Malloc(sizeof(PyObject*) * (total_args));
+ Py_ssize_t j, n;
+ n = total_args;

+ for (j = 0; j < n; j++)
+ {
+     PyObject *x = args[j];
+     newargs[j] = x;
+ }
```

Next, drop the current call frame. The snippet is a copy of the `Python/bytecodes.c/POP_FRAME` instruction.
```diff
+ STACK_SHRINK(oparg + 2);
+ _Py_LeaveRecursiveCallPy(tstate);
+_PyFrame_SetStackPointer(frame, stack_pointer);
+ _PyInterpreterFrame *dying = frame;
+ frame = tstate->current_frame = dying->previous;
+_PyEval_FrameClearAndPop(tstate, dying);
+ LOAD_SP();
```

Init a new frame using callable and new args.
```diff 
_PyInterpreterFrame *new_frame = _PyEvalFramePushAndInit(
    tstate, (PyFunctionObject *)callable, locals,
-    args, total_args, NULL
+    newargs, total_args, NULL
);
```
Clean up the argument stash:

```diff
+ PyMem_Free(newargs);
```
And pass contoll to the new frame:

```c
DISPATCH_INLINED(new_frame);
```


Recompile CPython with `make regen-cases && make regen-importlib && make -j6` and test the new operator. 


## Final check

```python
>>> def fact(n, acc):
...    if n == 1:
...        return acc
...    return fact(n-1, acc*n)

>>>fact (1500,1)

48119977967797748601669900935...
```

<!-- 
All together:

```c
if (Py_TYPE(callable) == &PyFunction_Type &&
                tstate->interp->eval_frame == NULL &&
                ((PyFunctionObject *)callable)->vectorcall == _PyFunction_Vectorcall)
            {
    int code_flags = ((PyCodeObject*)PyFunction_GET_CODE(callable))->co_flags;
    PyObject *locals = code_flags & CO_OPTIMIZED ? NULL : Py_NewRef(PyFunction_GET_GLOBALS(callable));
    Py_INCREF(callable);
    PyObject **newargs = PyMem_Malloc(sizeof(PyObject*) * (total_args));
    Py_ssize_t j, n;
    n = total_args;

    for (j = 0; j < n; j++)
    {
        PyObject *x = args[j];
        newargs[j] = x;
    }
    STACK_SHRINK(oparg + 2);

    _Py_LeaveRecursiveCallPy(tstate);
    _PyFrame_SetStackPointer(frame, stack_pointer);
    _PyInterpreterFrame *dying = frame;
    frame = tstate->current_frame = dying->previous;
    _PyEval_FrameClearAndPop(tstate, dying);
    LOAD_SP();
    _PyInterpreterFrame *new_frame = _PyEvalFramePushAndInit(
        tstate, (PyFunctionObject *)callable, locals,
        newargs, total_args, NULL
    );
    PyMem_Free(newargs);
    
    if (new_frame == NULL) {
        GOTO_ERROR(error);
    }
    
        DISPATCH_INLINED(new_frame);
}
``` -->