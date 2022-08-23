## 装饰器详解

### 闭包

要想理解装饰器，首先得弄明白什么是闭包

> 函数定义和函数表达式位于另一个函数的函数体内。而且，这些内部函数可以访问它们所在的外部函数中声明的所有局部变量、参数和声明的其他内部函数。当其中一个这样的内部函数在包含它们的外部函数之外被调用时，就会形成闭包



```python
def wrapper():
    name = "ivy"
    
    def inner():
        print(name)
    return inner
    
g = wrapper()

g()
```

根据上面的定义，wrapper函数里面定义了inner函数，inner函数里面使用了wrapper中的name变量。wrapper函数被调用后会返回内部的inner函数给g，当g再次被调用时，就会形成闭包。

### 为什么要使用装饰器

需求:  在不修改调用代码的情况下，计算一段业务代码的耗时

```python
import time


def your_work():
    start = time.time()
    time.sleep(3)  # 模仿业务耗时
    end = time.time()
    print(end - start)
    
    
your_work()

```

这里的time.sleep模拟任务的耗时，从这段代码中可以看到，计时和实际的业务代码写在一个函数中，可读性差，如果从面向对象的角度来看，这段代码违反了单一职责原则。



```python
import time


def your_work():
    time.sleep(3)
    
def spent_time(func):
    start = time.time()
    func()
    end = time.time()
    print(end - start)
    
    
spent_time(your_work)

```

将上面的代码修改，将计时和业务代码分开，虽然可读性高了，但是调用方式发生了改变，不符合需求。



这个时候，就要使用到装饰器了。

### 装饰器原理

```python
import time


def wrapper(func):
    print("wrapper....")

    def inner():
        start = time.time()
        func()
        end = time.time()
        print(end - start)
    
    return inner


def your_work():
    time.sleep(3)

g = wrapper(your_work)

g()

```

分析：
* 利用上面的闭包原理，wrapper函数接受一个函数作为参数，当wrapper函数被执行的时候，实际执行print("wrapper....")和返回inner函数，因为执行wrapper函数的时候inner函数没有被执行，所以传进来的参数函数也没有被执行。
* 在上述代码中，只有当wrapper函数的接受者(inner的接受者)g被调用的时候，inner才会被执行。因为inner里面有且仅有func,也就是传进来的your_work,所以当g被调用的时候，也就是your_work被调用的时候，这时候可以把g看做your_work



```python
import time


def wrapper(func):
    print("wrapper....")

    def inner():
        start = time.time()
        func()
        end = time.time()
        print(end - start)
    return inner


@wrapper
def your_work():
    time.sleep(3)

your_work()
```

python提供了一种语法糖@，@的作用只是将所装饰的函数(your_work)传入装饰器函数(wrapper)作为参数，并执行装饰器函数。当被装饰的函数(your_work)被调用的时候，实际上是调用的装饰器内部返回的函数(inner),  这样就可以完成不修改调用处代码，又能达到计时目的。



### 装饰器返回值和装饰器参数

上述的所装饰的函数(your_work)既没有返回值也没有参数，因为被装饰的函数最后会执行装饰器函数所返回的闭包函数(inner), 所以inner所接受的参数和返回的值就是被装饰函数返回的值和接受的参数，因为装饰器最初的目的是为原有的代码增加功能而不修改原来的逻辑，所以函数的返回值和函数的参数应该和被装饰的函数一致。

```python
import time


def wrapper(func):
    def inner(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(end - start)
        return result
    
    return inner


@wrapper
def do_something(num):
    time.sleep(3)
    return num
	
```

在inner里面使用不定长参数来接受任意参数，在调用参数func的函数时候将接受的参数传递给func，在将函数运行的结果保存，最后返回即可。



### 带参数的装饰器

```python
from flask import Flask

app = Flask(__name__)


@app.route('/index')
def index():
    return 'index page'


if __name__ == '__main__':
    app.run()

```



熟悉flask 的应该很熟悉这段代码，flask 使用装饰器接受参数来定义路由。

模仿flask写一个接受参数的装饰器

```python
class App:
    def __init__(self):
        self.urls = dict()

    def route(self, url:str):
        def wrapper(func):
            self.urls[url] = func
            return func

        return wrapper


app = App()


@app.route('/index')
def index():
    return 'index page'

```

在App.route方法中，接受一个字符串作为参数，再在内部将接受的参数func做一个映射关系存入实例属性中。

注意
* 在app.route对index进行装饰的时候，app.route接受了一个字符串作为参数，主动执行了route方法，返回了wrapper函数，然后@语法在使用route返回的wrapper对index进行了装饰。所以index函数最后传入了wrapper中而不是直接传入route。



### 类装饰器

```python
import time


class Wrapper:

    def __init__(self, func):
        self.func = func

    def __call__(self, *args, **kwargs):
        start = time.time()
        result = self.func(*args, **kwargs)
        end = time.time()
        print(end - start)
        return result


@Wrapper
def your_work(num):
    time.sleep(3)
    return num

```



类也可以作为一个装饰器，根据上述原理，装饰器会将your_work传入Wrapper，由于类直接被调用会触发它的__init___方法，所以在实例化方法里面接受被装饰器的函数作为实例函数属性保存。此时被装饰的函数可以看到装饰器类的一个实例变量，这个函数被调用的时候(可以看成是类的实例变量被调用，此时会触发该类的__call__方法)，最后在__call__方法里面写上自己的处理逻辑即可

### 一个简单的缓存装饰器

```python
import time


class Cache:

    def __init__(self, func):
        self.func = func
        self._value = None

    def __call__(self, *args, **kwargs):
        if self._value is None:
            result = self.func(*args, **kwargs)
            self._value = result
        return self._value


@Cache
def your_work(num):
    time.sleep(3)
    return num


print(your_work(3))
print(your_work(4))

```

在这个例子中，假设your_work是一个耗时的任务，但是它每次返回的结果都相同，所以可以在它第一次运行完毕之后将它的结果保存起来，下次再次调用它的时候直接将结果返回，这样就避免了重复的运算。





### 装饰类

装饰器不仅可以装饰函数，也可以装饰类，原理和函数一致，只用将传递的参数函数换成类即可！



### 多个装饰器

```python
from tornado.httpclient import AsyncHTTPClient
from tornado.web import gen
from tornado.ioloop import IOLoop


class Request:

    @classmethod
    @gen.coroutine
    def get_resp(cls, url):
        client = AsyncHTTPClient()
        resp = yield client.fetch(url)
        print(resp.body)


if __name__ == '__main__':
    Request.get_resp('https://www.baidu.com')
    IOLoop.instance().start()

```

熟悉`tornado`的都应该熟悉这段代码，这是`tornado`实现一个异步获取网络响应的最基本的写法。



当一个对象被多个装饰器所装饰，那么装饰器的执行顺序是怎样的？

```python
def test2(func):
    def inner():
        print("test2----before")
        func()
        print("test2----after")

    return inner


def test3(func):
    def inner():
        print("test3----before")
        func()
        print("test3----after")

    return inner


@test3
@test2
def test1():
    print("test1---")


if __name__ == '__main__':
    test1()



""" 运行结果
test3----before
test2----before
test1---
test2----after
test3----after
"""

```

#### 洋葱模型

从上面的运行结果可以看出，当一个对象被多个装饰器所装饰的时候，所有的需要在该对象本身`test1`运行前而运行的顺序是根据装饰器装饰的顺序从上而下的，而所有需要在被装饰对象`test1`本身运行后所运行的顺序是装饰器装饰顺序由下而上的。

如果读懂了上面的装饰器原理，那么很容易理解这个例子



我们可以把这个顺序看成一个洋葱模型，先从外面一层层进去，最后从里面一层层出来。



### 装饰器的基本原则

装饰器的本意是在不修改原来的代码和调用方式的情况下给被装饰的对象增加新的功能，所以我们在书写装饰器的时候如果没有特殊需求的话，尽量不要在装饰器内部去修改被装饰器对象的返回值(如果有返回值)，这样可以使用多个装饰器来装饰而不容易发生错误。




装饰器的本质就是一个可接收参数callable的对象去装饰另外一个callable对象。