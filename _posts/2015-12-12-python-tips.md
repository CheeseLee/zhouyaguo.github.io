---
layout: post
title: "Python Tips"
comments: true
categories:
- python
---


- a=0xffff  十六进制表示
- r''  排除转义符号
- unicode用两个字节表示一个字符，存纯英文会浪费，于是出现可变长的UTF-8。UTF-8中，英文占一个字节，汉字占三个字节
- python中，u'' 表示unicode
- unicode转换成utf-8： u'我'.encode('utf-8')    --> '\xe6\x88\x91'
- utf-8转换成unicode： '\xe6\x88\x91'.decode('utf-8')   -> u'\u6211'
- 当python源码中包含中文时，要注明： `# -*- coding: utf-8 -*-`
- python2.x由于历史原因，虽然支持unicode，但需要写成 u'xxx'，python3.x中不需要。
- list
  - [] 访问元素
  - append、insert、pop
  - 元素类型可以不同
- tuple
  - a=(1,) 与 a=(1) 不同
- raw_input返回的永远是string
- dict： hash算法，空间换时间
- 命令行中，`help(函数名)`查看函数的帮助
- 类型强转： int(1.2) int('12') float(3) bool(1) bool('') unicode(100)
- 函数名其实就是一个函数对象的引用，可以赋值给其他变量，等于起个别名。
- 函数没有return时，自动return None。return None可以简写成return
- 返回多个值其实就是返回了一个tuple
- \*args是可变参数，args接收的是一个tuple
- \*\*kw是关键字参数，kw接收的是一个dict
- python没有对尾递归做优化
- 任何可迭代对象都可以作用于for循环
- dict迭代的是key，如果要迭代value，用for value in d.itervalues()，如果要同时，用 for k, v in d.iteritems()
- isinstance('abc', Iterable) 判断是否可迭代
- Generator: 边用边生成，而不是一次性生成。 `a = (x * x for x in range(5)) a`就是一个generator，注意：和 `[x * x for x in range(5)]` 不同
- 如果一个函数中包含`yield`，那么该函数就不是一个普通的函数，而是一个generator
- generator和普通函数的执行流程不一样，函数是顺序执行，遇到return或最后一句时就返回，而generator调用next()时才执行，遇到yield就返回，再次执行时，从上次的yield处继续执行。
- python内建了map()，reduce()，filter()函数。这些函数接受两个参数，一个是函数，一个是序列。
- reduce对参数函数的要求是要有两个参数
- filter读参数函数的要求是返回值是bool类型
- map对每个元素做function，reduce是将计算的结果和下一个值再次计算
- python内置的sorted()可以对list进行排序
- lambda关键字表示匿名函数，例如：`f = lambda x: x * x`。匿名函数有个限制：只能有一个表达式，不用写return，返回值就是结果。
- decorator本质就是一个返回函数的高阶函数，实现aop。
- functools.wraps的作用是：保持原来的 `__name__`

  ```
  import functools

  def log(func):
      @functools.wraps(func)
      def wrapper(*args, **kw):
          print 'call %s():' % func.__name__
          return func(*args, **kw)
      return wrapper
  ```
- 偏函数: functools.partial的作用就是，把一个函数的某些参数给固定住（也就是设置默认值），返回一个新的函数，调用这个新函数会更简单。当函数的参数个数太多，需要简化时，使用functools.partial可以创建一个新的函数，这个新函数可以固定住原函数的部分参数，从而在调用时更简单
- 一个.py文件就是一个module
- 每一个包目录下面都会有一个 `__init__.py`的文件，这个文件是必须存在的，否则，Python就把这个目录当成普通目录，而不是一个包。`__init__.py`可以是空文件，也可以有Python代码，因为`__init__.py`本身就是一个模块
- `__xxx__` 是特殊变量，可以直接访问，不是private变量
- `_xxx` 虽然我可以被访问，但是，请把我视为私有变量，不要随意访问
- `__xxx` 不能直接从外部访问
- import types types.StringType
- 判断类型用type()函数，判断继承类的类型用isinstance()函数
- dir()获取对象的所有属性和方法
- 调用len()函数会自动调用`__len__()`方法
- `setattr()`可以直接动态操作一个对象
- `__slots__`用来限制该class能动态添加的属性
- Python内置的`@property`装饰器负责把一个方法变成属性调用
- Mixin的目的就是给一个类增加多个功能，由于python支持多重继承，所以很轻松的实现了Mixin的功能。
- `__str__()` print时自动调用
- `__repr__()` 返回程序开发者看到的字符串，是为调试服务的
- `__iter__` 如果想被for循环使用，必须实现该方法，然后for循环会不断调用next()方法，直到遇到StopIteration()方法
- `__getitem__` 像list一样访问元素
- `__getattr__` 只有在没有找到属性的情况下，才调用__getattr__，已有的属性，不会在__getattr__中查找
- `__call__` 对类的实例直接调用时，会自动调用该方法。比如：原来要instance.method()，现在可以instance()，看起来像函数了。
- 通过callable()函数，我们就可以判断一个对象是否是“可调用”对象，即是否实现了`__call__`
- type()可以动态创建类，而不需要使用class定义的方式，例如： type('Hello', (object,), dict(hello=fn))，通过type()函数创建的类和直接写class是完全一样的，因为Python解释器遇到class定义时，仅仅是扫描一下class定义的语法，然后调用type()函数创建出class
- 元类metaclass：除了使用type()动态创建类以外，要控制类的创建行为，还可以使用metaclass
- exception的stack要从上往下看
- logging.exception(e) 打印stack
- raise语句如果不带参数，就会把当前错误原样抛出
- 测试类要继承`unittest.TestCase`
- 只有`test`开头的方法才会被执行
- python -m unittest xxx 执行
- setUp()和tearDown()是两个特殊的方法，会自动调用
- Python内置的“文档测试”（doctest）模块可以直接提取注释中的代码并执行测试
- `with open('/path/to/file', 'r') as f:`
- `f = open('/Users/michael/test.jpg', 'rb')` 二进制模式打开
- codecs模块方便解码： `with codecs.open('/Users/michael/gbk.txt', 'r', 'gbk') as f:`
- `shutil` module很方便，是os模块的补充
- 序列化在python中叫`pickling`，其他语言中称为： serialization，marshalling
- `cPickle`模块是c语言写的，专门处理序列化
- `cPickle.dumps()`dump成str，`cPickle.dump()`dump到文件中，`cPickle.loads()`反序列化str成对象，`cPickle.load()`从文件中反序列化出对象
- `json`模块提供的方法和cPickle模块类似，都有dump，dumps，load，loads方法
- 通常class的实例都有一个`__dict__`属性，它就是一个dict，用来存储实例变量
- `os.fork()`创建子进程，如果返回值为0，则是子进程，否则是父进程
- 由于windows平台不支持fork方法，所以python提供`multiprocessing`模块来跨平台多进程，multiprocessing模块封装了fork()调用，`multiprocessing`模块提供Process类代表进程。创建子进程时，只需要传入一个执行函数和函数的参数，创建一个Process实例，用start()方法启动
- 如果要启动大量的子进程，可以用进程池的方式批量创建子进程，from multiprocessing import Pool
- 多进程间如何通信？multiprocessing模块提供了Queue，Pipes
- python的线程编程需要用到`threading`模块
- Python的threading模块有个current_thread()函数，它永远返回当前线程的实例
- 多线程中，所有变量都由所有线程共享。创建一个锁就是通过threading.Lock()来实现，使用方法是先lock.acquire()，然后lock.release()，并确保lock.release()是在finally中
- Python的线程虽然是真正的线程，但解释器执行代码时，有一个GIL锁：Global Interpreter Lock，任何Python线程执行前，必须先获得GIL锁，然后，每执行100条字节码，解释器就自动释放GIL锁，让别的线程有机会执行。这个GIL全局锁实际上把所有线程的执行代码都给上了锁，所以，多线程在Python中只能交替执行，即使100个线程跑在100核CPU上，也只能用到1个核
- GIL是Python解释器设计的历史遗留问题，通常我们用的解释器是官方实现的CPython，要真正利用多核，除非重写一个不带GIL的解释器。所以，在Python中，可以使用多线程，但不要指望能有效利用多核
- ThreadLocal的好处是让线程有了自己的空间，而不用全局共享的，同时又避免了作为参数传来传去。
- 多进程比多线程稳定些，因为多线程如果其中一个线程有问题，有可能导致整个进程有问题。
- 大部分web应用都是IO密集型
- 单进程的异步编程模型称为协程，有了协程的支持，就可以基于事件驱动编写高效的多任务程序
- `import re` `re.match(r'^(\d+)(0*)$', '102300').groups()`
- `re.compile(r'^(\d{3})-(\d{3,8})$')` 先预编译，然后在match，提高效率
- collections模块
  - 用namedtuple可以很方便地定义一种数据类型，它具备tuple的不变性，又可以根据属性来引用，使用十分方便
  - deque 是双向列表，解决了list插入和删除低效的问题
  - defaultdict（带默认值的dict） 当key不存在时，不再抛出KeyError，而是返回一个默认值
  - OrderedDict，有顺序的dict
  - Counter实际上是dict的一个子类
- base64模块
  - base64.b64encode
  - base64.b64decode
- struct模块：解决str和其他二进制数据类型的转换
- hashlib模块
  - hashlib.md5()
  - hashlib.sha1()
- itertools模块提供的全部是处理迭代功能的函数，它们的返回值不是list，而是迭代对象，只有用for循环迭代的时候才真正计算
- from HTMLParser import HTMLParser 方便的解析html页面
- PIL：Python Imaging Library，已经是Python平台事实上的图像处理标准库了
- Python对SMTP支持有smtplib和email两个模块，email负责构造邮件，smtplib负责发送邮件
- Python内置一个poplib模块，实现了POP3协议，可以直接用来收邮件。要把POP3收取的文本变成可以阅读的邮件，还需要用email模块提供的各种类来解析原始文本，变成可阅读的邮件对象
- 数据库操作步骤
  - import驱动
  - 连接conn
  - 创建cursor
  - 通过cursor执行sql
  - 通过cursor获取结果
  - conn提交事务
  - 关闭cursor
  - 关闭conn
- SQLAlchemy使用步骤
  - pip install sqlalchemy
  - 导入SQLAlchemy
  - 创建类，定义表名和字段名
  - create_engine('数据库类型+数据库驱动名称://用户名:口令@机器地址:端口号/数据库名') 初始化数据库链接
  - 创建session： sessionmaker(bind=engine)
  - session.add(对象)
  - session.commit()
  - session.close()
- 协程（Coroutine）
  - coroutines是线性处理的，同一时间只有一个coroutine在执行。
  - subroutine是coroutine的特例
  - subroutine只返回一次，并且不保存调用别人后不保存状态，相反，coroutine保存状态，coroutine可以调用coroutine，是通过yield的方式
  - 设计一个支持subroutine的语言只需要预分配一个stack即可，对应的，要支持coroutine的话，就需要预分配多个stack
  - coroutine有什么用？以producer和consumer为例，用coroutines会相当高效，都不需要底层thread切换，也不需要经过OS的公共资源，而是直接coroutine间调用就搞定了。
- green thread是由vm管理的，不是OS管理的，green threads是仿真了多线程环境，不依赖与底层的OS，是运行在user space下。
  - 在多核处理器下，native thread可以自动分配到多核去，而green thread不行
  - java 1.1中，green thread是唯一的线程模型，由于green thread比native thread有很多不足，java后续版本抛弃了green thread
  - python中的eventlet是green thread
