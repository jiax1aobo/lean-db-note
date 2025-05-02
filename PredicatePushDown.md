
> Predicate Push Down 谓词下推
> Filter Push Down 过滤器下推
> 这两个概念是等价的

**predicate**(谓词)即条件表达式,在SQL中,谓词就是返回true/false的函数,或是隐式转换为true/false的函数.

SQL中的谓词主要有:
```SQL
LIKE
BETWEEN
IS NULL
IS NOT NULL
IN
EXISTS
```

**predicate push down**即是将SQL语句中的部分语句(predicates谓词部分)可以被“**pushed**”下推到数据源或者靠近数据源的部分.通过尽早地过滤掉数据，这种处理方式能大大减少数据处理的量,降低资源消耗.


