# __class__
 对象所属类的引用，obj.__class__ == type(obj), Python的某些特殊
 方法，例如__getattr__只在对象的类中寻找，不在实例中寻找

# __dict__
一个映射，存储对象或者类的可写属性，有__dict__属性的对象，任何时候都可以
设置新属性。

# __slot__
类可以定义这个属性， 限制实例能有哪些属性。__slot__属性的值是一个字符串组成
的元组。

# dir(objct)
列出对象的*大多数*属性, 没有提供完整的属性列表

# vars([object])
返回object的__dict__属性，如果没有指定参数作用于
locals相同，返回本地作用域的字典

# __getattr__(self, name)
仅当获取指定属性失败，搜索过obj、class和超类后调用。
表达式obj.no_such_attr, get_attr(ob, 'no_such_attr'),
hasattr(obj, 'no_such_attr')可能会触发，Class.__getattr__(obj, 'no_such_attr')
方法， 但是，仅当在obj, Class和超类中找不到指定属性时才会触发。


# 属性描述符
描述符是实现了特定协议的类， 这个协议包括__get__, __set__和 __delete__方法。
描述符是类属性。
class Overriding:
    """数据描述符
    """
    def __get__(self, instance, owner):
        print_args('get', self, instance, owner)
    def __set__(self, instance, value):
        print_args('set', self, instance, value)

class OverridingNoGet:
    def __get__(self, instance, owner):
        print_args('get', self, instance, owner)
    def __set__(self, instance, value):
        print_args('set', self, instance, value)

class NonOverriding:
        """非数据描述符
        “”“
        def __get__(self, instance, owner):
        print_args('get', self, instance, owner)

class Managed:
    over = Overriding()
    over_no_get = OverridingNoGet()
    non_over = NonOverriding()

obj = Managed()

- 覆盖型描述符：实现__set__方法的描述符，覆盖对实例属性的赋值操作。property类
  也是覆盖型描述符。描述符属性会覆盖实例属性的访问
  obj.over -> Overriding.__get__(over, obj, Managed)
  Managed.over -> Overriding.__get__(over, None, Managed)
  obj.over = 7 -> Overriding.__set__(over, obj, 7)

- 没有__get__方法的覆盖性描述符
  设值方法一直会使用描述符方法， 但是读取时候实例属性会覆盖描述符类属性

- 非覆盖性描述符
  没有实现__set__方法的描述符，如果设值了同名实例属性，描述符会被覆盖。
  obj.over -> Overriding.__get__(over, obj, Managed)
  obj.non_over = 7
  obj.non_over -> 7

- 描述符用法建议
  - 只读描述符必须有__set__方法，并且抛出AttributeError异常
  - 验证描述符可以只有__set__,将数据直接放入__dict__，读取的时候直接读取实例属性
  - 仅有__get__方法的描述符可以实现缓存
  - 非特殊方法可以被实例属性覆盖

```

class Test:
   @property
   def attr(self):
       return a

    @attr.setter
    def attr(self, value):
        pass

t = Test()
attr = property(attr)
t.attr -> property.__get__(attr, t, Test)

t.attr = 7
t.attr -> property.__set__(attr, t, value)


# 一个简单的实现
class property:
    def __init__(fget=None, fset=None, fdel=None, fdoc):
        self.fget = fget
        self.fset = fset
        self.fdel = fdel
        self.fdoc = foc


    def setter(self, fset):
        self.fset = fset

    # owner是托管类的引用
    def __get__(self, instance, owner):
        # 不是通过实例调用返回，返回描述符本身
        if instance is None:
            return self
        else:
            return instance.fget()

    def __set__(self, instance, value):
        if self.fset == None:
            raise AttributeError
        else
           instance.fset(value)
```

# collection.nametuple表示记录
# from random import choice 随机选取一个元素
   choice(obj_list)
# doctest 测试代码
# __contains__
如果一个集合类型没有实现__contains__方法， 那么in操作符
就会按照顺序做一次迭代搜索
# 特殊方法
- 特殊方法的存在是为了被Python解释器调用, eg. len(my_obj)
等价于my_obj.__len__(), 但内置类型直接抄近路，读PyVarObject
的ob_size。可以使内置类型和自有类型拥有相同的接口
- 特殊方法有时调用是隐式的，eg. for i in x:,背后用的是iter(x),
函数背后则是x.__iter__()方法。

# __str__和 __repr__
__str__会被str调用，print也会隐式调用，如果一个对象没有实现__str__
而又需要它的时候，解释器会用__repr__代替。%r, !r是__repr__对应的
格式字符串

# __bool__
bool(x)背后调用x.__bool__,如果不存在__bool__，会尝试调用x.__len__()

# 容器序列和扁平序列
- 容器序列：list tuple collenctions.deque存放不同类型的引用
- 扁平序列：str bytes bytearray memoryview array.array 只能容纳一种类型，
  扁平序列是一段连续的内存空间，支持字符、字节、数字基础类型。
- 可变序列 list bytearray array.array deque memoryview
- 不可变序列 tuple str bytes

# 换行忽略
python会忽略代码中的[] () {}中的换行
# 列表推导
lstcom = [item for item in items]
- 只用列表推导创建新列表，并尽量保持简短，
- python3列表推导有了自己的局部作用域

# 生成器表达式
gencomp = (item for item in items)
逐个产生元素，遵守迭代器协议，减少内存占用
