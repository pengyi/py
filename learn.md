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
一个工厂函数，用来构建带有字段名的元组合一个有名字的类
Card = collection.nametuple('Card', ['rank', 'suit']) 
card = Card('A', 'heart')
Card._fields -> ('rank', 'suit')
card = Card._make(iterable) <=> Card(*iterable)

card._asdict() -> OrderedDict([('rank', 'A), ('suit', 'heart')])
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

#元组拆包
t = (va, vb)
- 平行赋值a, b = t
- 迭代拆包 for a, b in tuples:
- * 可以把一个可迭代对象拆开作为函数参数
  def func(a, b):  可以通过func(*t)调用

# os.path.split
path, filename = os.path.split('/path/to/file.ext')
path -> /path/to
filename -> file.ext

# 列表运算
- s.count(e) e在s中查询的次数
- s.index(e) 在s中找到e第一次出现的位置
- s.insert(p, e) 在位置p之前插入e
- s.extend(it) 把可迭代对象it追加给s
- s.remove(e) 删除s中第一次出现的e
- s.reverse() 就地把s的元素倒序排列
- s.sort([key], [reverse]) 对s就地进行排序

# 切片
切片不包括最后一个元素
- range(3), my_list[:3] 返回3个元素， 可以快速看见元素的个数
- 起止都有时，有stop-start个元素
- 便于分割列表 list[:x] list[x:]可以方便的切分为两部分
  
s[a:b:c] 在a和b之间以间隔c取值， c为负的话意味着反向取值
s = 'bicycle'
s[::-2] -> eccb

seq[start:stop:step] Python会调用seq.__getitem__(slice(start, stop, step))
P29

... <-> Ellipsis对象 省略号python解释器看来是一个特殊的符号。


# sorted
会新建一个列表作为返回值。可接受任何可迭代对象作为参数。

# bisect、insort
bisect二分查找
bisect(haystack, needle)返回值为needle可以插入后保持有序的位置
bisect <-> bisect_right 插入位置在相同元素的后面
bisect_left
```
def grade(score, breakpoints=[60, 70, 80, 90], grades='FDCBA'):
    i = bisect.bisect(breakpoints, score)
    return grades[i]
```
insort(seq, item) 插入元素保持有序
# collection.deque
双向队列是一个线程安全的包，可以快速从两点添加和删除元素。
dq = deque(range(10), maxlen=10)
- dq.popleft()
- dq.appendleft(e)
- dq.append(e)
- dq.pop()
- dq.rotate(n)将右边的n个元素放在左边
# heapq
堆队列 heappush heappop

# array.array
array(typecode, iterable)
- typecode b:有符号字符  d:浮点数 B:无符号字符 h：短整数
- fromfile(fp, n)
- tofile(fp)

# memoryview 
 内置类，在不复制内容的情况下操作同一个数组的不同切片
 numbers = array.array('h', [-2, -1, 0, 1, 2])
 memv = memoryview(numbers)
 memv_oct = memv.cast('B)
 memv.tolist() -> [254, 255, 255, 255, 0, 0, 1, 0, 2, 0]

# 字符串
字符串是字符的一个序列，字符的定义是unicode字符
- python3中str对象获取的元素就是unicode字符
- python2中从uniocde对象获取的对象是unicode字符
字符的码位：A U+0041 人类可读
字符的编码： UTF8 A -> \x41 机器可读
编码过程： 码位  -> 字节序列
解码过程：字节序列-> 码位

# 二进制序列 bytes和bytearray
python3新加了bytes类型
python2.6增加了bytearray类型
- bytes和bytearray的各个元素介于0~255之间的整数
- python2 str对象的字符是单个字符

cafe = bytes('string', encoding='utf-8')
bytes对象可以从str对象使用给定的编码构建

# time.perf_counter
python3.3+ 性能和精度更高的库

# 缓冲协议对象

# pickle序列化模块

<<<<<<< HEAD

# 构造字典的方法
- a = dict(one=1, two=2, three=3)
- a = {'one': 1, 'two': 2, 'three': 3}
- a = dict(zip(['one', 'two', 'three'], [1, 2, 3]))
- a = dict([('one', 1), ('two', 2), ('three', 3)])
- a = dict({'one': 1, 'two': 2, 'three': 3})
- 字典推导 

# 字典方法
- d.fromkeys(it, [init]) 将迭代器it的值作为键， init作为初始值
- d.__iter__() 获得键的迭代器
-  d.popitem()随机返回一个键值对，并且从d中移除
-  d.upate(m, [**kwargs]) m可以是映射或者键值对迭代器
-  d.setdefault(key, [default]) 若字典里有值则返回该值，没有则设置值为default
  
# collections.defaultdict
collections.defaultdict(default_factory)
default_factory会在__getitem__里找不到建的时候调用而d.get()不会调用

# inspect
```
from inspect import signature
sig = signature(func)
for name, param in sig.parameters.items():
   print(param.kind, ':', 'name', '=', param.default)
```

Parameter
 - name: 参数名
 - kind: 
    - VAR_POSITIONAL: 通过一个*前缀来声明，如果你看到一个*xxx的函数参数声明（不是函数调用！声明和调用是两种不同的含义的），那一定是属于        VAR_POSITIONAL类型的，如同语义，这种类型的参数只能通过位置POSITIONAL传参调用，不支持关键字KEYWORD传参，在函数内部，VAR_POSITIONAL类型的参数以一个元祖(tuple)显示
    - POSITIONAL_OR_KEYWORD: 如果没有任何*的声明，那么就是POSITIONAL_OR_KEYWORD类型的，如同语义一样，POSITIONAL_OR_KEYWORD类型的参数可以   通过位置POSITIONAL传参调用，也可以过关键字KEYWORD传参.

    - KEYWORD_ONLY: 第三种是关键字参数，这种参数只会在VAR_POSITIONAL类型参数的后面而且不带**前缀。如同语义，这类参数只能用关键字KEYWORD来传参，不可以用位置传参，因为位置传的参数全让前面的VAR_POSITIONAL类型参数接收完了，所以KEYWORD_ONLY只能通过关键字才能接收到参数值
    -  VAR_KEYWORD: 第四种是可变的关键字参数，VAR_KEYWORD类型的参数通过**前缀来声明（不是函数调用！声明和调用是两种不同的含义的）。如同语义，这种类型的参数只能通过关键字KEYWORD调用，但可以接收任意个关键字参数，甚至是0个参数，在函数内部以一个字典(dict)显示。VAR_KEYWORD类型的参数只允许有一个，只允许在函数的最后声名
    - POSITIONAL_ONL: 历史产物
```
import inspect

def foo(a, *b, c, **d):
    pass
for name, parame in inspect.signature(foo).parameters.items():
    print(name, ': ', parame.kind)

a :  POSITIONAL_OR_KEYWORD
b :  VAR_POSITIONAL
c :  KEYWORD_ONLY
d :  VAR_KEYWORD

```

 - default

