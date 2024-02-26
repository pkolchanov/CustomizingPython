> [!WARNING]
> I'm a Python internals newbie, the turoial may contain mistakes and clunky solutions. 

# Tail call optimization
A tail call happens when a function calls another as its last action, so it has nothing else to do: 
```python
def g():
   return f()
```
Tail call optimization eliminates the need for adding a new stack frame to the call stack. This is useful for wiring recursion based funtions:

```python
def u(i):
    if i == 0:
         return 'a'
    return u(i-1)

u(10000)
```

<!-- todo factorial  -->

Altrough Guido Van Rossum [considering](https://neopythonic.blogspot.com/2009/04/final-words-on-tail-calls.html ) tail call optimizatoin as unpythonic, it's interesing to implement it in educational purposes. Let's start. 

## Plan 
We are going to modify three steps of python inperpreter:

1. Introduce a new bytecode opeartor to python vm
2. Update a compilation to bytecode
3. Implement an interpretator for the new bytecode

## Preparations
Clone the cpython python repo and checkout to a new branch.
```bash
$ git clone git@github.com:python/cpython.git && cd cpython
$ git checkout tags/v3.12.1 -b tail-call
```

## Bytecode instruction

#### Instruction defenition
Python source code is compiled into bytecode. Bytecode is a representation of a Python program. Bytecode contains of a set of insturctions for the python vm.  

EXAMPLE



Let's introduce a new bytecode instruction. The `Python/bytecodes.c` contains defenitions and implementations of python bytecodes. It's written in a custom syntax. 

For exmaple, `JUMP_FORWARD` instruction:
```c
inst(JUMP_FORWARD, (--)) {
    TIER_ONE_ONLY
    JUMPBY(oparg);
}
```

We're interested in calls. Let's introduce an `TAIL_CALL` by blank copy of regular `CALL`

```diff
macro(CALL) = _SPECIALIZE_CALL + unused/2 + _CALL;
+ macro(TAIL_CALL) =  _SPECIALIZE_CALL + unused/2 + _CALL;
```

Next, run `make regen-cases` to translate the `Python/bytecodes.c` to a propper c code. Let's have a look what is changed.

First, it defined a new bytecode `Include/opcode_ids.h`,  `Include/opcode_targets.h` and `Lib/_opcode_metadata.py`

```diff
+#define TAIL_CALL                              116
```

Second, it defined how to interpret the new bytecode in the `Python/generated_cases.c.h`. This file contains a whole python interptetiner. 

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
The other important thing is to update imporlib after the new bytecode is introduced. 

Change `MAGIC_NUMBER` constant in the `Lib/importlib/_bootstrap_external.py`. This will lead to .pyc files with the old `MAGIC_NUMBER` to be recompiled by the interpreter on import. 

Then, run `make regen-importlib`. 

<!-- todo importlib  -->


## Flowgraph optimization

According to the
[CPython devguide](https://devguide.python.org/internals/compiler/#control-flow-graphs)  control flow graph is an intermediate result of python source code compilation. CFGs are usually one step away from final code output, and are perfect place to perfom code optimization. 

Look to the to the `Python/flowgraph.c/_PyCfg_OptimizeCodeUnit`. The function updates code graph: removes unused condsts, insterts super instructions, etc.

```c
int
_PyCfg_OptimizeCodeUnit(cfg_builder *g, PyObject *consts, PyObject *const_cache,
                        int nlocals, int nparams, int firstlineno)
{
 lock));
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

Let's instert a new optimization `optimize_tail_call`.

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
            if (inst->i_opcode == CALL && nextop == RETURN_VALUE)
            INSTR_SET_OP1(inst, TAIL_CALL, inst->i_oparg);
  
        }
    }
}
```

Let's check if a new optimizaiton is working.

Recompile cpython with `make -j6`

And check the new optimization with `dis` module:

```python
>>> def f():
....    return g():

>>> dis.dis(f)

1           RESUME                   0

2           LOAD_GLOBAL              1 (g + NULL)
              TAIL_CALL                0
              RETURN_VALUE
```

Perfect. Let's move to the next step. 