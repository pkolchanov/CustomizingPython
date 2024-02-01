## Introdution

The goal is to add an `almostequal` `assert(1 ?= 1.1)` operator to CPython. This tutorial is inspired by the [Anthony Shaw’s CPython Internals](https://realpython.com/products/cpython-internals-book/), but covers newer Python version. 

Disclaimer: I'm a Python internals newbie, and I'm writing this to teach myself.

## Preparations
Clone the CPython Python repo and checkout to a new branch.
```bash
git clone git@github.com:python/cpython.git
```

```bash 
cd cpython 
git checkout tags/v3.12.1 -b v3.12.1
```

Next, compile Python using [devguide instructions](https://devguide.python.org/#quick-reference). 


## Plan
In short, Cpython works in three phases:
1. Parsing source code string to AST
2. Compilation from AST to bytecode operations
3. Running Python VM on bytecode ops

So, to implement almostequal op, we need to modify every phase.


## Parsing source code string to AST

#### Tokenezation 
Let's start with tokenization. Tokenization is a process of splitting an input string into a sequence of tokens.

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

Now let's move to `Parser/Python.asdl`. Python AST nodes are defined by the ASDL. The ASDL definitions used to genrate the C structure type.

Let's add  `Aeq` to `cmpop` operation list. This will help us later. 

```diff
     unaryop = Invert | Not | UAdd | USub
 
-    cmpop = Eq | NotEq | Lt | LtE | Gt | GtE | Is | IsNot | In | NotIn
+    cmpop = Eq | NotEq | Lt | LtE | Gt | GtE | Is | IsNot | In | NotIn | Aeq
 
     comprehension = (expr target, expr iter, expr* ifs, int is_async)
 

```
Then run `make regen-ast` to regenerate `pycore_ast.h` and `Python-ast.c`. 


#### Grammar
Now let's move to `Parser/python.gram`. This file contains the python grammar.  In short: it describes how to construct an abstract syntax tree using the grammar rules.

Add a new `aeq_bitwise_or` rule.


```diff
compare_op_bitwise_or_pair[CmpopExprPair*]:
     | eq_bitwise_or
     | noteq_bitwise_or
     | lte_bitwise_or
     | lt_bitwise_or
     | gte_bitwise_or
     | gt_bitwise_or
     | notin_bitwise_or
     | in_bitwise_or
     | isnot_bitwise_or
     | is_bitwise_or
+    | aeq_bitwise_or
```

And define it. "When matching a `bitwise_or` after `'?=`' token, construct a AST node using `_PyPegen_cmpop_expr_pair` function with `Aeq` parameter"

```diff 
 in_bitwise_or[CmpopExprPair*]: 'in' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, In, a) }
 isnot_bitwise_or[CmpopExprPair*]: 'is' 'not' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, IsNot, a) }
 is_bitwise_or[CmpopExprPair*]: 'is' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, Is, a) }
+aeq_bitwise_or[CmpopExprPair*]: '?=' a=bitwise_or { _PyPegen_cmpop_expr_pair(p, Aeq, a) }

```

Then run `make regen-pegen` to regenerate  `Parser/parser.c`. Let's see how the parser is changed. 

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
>>> 1 ?= 1.1

Fatal Python error: compiler_addcompare: We've reached an unreachable state. Anything is possible.
```

We can also check a new `Aeq` paramenter with `ast.parse`:

```python
>>> import ast
>>> tree = ast.parse('1 ?= 1.1')
>>> ast.dump(tree)
'Module(body=[Expr(value=Compare(left=Constant(value=1), ops=[Aeq()], comparators=[Constant(value=1.1)]))], type_ignores=[[])'
```

Parsing is working, so let's move to the compilation step.


## Compilatoin from AST to bytecode operations
The next stage is compilation. Now we need to translate an AST to a sequence of commands for the Python VM.

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


Then edit the `Py_RETURN_RICHCOMPARE` macros.The macros help to implement fast comparison among various Python objects:
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


Now we can use the new `PY_AE` parameter in the `Python/compile.c/compiler_addcompare`

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
>>> 1 ?= 1.1

Assertion failed: ((oparg >> 5) <= Py_GE), function _PyEval_EvalFrameDefault, file generated_cases.c.h, line 2005.
```

Additionally, we need to fix some other places. Patch a compare assertion in the  `Python/bytecodes.c`

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
>>> 1 ?= 1.1

False
```

Yaay! Let's move to the final part.

## Running python vm on bytecode ops

The last goal is to modify python virtual machine. We need to define how to use  the new `Py_AE` parameter of `COMPARE_OP` command. 

In educational purposes let's change behaviour of float ojbects. The `Objects/floatobject.c/float_richcompare` function defines how float objects is compared. It's kind of tricky, and we are intersted in the last compare switch: 

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

Let's add a new case for almost equal comparison:

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
