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