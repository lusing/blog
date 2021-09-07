# 符号执行(2) - 用例搜索时间的控制与优化

通过上一节，大家对klee符号执行的概念已经有了比较清晰的了解了。
但是，一个工具要用起来，面对的是复杂的场景，还有很多困难等着我们去克服。

## 生成用例可能是个很耗时的操作

第一个要强调的问题是针对复杂情况，我们对于搜索case的时间要有个明确的概念。

以上节的求最大公约数为例。当我们计算short类型的最大公约数时，用时3分32秒，生成测试用例44个。

```c
#include "klee/klee.h"

short gcd(short a, short b){
    short a0 = a;
    short b0 = b;
    short c0 = 0;
    if(a<=0 || b<=0){
        return 0;
    }

    if(a<b){
        a0 = b;
        b0 = a;
    }

    for(;;){
        c0 = a0 % b0;
        if(c0==0){
            return b0;
        }else{
            a0 = b0;
            b0 = c0;
        }
    }

    return 1;
}

int main()
{
    short a,b;
    klee_make_symbolic(&a, sizeof(a), "a");
    klee_make_symbolic(&b, sizeof(b), "b");
    return (gcd(a,b));
}
```

而将类型改成int时，用时就达到5小时43分之多，生成测试用例增加到90个。

```c
#include "klee/klee.h"

int gcd(int a, int b){
    int a0 = a;
    int b0 = b;
    int c0 = 0;
    if(a<=0 || b<=0){
        return 0;
    }
    
    if(a<b){
        a0 = b;
        b0 = a;
    }
    
    for(;;){
        c0 = a0 % b0;
        if(c0==0){
            return b0;
        }else{
            a0 = b0;
            b0 = c0;
        }
    }
    
    return 1;
}

int main()
{
    int a,b;
    klee_make_symbolic(&a, sizeof(a), "a");
    klee_make_symbolic(&b, sizeof(b), "b");
    return (gcd(a,b));
}
```

我们可以通过查看klee-out-*的info文件来查看运行相关的信息。

short型最大公约数的运行信息如下：
```
[root@7a5293f64325 klee-out-2]# cat info
klee gcd2.bc
PID: 164
Using monotonic steady clock with 1/1000000000s resolution
Started: 2020-10-14 08:36:23
BEGIN searcher description
<InterleavedSearcher> containing 2 searchers:
RandomPathSearcher
WeightedRandomSearcher::CoveringNew
</InterleavedSearcher>
END searcher description
Finished: 2020-10-14 08:39:55
Elapsed: 00:03:32
KLEE: done: explored paths = 44
KLEE: done: avg. constructs per query = 389
KLEE: done: total queries = 49
KLEE: done: valid queries = 4
KLEE: done: invalid queries = 45
KLEE: done: query cex = 49

KLEE: done: total instructions = 1431
KLEE: done: completed paths = 44
KLEE: done: generated tests = 44
```

从上面信息可以看到，运行时长00:03:32，执行1431条指令，生成测试用例44个。

而对比int型的执行情况：
```
[root@1d4645e4ca3f klee-out-8]# cat info
klee gcd.bc
PID: 23411
Using monotonic steady clock with 1/1000000000s resolution
Started: 2020-10-14 08:41:30
BEGIN searcher description
<InterleavedSearcher> containing 2 searchers:
RandomPathSearcher
WeightedRandomSearcher::CoveringNew
</InterleavedSearcher>
END searcher description
Finished: 2020-10-14 14:24:30
Elapsed: 05:43:00
KLEE: done: explored paths = 90
KLEE: done: avg. constructs per query = 913
KLEE: done: total queries = 95
KLEE: done: valid queries = 4
KLEE: done: invalid queries = 91
KLEE: done: query cex = 95

KLEE: done: total instructions = 2503
KLEE: done: completed paths = 90
KLEE: done: generated tests = 90
```

我们看到，指令数从1431增加到2503，生成的用例数从44增加到90，但执行时间从3分钟32秒增加到5小时43分。

不过，虽然只增长了一倍的测试用例，但是生成1836311903和1134903170这样的用例也是值得的。

## 指定超时时间

为了避免无限执行下去而导致运行时长不可控，我们可以用--max-time参数给klee来设置超时时间。

比如给前面的最大公约数函数限定执行10分钟：
```
klee --max-time=600s gcd.bc
```

从运行情况可以看到，执行了10分52秒，找到50个用例。虽然还有40个用例没生成，但比起short类型的44个，还是增加了6个超出short类型之外的。

```
klee --max-time=600s gcd.bc
PID: 18
Using monotonic steady clock with 1/1000000000s resolution
Started: 2020-10-15 10:24:23
BEGIN searcher description
<InterleavedSearcher> containing 2 searchers:
RandomPathSearcher
WeightedRandomSearcher::CoveringNew
</InterleavedSearcher>
END searcher description
Finished: 2020-10-15 10:35:15
Elapsed: 00:10:52
KLEE: done: explored paths = 50
KLEE: done: avg. constructs per query = 408
KLEE: done: total queries = 53
KLEE: done: valid queries = 2
KLEE: done: invalid queries = 51
KLEE: done: query cex = 53

KLEE: done: total instructions = 1330
KLEE: done: completed paths = 50
KLEE: done: generated tests = 50
```

增加超时之后，我们就可以在常规的测试中保证比如一晚上时间可以跑完。需要更长时间的可以在专门的环境中继续长期运行。

## 搜索时间优化初步

### 尝试不同的SMT求解器

除了默认的stp求解器之外，我们还可以通过--solver-backend来使用z3或者metasmt来进行求解。

比如我们用效率也许更高的z3来做后台的求解器，我们还是跑10分钟：
```
klee --max-time=600s --solver-backend=z3 gcd.bc
```

从下面结果可以发现，z3为求解器的情况下，用时10分9秒时间，可以发现53个用例，比stp多三个用例：

```
[root@433437c6ceb7 libs]# cat klee-out-12/info
klee --max-time=600s --solver-backend=z3 gcd.bc
PID: 154
Using monotonic steady clock with 1/1000000000s resolution
Started: 2020-10-15 11:17:54
BEGIN searcher description
<InterleavedSearcher> containing 2 searchers:
RandomPathSearcher
WeightedRandomSearcher::CoveringNew
</InterleavedSearcher>
END searcher description
Finished: 2020-10-15 11:28:03
Elapsed: 00:10:09
KLEE: done: explored paths = 53
KLEE: done: avg. constructs per query = 86
KLEE: done: total queries = 56
KLEE: done: valid queries = 2
KLEE: done: invalid queries = 54
KLEE: done: query cex = 56

KLEE: done: total instructions = 1421
KLEE: done: completed paths = 53
KLEE: done: generated tests = 53
```

### 尝试不同的搜索策略

通过--search参数可以指定搜索策略。
可以取的值有：
- dfs: 深度优先搜索
- bfs: 广度优先搜索
- random-state: 随机状态 
- random-path: 随机路径选择
- nurs:covnew 非均匀随机搜索和新覆盖
- nurs:md2u 最小未覆盖距离的非均匀随机搜索
- nurs:depth 深度非均匀随机搜索
- nurs:rp 深度为1/2的非均匀随机搜索
- nurs:icnt 指令数的非均匀随机搜索
- nurs:cpicnt 调用路径指令数的非均匀随机搜索
- nurs:qc 基于查询代价的非均匀随机搜索

默认是随机路径选择与新覆盖的非均匀随机搜索的交织。

比如我们将策略换成最小未覆盖距离的非均匀随机搜索，命令如下：
```
klee --max-time=600s --search=nurs:md2u gcd.bc
```

我们可以看到，nurs:md2u策略用时10分19秒，找到42个用例，效果不如默认策略。

```
klee --max-time=600s --search=nurs:md2u gcd.bc
PID: 203
Using monotonic steady clock with 1/1000000000s resolution
Started: 2020-10-15 12:22:01
BEGIN searcher description
WeightedRandomSearcher::MinDistToUncovered
END searcher description
Finished: 2020-10-15 12:32:20
Elapsed: 00:10:19
KLEE: done: explored paths = 42
KLEE: done: avg. constructs per query = 394
KLEE: done: total queries = 45
KLEE: done: valid queries = 2
KLEE: done: invalid queries = 43
KLEE: done: query cex = 45

KLEE: done: total instructions = 1051
KLEE: done: completed paths = 42
KLEE: done: generated tests = 42
```

### 交织组合新策略

不过别担心，我们还可以对策略进行交织，加上随机搜索，搞成探索和规则一起发力，比如我们用nurs:md2u搭配random-state：
```
klee --max-time=600s --search=random-state --search=nurs:md2u gcd.bc
```

同样跑10分钟，这次我们找到了51个用例：
```
klee --max-time=600s --search=random-state --search=nurs:md2u gcd.bc
PID: 15
Using monotonic steady clock with 1/1000000000s resolution
Started: 2020-10-16 09:57:39
BEGIN searcher description
<InterleavedSearcher> containing 2 searchers:
RandomSearcher
WeightedRandomSearcher::MinDistToUncovered
</InterleavedSearcher>
END searcher description
Finished: 2020-10-16 10:08:28
Elapsed: 00:10:49
KLEE: done: explored paths = 51
KLEE: done: avg. constructs per query = 424
KLEE: done: total queries = 54
KLEE: done: valid queries = 2
KLEE: done: invalid queries = 52
KLEE: done: query cex = 54

KLEE: done: total instructions = 1352
KLEE: done: completed paths = 51
KLEE: done: generated tests = 51
```

看起来我们组合出来的也不比默认的差。
