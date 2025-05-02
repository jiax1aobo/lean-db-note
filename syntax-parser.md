# 移进规约冲突（Shift-Reduce Conflict）

在某个分析状态中，语法分析器无法决定是应该将下一个输入符号**移进（Shift）**到栈中，还是将当前栈顶的符号序列**规约（Reduce）**为某个非终结符。

以算术表达式文法为例：

```yacc
E:
  E + E   {}
  | E * E {}
  | id    {}
  ;
```

上面的文法解析 `id + id * id` 这个输入，在输入 `*` 前，栈顶为 `E + E` ，此时分析器可以采取的行动有两种：

1. 移进（Shift）：将 `*` 移入栈，即最终使得栈顶为 `E + E * E` 然后从栈顶一次规约（先计算乘法）
2. 规约（Reduce）：现将栈顶规约为 `E` （先计算加法）

再以 `IF-ELSE` 文法为例：

```yacc
S :
  if E then S {}
  | if E then S else S {}
  | others {}
  ;
```

上面的文法解析 `if E then if E then S else S` 这个输入，在输入 `else` 前，栈顶状态为 `if E then if E then S` ，这时分析器可以采取的行动有：

1. 移进（Shift）：将 `else` 移入栈，将之与最近的 `if` 进行匹配，即属于内层。
2. 规约（Reduce）：将已经匹配的栈顶 `if E then S` 规约为 `S` ，即将栈顶规约为 `if E then S` 再移进 `else` ，即属于外层。

==移进规约冲突的原因是当前栈顶可规约的token与下个token的优先级不确定。==

# 规约规约冲突（Reduce-Reduce Conflict）

在某个分析状态中，语法分析器发现当前栈顶符号序列可以按照多个不同的产生式进行规约，导致无法去顶选择哪个产生式。

以变量声明和函数声明的文法为例：

```yacc
DECL:
  id {}
  ;
CALL:
  id {}
  ;
EXPR:
  DECL {}
  | CALL {}
  ;
```

当分析到 `id` 时，既可以规约为 `DECL` 也可以规约为 `CALL` ，此时 `id` 具有二义性。

可以通过引入额外符号或上下文消除二义性：

```yacc
DECL:
  id {}
  ;
CALL:
  id ( ARGS ) {}
  ;
EXPR:
  DECL {}
  | CALL {}
  ;
```

