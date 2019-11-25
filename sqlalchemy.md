# pool
## 对象

weakref.ref(object[, callback]) 
创建一个弱引用对象，object是被引用的对象
callback是回调函数（当被引用对象被删除时的，会调用改函数）


- _ConnectionFairy
   DBAPI connection的透明代理，生命周期是从pool中checkout的时间
   ```
    @classmethod
    def _checkout(cls, pool, threadconns=None, fairy=None):
        if not fairy:
            fairy = _ConnectionRecord.checkout(pool)

            fairy._pool = pool
            fairy._counter = 0

            if threadconns is not None:
                threadconns.current = weakref.ref(fairy)

        if fairy.connection is None:
            raise exc.InvalidRequestError("This connection is closed")
        fairy._counter += 1

        if (not pool.dispatch.checkout and not pool._pre_ping) or \
                fairy._counter != 1:
            return fairy
    
    # 透明代理方法
    def __getattr__(self, key):
        return getattr(self.connection, key)
    ```

- _ConnectionRecord
   pool里存放的对象, 生命周期比DBAPI connection长

```
    @classmethod
    def checkout(cls, pool):
        rec = pool._do_get()
        try:
            # 返回原始connection
            dbapi_connection = rec.get_connection()
        except Exception as err:
            with util.safe_reraise():
                rec._checkin_failed(err)
        echo = pool._should_log_debug()
        fairy = _ConnectionFairy(dbapi_connection, rec, echo)
        rec.fairy_ref = weakref.ref(
            fairy,
            lambda ref: 
            _finalize_fairy and
            _finalize_fairy(
                None,
                rec, pool, ref, echo)
        )
        _refs.add(rec)
        if echo:
            pool.logger.debug("Connection %r checked out from pool",
                              dbapi_connection)
        return fairy
```

- Pool
```
    def connect(self):
        """Return a DBAPI connection from the pool.

        The connection is instrumented such that when its
        ``close()`` method is called, the connection will be returned to the pool.
        """
        if not self._use_threadlocal:
            return _ConnectionFairy._checkout(self)

        try:
            rec = self._threadconns.current()
        except AttributeError:
            pass
        else:
            if rec is not None:
                return rec._checkout_existing()

        return _ConnectionFairy._checkout(self, self._threadconns)
```


- QueuePool
pool_size决定了池维护多少链接，链接不会预先建立，只是在创建后返回连接池
max_overflow可以超过size的数量，连接池可以维护数量pool_size + max_overflow。
```
    def _do_get(self):
        use_overflow = self._max_overflow > -1

        try:
            wait = use_overflow and self._overflow >= self._max_overflow
            # 从连接池取链接
            return self._pool.get(wait, self._timeout)
        except sqla_queue.Empty:
            # don't do things inside of "except Empty", because when we say
            # we timed out or can't connect and raise, Python 3 tells
            # people the real error is queue.Empty which it isn't.
            pass
        if use_overflow and self._overflow >= self._max_overflow:
            if not wait:
                return self._do_get()
            else:
                raise exc.TimeoutError(
                    "QueuePool limit of size %d overflow %d reached, "
                    "connection timed out, timeout %d" %
                    (self.size(), self.overflow(), self._timeout), code="3o7r")

        if self._inc_overflow():
            try:
                # 获得_ConnectionRecord ********************************************88
                return self._create_connection()
            except:
                with util.safe_reraise():
                    self._dec_overflow()
        else:
            return self._do_get()

    def _do_return_conn(self, conn):
        try:
            # 放回连接池 **************************************
            self._pool.put(conn, False)
        except sqla_queue.Full:
            try:
                conn.close()
            finally:
                self._dec_overflow()
```


# 连接创建过程
pool.unique_connection
_ConnectionFairy._checkout
 _ConnectionRecord.checkout
 rec = pool._do_get()
 dbapi_connection = rec.get_connection()
 _ConnectionRecord.__connect()
 connection = pool._invoke_creator(self)
 dbapi.connect()



## 重连
- 发起ping
```
try:
    result = pool._dialect.do_ping(fairy.connection)
except:
    fairy.connection = fairy._connection_record.get_connection()
```

## base model
_extract_mappable_attributes -> self.properties
_extract_declared_columns 设置self.properties -> declared_columns
_setup_table # 创建Table对象 Table(tablename, declared_columns)
_early_mapping 将table和用户定义的类调用mapper函数做映射, 设置__mapper__属性 mapper(class_, local_table)
mapper流程
_prepare_mapper_arguments map
_configure_class_instrumentation
_configure_properties
_property_from_column => Column ----> ColumnProperty
ColumnProperty::instrument_class
attributes.register_descriptor
ClassManager::instrument_attribute (inst is descriptor)
install_descriptor
setattr(self.class_, key, descriptor_ins)



register_attribute_impl注册具体实现
默认ScalarAttributeImpl  sqlalchemy.orm.attributes.ScalarAttributeImpl
DynaLoader 


# ClassManager
tracks state information at the class level.
ClassManager和映射到的类相联系


# properties
_configure_properties

#InstrumentedAttribute
和映射到类上的每个属性相联系。InstrumentedAttribute还是前面提到的Python描述符，
在基于类的表达式（如User.id==5中使用时，产生SQL表达式，当处理User的一个实例时，
InstrumentedAttribute将属性的行为委托给一个AttributeImpl对象——为所表示的数据
定制的多个变体之一


# Mapper

Mapper维护了一组MapperProperty属性对象，每个属性对象处理一个特定属性的SQL表示。
MapperProperty最常见的变体是ColumnProperty（表示了一个映射到的字段或SQL表达式）
和RelationshipProperty（表示了到另一个映射器的关联）。

mapper.c <--> Mapper.columns


# 编译
default_comparator定义了操作符，操作符返回expression，
expression的父类ClauseElement的__str__方法调用compile()
compile方法触发dialect.statement_compiler(dialect, self, **kw)调用
statement_compiler具体实现statement_compiler = MySQLCompiler_mysqldb
MySQLCompiler_mysqldb的父类Compiled的__str__方法返回编译后的字符串表示

Compiled的子类提供了一系列的visit方法，每一个visit方法都被ClauseElement的一个特殊子类所引用。
通过对（语法树上的）ClauseElement节点进行遍历，递归地连接每个visit函数的字符串输出，就构建出了一个语句


# class instrumentation
ColumnProperty instrument_class
InstrumentedAttribute

#connect sring
dialect[+driver]://user:password@host/dbname[?key=value..]

# Connection
 The Connection object represents a single dbapi connection 
 checked out from the connection pool.
 由engine使用。
 维护__connection：FairyConncetion
 dialect = engine.dialect
 engine

# Engine
Connects a Pool and Dialect together to provide 
a source of database connectivity and behavior.

- pool
- dialect
- raw_connection()

# create_engine
- encoding
- pool
- poolclass
- pool_size
- pool_recycle:Note that MySQL in particular will disconnect automatically if no
    activity is detected on a connection for eight hours 
- strategy='plain' selects alternate engine implementations

# Session
 - bind: engine
 


# 连接建立过程
Connection.connection
## Return a "raw" DBAPI connection from the connection pool
## The returned object is a proxied version of the DBAPI connection object
engine.raw_connection
## Produce a DBAPI connection that is not referenced by any thread-local context
pool.unique_connection
_ConnectionFairy._checkout(pool) -> _ConnectionFairy:
fairy = _ConnectionRecord.checkout(pool)
DefaultDialect.connect  => creator
dbapi.connect



# Session
Manages persistence operations for ORM-mapped objects
 ```
 def execute():
    clause = expression._literal_as_text(clause)
    if bind is None:
        bind = self.get_bind(mapper, clause=clause, **kw)
    return self._connection_for_bind(
        bind, close_with_result=True).execute(clause, params or {})

def query(self, *entities, **kwargs):
    """Return a new :class:`.Query` object corresponding to this
    :class:`.Session`."""

    return self._query_cls(entities, self, **kwargs)
```

# Mapper
Define the correlation of class attributes to database tablecolumns, 
.mapper` is used explicitly to link a user defined class with table metadata



# declarative_base
BaseModel=declarative_base()
BaseModel的作用本身是作为一个桥梁链
接用户自定义table类和元类
用户自定义的table类是元类的实例，table类会使用元类创建，调用元类的__new__或者__init__


# ColumnClause
Represents a column expression from any textual string
sql/elements.py self.type = type_api.to_instance(type_)


# table = public_factory(TableClause, ".expression.table")

 user = table("user",
                column("id"),
                column("name"),
                column("description"),
        )
user.c.id

# 查找元类的顺序
1 Foo中有__metaclass__这个属性吗？如果是，Python会在内存中通过__metaclass__创建一个名字为Foo的类对象（我说的是类对象，请紧跟我的思路）。
2 如果Python没有找到__metaclass__，它会继续在Bar（父类）中寻找__metaclass__属性，并尝试做和前面同样的操作。
3 如果Python在任何父类中都找不到__metaclass__，它就会在模块层次中去寻找__metaclass__，并尝试做同样的操作。
4 如果还是找不到__metaclass__,Python就会用内置的type来创建这个类对象。

Base = declarative_base()

生成Base的元类为DeclarativeMeta, 可以理解为Base.__metaclass__ == DeclarativeMeta
因此继承自Base的所有子类都使用DeclarativeMeta元类创建类

class DeclarativeMeta(type):
    def __init__(cls, classname, bases, dict_):
        # _decl_class_registry在Base的子类中是没有的
        if "_decl_class_registry" not in cls.__dict__:
            _as_declarative(cls, classname, cls.__dict__)
        type.__init__(cls, classname, bases, dict_)

    def __setattr__(cls, key, value):
        _add_attribute(cls, key, value)

    def __delattr__(cls, key):
        _del_attribute(cls, key)




# 运算符重载

cls.id == 2
cls.id -> sqlalchemy.orm.attributes.InstrumentedAttribute
ColumnOperators.__eq__
QueryableAttribute.operate
ColumnProperty.Comparator.__eq__
ColumnProperty.Comparator.operate
self.__clause_element__() -> sqlalchemy.sql.annotation.AnnotatedColumn->Column
Column.__eq__ 
ColumnElement.operate
sqlalchemy.sql.sqltypes._LookupExpressionAdapter.Comparator.__eq__
sqlalchemy.sql.sqltypes._LookupExpressionAdapter.Comparator.operate
_boolean_compare()
BinaryExpression



# 编译sql语句
_compile_context
context = QueryContext(self)
context.statement = self._simple_statement(context)
statement = sql.select(context.primary_columns + context.secondary_columns,context.whereclause,) -> sqlalchemy.sql.selectable.Select

ClauseElement._execute_on_connection
base.Connection._execute_clauseelement
    compiled_sql = elem.compile(dialect=dialect)
    ClauseElement.compile()
    dialect.statement_compiler
    self.string = sqlalchemy.sql.schema.Column._compiler_dispatch
Connection._execute_context(dialect,dialect.execution_ctx_cls._init_compiled,compiled_sql,)
sqlalchemy.dialects.mysql.mysqldb.MySQLExecutionContext_mysqldb._init_compiled
MySQLCompiler_mysqldb.__str__
ialects.mysql.mysqldb.MySQLDialect_mysqldb.do_execute()

# flask-sqlalchemy
创建session对象的时候，创建engine对象

# scoped session
数据库设计的难点之一，是session生命周期的管理问题。sqlalchemy提供了一个简单的session管理机制，即scoped session。
它采用的注册模式。所谓的注册模式，简单来说，是指在整个程序运行的过程当中，只存在唯一的一个session对象。我们知道scoped session
本质上是一个全局变量。可是，如果直接把session定义成全局变量，在多线程的环境下，会造成线程同步的问题。为此，scoped session在默认情况下，
采用的线程本地化存储方式。也就是说，每个线程的session对象是不同的。这样，不同线程对数据库的操作不会相互影响。

sessionmaker定义了一个session的工厂对象，用于生产session对象

https://www.cnblogs.com/geeklove01/p/8179220.html
scoped_session类似单例模式，当我们调用使用的时候，会先在Registry里找找之前是否已经创建session了。
要是有，就把这个session返回。要是没有，就创建新的session，注册到Registry中以便下次返回给调用者。
scoped_session实现了代理模式。能够将操作转发到代理的对象中去执行,
scoped_session的实现使用了thread local storage技术，使session实现了线程隔离。这样我们就只能看见本线程的session

scope session的销毁
@app.teardown_appcontext
def shutdown_session(response_or_exc):
    if app.config['SQLALCHEMY_COMMIT_ON_TEARDOWN']:
        if response_or_exc is None:
            self.session.commit()
    self.session.remove()
    return response_or_exc


# load stratety
 有两种使用方式，一种是作为relationship的参数全局指定加载方式
 relationship("Child", lazy='joined')
 另一种作为query的方法options动态针对查询设置载人方式
  - lazy loading 
    azy='select' or the lazyload() option，首次访问属性时通过
    单独的select查询载入数据
  - joined loading
    急加载，lazy='joined' or the joinedload() option，这种方式
    使用join语句跟原查询一起得到结果

  - subquery loading
    急加载， lazy='subquery' or the subqueryload() option，
    原查询单独进行，会单独针对加载对象进行第二次查询，通过将第一个
    查询作为子查询，join加载对象，得到结果.
    这种方式第一次原样查询，不会因为强加join导致效率降低
    jack = session.query(User).options(subqueryload(User.addresses)).filter_by(name='jack').all()
    => 
    SELECT
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
    FROM users
    WHERE users.name = ?
    ('jack',)


    SELECT
        addresses.id AS addresses_id,
        addresses.email_address AS addresses_email_address,
        addresses.user_id AS addresses_user_id,
        anon_1.users_id AS anon_1_users_id
    FROM (
        SELECT users.id AS users_id
        FROM users
        WHERE users.name = ?) AS anon_1
    JOIN addresses ON anon_1.users_id = addresses.user_id
    ORDER BY anon_1.users_id, addresses.id
    ('jack',)

  - select IN loading
    急加载，lazy='selectin' or the selectinload() option
    单独使用主查询的主键，发起第二次select查询，将主键id作为IN
    的参数。
    selectin是最简单高效的加载对象的方式，不能用于复合主键
    selectin loading is the most simple and efficient way to eagerly load collections of objects in most cases
    jack = session.query(User).options(selectinload('addresses')).filter(or_(User.name == 'jack', User.name == 'ed')).all()
    SELECT
    users.id AS users_id,
    users.name AS users_name,
    users.fullname AS users_fullname,
    users.nickname AS users_nickname
    FROM users
    WHERE users.name = ? OR users.name = ?
    ('jack', 'ed')


    SELECT
        addresses.id AS addresses_id,
        addresses.email_address AS addresses_email_address,
        addresses.user_id AS addresses_user_id
    FROM addresses
    WHERE addresses.user_id IN (?, ?)
    ORDER BY addresses.user_id, addresses.id
    (5, 7)



# declarative
元类在生成类对象的时候调用，在定义类的时候调用
class DeclarativeMeta(type):
    def __init__(cls, classname, bases, dict_):
        # 有_decl_class_registry属性说明是定义Model的基类，
        # 没有则是用户自定义的model需要进行instrument
        if "_decl_class_registry" not in cls.__dict__:
            _as_declarative(cls, classname, cls.__dict__)
        type.__init__(cls, classname, bases, dict_)

declarative_base 生成一个model类共同的一个基类，
基类也可以通过指定cls设置一个它的基类。


def declarative_base(
    bind=None,
    metadata=None,
    mapper=None,
    cls=object,
    name="Base",
    constructor=_declarative_constructor,
    class_registry=None,
    metaclass=DeclarativeMeta,
):
    lcl_metadata = metadata or MetaData()
    if bind:
        lcl_metadata.bind = bind

    if class_registry is None:
        class_registry = weakref.WeakValueDictionary()

    bases = not isinstance(cls, tuple) and (cls,) or cls
    class_dict = dict(
        _decl_class_registry=class_registry, metadata=lcl_metadata
    )

    if isinstance(cls, type):
        class_dict["__doc__"] = cls.__doc__

    if constructor:
        class_dict["__init__"] = constructor
    if mapper:
        class_dict["__mapper_cls__"] = mapper

    return metaclass(name, bases, class_dict)
return metaclass(name, bases, class_dict)



model类对象有一个__mapper__属性用于存放Mapper对象，
_setup_table
cls.__table__ = table = table_cls(tablename, cls.metadata)
为model类设置__table__属性
Column._set_parent()




# sqlalchemy 连接管理
pool管理的是_ConnectionRecord，通过_ConnectionRecord创建

pool.unique_connection
 - _ConnectionFairy._checkout
   - _ConnectionRecord.checkout
    - rec = pool._do_get()
      # 检查链接有效
    - dbapi_connection = rec.get_connection()
    - fairy = _ConnectionFairy(dbapi_connection, rec, echo)
    - return fairy

## Proxies a DBAPI connection
_ConnectionFairy:
    + connection: A reference to the actual DBAPI connection being tracked
    + _connection_record: A reference to the :class:`._ConnectionRecord` object associated with the DBAPI connection
    + def __getattr__(self, key):
        """ proxy method """
        return getattr(self.connection, key)

## Internal object which maintains an individual DBAPI connection referenced by a :class:`.Pool`.
_ConnectionRecord:
    + __pool
    + connection: A reference to the actual DBAPI connection being tracked
    + connection = pool._invoke_creator(self)
    + def checkout(cls, pool) -> fairy:
