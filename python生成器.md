# 生成器——generator

## 简介

1. 语法类似函数，`yield`返回值而非`return`返回值
2. 实现**迭代器协议**：便于迭代，使用`next`返回值。注意：没有值返回时`StopIteration`异常
3. 状态挂起：`yield`返回值，此时状态挂起，便于恢复执行
4. **注意**：生成器只能遍历一遍，无法多次遍历

## 使用方式

### 生成器函数

例子如下：
```python
def gen_squares(N):
    for i in range(N):
        yield i ** 2
for i in gen_squares(5):
    print(i)
```

### 生成器表达式

例子如下：
```python
# squres = [x**2 for x in range(5)]   # 列表产生式
squres = (x**2 for x in range(5))   # 生成器表达式
print(squres)
print(next(squres))
print(next(squres))
print(list(squres))
```

## 为什么要使用生成器？

1. 延迟计算——便于大数据处理

```python
print(sum([i for i in range(10000000000)]))     # 内存溢出
print(sum(i for i in range(10000000000)))       # 内存几乎没有消耗
```

2. 减少代码量
