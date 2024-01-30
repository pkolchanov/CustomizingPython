## Introdution

The goal is to add an almostequal `assert(1 ?= 1.1)` operator to cpython. This tutorial will show how the Python internals works.  

Disclamer: I'm python internals newbie, and writing to teach myself.

## Preparations
Clone the cpython python repo and checkout to a new branch.
```bash
git clone git@github.com:python/cpython.git
```

```bash 
cd cpython 
git checkout tags/v3.12.1 -b v3.12.1
```

Next, compile python using [devguide instructions](https://devguide.python.org/#quick-reference). 


## Plan
In short, to process code cpython works in three phases:
1. Parsing from code string to AST
2. Compilatoin from AST to bytecode operaons
3. Running python vm on bytecode ops

So, wee need to modify every phase.


## Parsing from code string to AST

#### Tokenezation 
Let's start with tokenezation. Tokenization is a process of splitting input string to sequence of tokens. 

Add a new `ALMOSTEQUAL` token to the  `Grammar/Tokens`
```diff 
 ELLIPSIS                '...'
 COLONEQUAL              ':='
 EXCLAMATION             '!'
+ALMOSTEQUAL             '?='
 
 OP
 TYPE_IGNORE
```

And run `make regen-token`. This will regenerate `pycore_token.h`, `Parser/token.c`, `Lib/token.py`. For example, look to the changes of the `token.c`. 

```diff 
         case '=': return GREATEREQUAL;
         case '>': return RIGHTSHIFT;
         }
+        break;
+    case '?':
+        switch (c2) {
+        case '=': return ALMOSTEQUAL;
+        }
         break;
     case '@':
         switch (c2) {

```

#### ASDL 

> Todo

Now let's move to `Parser/Python.asdl`. This file contains ???. Add `Aeq` to `cmpop` operation list. This will help us later. 


```diff
     unaryop = Invert | Not | UAdd | USub
 
-    cmpop = Eq | NotEq | Lt | LtE | Gt | GtE | Is | IsNot | In | NotIn
+    cmpop = Eq | NotEq | Lt | LtE | Gt | GtE | Is | IsNot | In | NotIn | Aeq
 
     comprehension = (expr target, expr iter, expr* ifs, int is_async)
 

```
Then run `make regen-ast` to regenerate `pycore_ast.h` and `Python-ast.c`.


#### Grammar
Now let's move to `Parser/python.gram`. This file contains the python grammar. In short: it describes how to construct an abstract syntaxt tree using the grammar rules. 

> Todo

Add new `aeq_bitwise_or` rule.
It says "when math a `bitwise_or` after  `'?=`' construct a AST node using `_PyPegen_cmpop_expr_pair` function with `Aeq` param"

```diff 

     | in_bitwise_or
     | isnot_bitwise_or
     | is_bitwise_or
+    | aeq_bitwise_or
```


```diff 
 in_bitwise_or[CmpopExprPair*]: 'in' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, In, a) }
 isnot_bitwise_or[CmpopExprPair*]: 'is' 'not' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, IsNot, a) }
 is_bitwise_or[CmpopExprPair*]: 'is' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, Is, a) }
+aeq_bitwise_or[CmpopExprPair*]: '?=' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, Aeq, a) }
 
 # Bitwise operators
 # -----------------

```

Then run `make regen-pegen` to regenerate  `Parser/parser.c`. Let's see how parser is chanegd. Added a new condition defined in grammar. 

``` diff
+        if (
+            (_literal = _PyPegen_expect_token(p, 55))  // token='?='
+            &&
+            (a = bitwise_or_rule(p))  // bitwise_or
+        )
+        {
+            _res = _PyPegen_cmpop_expr_pair ( p , Aeq , a );
+            goto done;
+ }

```

Thats all. Recompile cypthon with `make -j4` and test the new operator. 

``` python
>>> 1 ?= 2

Fatal Python error: compiler_addcompare: We've reached an unreachable state. Anything is possible.
```

Parsing is working, so let's move to the compolation step.


## Compilatoin from AST to bytecode operations
The next stage is comilation. Now need to translate   an AST to sequence of commands for Python VM.

Let's look a the `Python/compile.c/compiler_addcompare`. The function translates a `cmpop_expr_pair` AST node to a `COMPARE_OP` operation  with a simple switch:

```c 

static int compiler_addcompare(struct compiler *c, location loc,
                               cmpop_ty op)
{
    int cmp;
    switch (op) {
    case Eq:
        cmp = Py_EQ;
        break;
    case NotEq:
        cmp = Py_NE;
        break
    ...
    
    ADDOP_I(c, loc, COMPARE_OP, (cmp << 5) | compare_masks[cmp]);
   
```

So we need to add a new paramter to `COMPARE_OP` command. 


Let's jump to compare operation parameters. Move to `Include/object.h` and define a new `Py_AE` constant.

```diff 
 #define Py_NE 3
 #define Py_GT 4
 #define Py_GE 5
+#define Py_AE 6
```


Then edit the `Py_RETURN_RICHCOMPARE` macros. The macros helps to implement fast compartion among varios  python objects: 
```diff
#define Py_RETURN_RICHCOMPARE(val1, val2, op)                          
    do {                                                                                                       
         switch (op) {                                                      
         case Py_EQ: if ((val1) == (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;  
+        case Py_AE: if ((val1) == (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;  
         case Py_NE: if ((val1) != (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;  
         case Py_LT: if ((val1) < (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;   
         case Py_GT: if ((val1) > (val2)) Py_RETURN_TRUE; Py_RETURN_FALSE;   
```


Now we can use new `PY_AE` parameter in the `Python/compile.c/compiler_addcompare`

```diff 
     case GtE:
         cmp = Py_GE;
         break;
+    case Aeq:
+        cmp = Py_AE;
+        break;
     case Is:
         ADDOP_I(c, loc, IS_OP, 0);
         return SUCCESS;

```

Recompile cpython with `make -j4` and try new op

``` python
>>> 1 ?= 2

Assertion failed: ((oparg >> 5) <= Py_GE), function _PyEval_EvalFrameDefault, file generated_cases.c.h, line 2005.
```

Additionaly, we need to fix some other places. Patch an compare asserion in the `Python/bytecodes.c`

```diff
         }
 
         op(_COMPARE_OP, (left, right -- res)) {
-            assert((oparg >> 5) <= Py_GE);
+            assert((oparg >> 5) <= Py_AE);
             res = PyObject_RichCompare(left, right, oparg >> 5);
             DECREF_INPUTS();
             ERROR_IF(res == NULL, error);

````

Next, run `make regen-cases` to update asserions.

And fix the `Objects/object.c`.

```diff
 {
     PyThreadState *tstate = _PyThreadState_GET();
 
-    assert(Py_LT <= op && op <= Py_GE);
+    assert(Py_LT <= op && op <= Py_AE);
     if (v == NULL || w == NULL) {
         if (!_PyErr_Occurred(tstate)) {
             PyErr_BadInternalCall();

```


Recompile cpython with `make -j4` and try the new operator.


``` python
>>> 1 ?= 2

False
```

Yaay! Let's move to the final part.

## Running python vm on bytecode ops

The last goal is to modify python virtual machine. We need to define how to use  the new `Py_AE` parameter of `COMPARE_OP` command. 

In educational purposes let's change behaviour of float ojbects. The `Objects/floatobject.c/float_richcompare` function defines how float objects is compared. It's kind of tricky, and we are intersted in a last compare switch: 

```c 

static PyObject*
float_richcompare(PyObject *v, PyObject *w, int op) {
...

 Compare:
    switch (op) {
    case Py_EQ:
        r = i == j;
        break;
    case Py_NE:
        r = i != j;
        break;
    case Py_LE:
        r = i <= j;
        break;
...

```

Lets add a new case for almost equal comparion:

```diff
     case Py_GT:
         r = i > j;
         break;
+    case Py_AE:
+        r = fabs(i - j) < 1;
+        break;
     }
     return PyBool_FromLong(r);

```

We also add a new richcompare operation to the `Objects/object.c`:

```diff

 /* Map rich comparison operators to their swapped version, e.g. LT <--> GT */
-int _Py_SwappedOp[] = {Py_GT, Py_GE, Py_EQ, Py_NE, Py_LT, Py_LE};
+int _Py_SwappedOp[] = {Py_GT, Py_GE, Py_EQ, Py_NE, Py_LT, Py_LE, Py_AE};
 
-static const char * const opstrings[] = {"<", "<=", "==", "!=", ">", ">="};
+static const char * const opstrings[] = {"<", "<=", "==", "!=", ">", ">=", "?="};
 
```

Thats all!
Recompile cpython with `make -j4` and try to use the shiny almostequal operatior. 

``` python
>>> 1 ?= 1.1

True
```
