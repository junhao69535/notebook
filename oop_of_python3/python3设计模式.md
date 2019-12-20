# python3设计模式

## 装饰器模式
装饰器模式允许我们将一个提供核心功能的对象和其他可以改变这个功能的对象“包裹”在一起。装饰器对象实现的接口和核心对象一样。只不过装饰器对象在执行核心代码之前或之后执行了额外的代码。
### 功能：
#### 增强一个组件向另一个组件发送数据时的响应能力
#### 支持多种可选行为
当需要多种可选行为的时候，应当选择装饰器模式，而不是多重继承。

### UML表现形式：
![装饰器模式UML](images/20191210141555567_2926.png =605x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210141555567_2926.png)

### Python中的装饰器@xxx：
这种语法会永久修改原来核心对象，就是核心对象不再存在了，已经被装饰器对象覆盖了。但python3支持用functools.wraps装饰器修饰的核心对象获取原来的核心对象。即几颗获取原来核心对象，也能获取装饰器对象。


##	观察者模式
观察者模式在状态检测和事件处理等场景中非常有用。这种模式确保一个核心对象可以由一组未知并可能正在扩展的“观察者”对象来监控。一旦核心对象的某个值（状态）发生变化，它便通过调用update()方法让所有观察者对象知道状态发生变化。各个观察者在核心对象发生变化时，有可能会负责处理不同的任务。核心对象和观察者对象甚至观察者对象之间都互不关心其他对象正在做什么，只关注自己。

### UML表现形式：
![观察者模式UML](images/20191210141746855_4427.png =659x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210141746855_4427.png)

如图，核心对象必须实现attack()方法，用于给自身添加观察者，也需要实现update()方法，用于通知其他观察者。而观察者对象必须实现观察者接口，实现__call__()方法，用于收到核心对象信息时调用处理。

### 核心：
观察者模式实际上就是将正在被观察的代码和执行观察的代码分离，实现解耦。如果需要新添加对核心对象处理，那么只需新建一个观察者，并添加到核心对象的观察列表中即可。
如原来已经有一个观察者对象，用于把核心对象的信息插入到数据库中，但现在我们多了一个方式，需要把信息插入到redis中。这时候我们只需要新写一个观察者对象并“注册”到被观察对象（核心对象）即可，而不需要修改被观察对象（核心对象）。

观察者模式特别适合用于数据结构不怎么变化，而对数据的操作经常需要变化时。基本不变的数据结构对象就是被观察对象，而需要用于操作数据结构的对象就是观察对象。


## 策略模式
策略模式是一种常见的面向对象编程的抽象模式。针对同一问题，这个模式实现了不同对象采用不同的解决方案。客户端代码可以在运行时动态选择其中最恰当的实现。

### UML表现形式：
![策略模式UML](images/20191210141939086_15591.png =596x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210141939086_15591.png)

如图，User和Abstraction之间关联，而Implementation1和2实现了Abstraction接口。User对象不关心他们怎么实现。User对象只需要根据情况调用对应的实现类即可。如user1对象需要使用策略1（implementation1），那么其在调用的时候传参只需要把implementation1参进去即可。而user2需要使用策略2（implementation），那么其在调用的时候传参只需要把implementation2参进去即可。

连接到策略模式的用户代码仅仅需要知道和它打交道的抽象接口。对同一任务的实现可能会选择完全不同的方法，但是无论采用哪种方法，接口都是完全相同。实际上这和多态没有区别。

### Python中的策略模式：
上面说的策略模式规范在很多面向对象语言中普遍运用，但在python中运用极少。因为这些类仅仅提供了一个函数。我们可以轻松地调用___call__函数使对象可以直接被调用。由于这些对象没有和任何数据关联，我们实际上可以创建一组顶级函数并传递它们来替代。而python中函数就是一级对象。因此可以使用更直接的方式实现策略模式，而不需要创建类对象。


## 状态模式
状态模式的目的是实现状态转移系统，对象显而易见地处于一个特定状态，同时某些活动可能驱动它转变到另一个不同的状态。为了实现这个目的，需要一个管理者类或上下文类为状态转换提供一个接口。在内部，这个类包含一个指向当前状态的指针，每个状态都知道可以转换为哪些其他状态并如何根据调用的操作转换到这个状态。

### UML表现形式
![状态模式UML](images/20191210142034064_15026.png =605x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210142034064_15026.png)

如图，有两个类型的类，一个上下文类和多个状态类。上下文类维护当前状态，并且转发操作到状态类。状态类对其他调用上下文的对象（如User）而言通常是隐藏的。

### 状态和策略模式的对比
虽然两种模式具有相同的结构，其目的却是非常不同的。策略模式用于运行时选择一种算法：一般来说，只有其中一个算法将被选择用作特定的用途。另一方面，状态模式被设计成允许在不同状态之间金星动态切换，正如同某些过程的发展那样。在代码中，最主要的区别在于，策略模式通常对其他策略对象没有意识。而在状态模式中，状态或上下文需要知道它们将会切换到什么样的状态。


## 模板模式
模板模式用于去除重复代码。例如有不同的任务需要完成，但这些任务中的一些（不是全部）步骤相同。相同的步骤在基类中被执行，而不同的步骤会在子类中被重写以提供自定义的行为。

### UML表现形式
![模板模式UML](images/20191210142142566_13860.png =524x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210142142566_13860.png)

如图，Task1继承基类然后重写自己不同的步骤。这些接口向外提供do_process接口，对外界来说内部实现是隐藏的。


## 适配器模式
适配器模式被设计用于与已有代码进行交互。适配器用于允许两个已存在的对象在一起工作，即使他们的接口不兼容。适配器对象的唯一工作就是执行转换。和装饰器模式的区别：装饰器模式通常提供它们所替代的相同接口，而适配器模式在两个不同的接口之间进行匹配。

### UML表现形式：
![适配器UML](images/20191210142218934_5283.png =550x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210142218934_5283.png)

如图，facede对象使Complex Component和Another Component可以一起工作，例如Complex Component是其他模块提供的接口，我们不能修改，而这个接口的参数格式和我们需要提供的Another Component有区别，这时就需要facede适配器对象进行协调。我们提供的Another Component接口接收参数，然后通过facede适配器对象转换成适用Complex Component接口的参数。


## 外观模式
外观模式的目的是为了拥有多个组件的复杂系统提供简单的接口。外观模式允许我们定义一个新的对象来封装该系统的典型使用。

### UML表现形式：
![外观模式UML](images/20191210142300863_30288.png =550x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210142300863_30288.png)

外观模式和适配器模式的区别：外观模式试图从一个复杂的系统抽象出一个简单的接口；而适配器只不过是想将一个现有的接口匹配到另一个。假设我们有一个邮件应用程序，内部处理逻辑非常复杂，这时我们可以抽象出一个简单接口提供给外界，处理好所有底层。外界使用这个抽象出的接口便很容易使用这个邮件应用程序，同时如果外界想访问底层，以前的底层接口也是提供的，因此既可访问抽象出的接口，也能访问底层接口，按需所取。


## 享元模式
享元模式是一种内存优化模式。享元模式的基本原理是：保证共享同一状态的对象可以同时使用该共享状态的内存。

### UML表现形式
![享元模式UML](images/20191210143529477_23167.png =550x)

![image](https://github.com/junhao69535/notebook/blob/master/oop_of_python3/images/20191210143529477_23167.png)

每个享元都没有指定状态；在任何需要对特定状态执行操作的时候，该状态就需要被代码调用，传递到该享元处。一般来说，享元返回的工厂是一个单独的对象；它是为一个能够识别享元的给定关键词而返回的。和单例模式的区别是，单例模式只返回唯一实例，而享元是根据关键词返回不同的实例。在许多语言中，工厂不是作为一个单独的对象，而是作为Flyweight类本身的静态方法来执行。但在python中，享元工厂经常使用__new__构造，类似单例模式。

### 代码
```
#!coding=utf-8
"""
    享元模式
"""
import weakref


class CarModel:
    # 避免不需要使用该对象时，类字典还对这个对象存在引用而导致内存不能回收
    _models = weakref.WeakValueDictionary()

    def __new__(cls, model_name, *args, **kwargs):
        # 享元工厂，提供唯一的对象
        model = cls._models.get(model_name)
        if not model:
            model = super().__new__(cls)
            cls._models[model_name] = model
        return model

    def __init__(self, model_name, air=False, tilt=False,
                 cruise_control=False, power_locks=False,
                 alloy_wheels=False, usb_charger=False):
        if not hasattr(self, "initted"):  # 保证只初始化一次
            self.model_name = model_name
            self.air = air
            self.tilt = tilt
            self.cruise_control = cruise_control
            self.power_locks = power_locks
            self.alloy_wheels = alloy_wheels
            self.usb_charger = usb_charger
            self.initted = True

    def check_serial(self, serial_number):
        print("Sorry, we are unable to check the serial number {0} on the {1} "
              "at this time".format(serial_number, self.model_name))

class Car:
    # 创建一个类来存储附加信息，以及对享元的引用：
    def __init__(self, model, color, serial):
        self.model = model
        self.color = color
        self.serial = serial

    def check_serial(self, serial_num):
        return self.model.check_serial(serial_num)


dx = CarModel("FIT DX")  # 汽车模型使用了享元模式，同一个模型是全局唯一的
lx = CarModel("FIT LX", air=True, cruise_control=True, power_locks=True, tilt=True)
car1 = Car(dx, "blue", "12345")
car2 = Car(dx, "black", "12346")
car3 = Car(lx, "red", "12347")

print(id(lx))
del lx
del car3
import gc
print(gc.collect())
lx = CarModel("FIT LX", air=True, cruise_control=True, power_locks=True, tilt=True)
print(id(lx))
lx = CarModel("FIT LX")  # 获取的和上面lx是同一个对象
print(id(lx))
print(lx.air)
```
显然，采用享元模式会使代码更加复杂。享元模式是转为节省内存而设计的；如果我们有数十或成百上千相似的对象，将它们的相同特性整合进一个享元中可以极大地减少对内存的消耗。