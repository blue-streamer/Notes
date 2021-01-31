### 参数传递

*args和**kwargs的用法，简单来说\*args用来传递可变长度的list，\*\*kwargs用来传递可变长度的dict。举例如下：

*args

```python
def test_args(normal_arg, *args):
    print("first normal arg:" + normal_arg)
    for arg in args:
        print("another arg through *args :" + arg)

test_args("normal", "python", "java", "C#")
```

执行结果如下：

```
first normal arg: normal
another arg through *args : python
another arg through *args : java
another arg through *args :C#
```

**kwargs

```python
def test_kwargs(**kwargs):
    if kwargs is not None:
        for key, value in kwargs.iteritems():
            print("{} = {}".format(key,value))
        # Or you can visit kwargs like a dict() object
        # for key in kwargs:
        #    print("{} = {}".format(key, kwargs[key]))
test_kwargs(name="python", value="5")
```

执行结果如下：

```
name = python
value = 5
```

