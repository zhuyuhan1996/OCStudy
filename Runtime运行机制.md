- 1.RunTime简称运行时。OC就是"运行时机制"，也就是在运行时候的一些机制，其中最主要的是消息机制。
- 2.对于C语言，"函数的调用在编译的时候会决定调用哪个函数".
- 3.对于OC的函数，属于"动态调用"过程，在编译的时候并不能决定真正调用哪个函数，只有在真正运行的时候才会根据函数的名称找到对应的函数来调用。
- 4.事实证明：
    1>在编译阶段，OC可以"调用任何函数"，即使这个函数并未实现，只要声明过就不会报错。
    2>在编译阶段，C语言调用"未实现的函数"就会报错。

消息机制：
- 1.OC 在向一个对象发送消息时，runtime库会根据对象的isa指针找到该对象对应的类或其父类中查找方法。
- 2.注册方法编号（这里用方法编号的好处，可以快速查找）。
- 3.根据方法编号去查找对应方法。
- 4.找到只是最终函数实现地址，根据地址去方法区调用对应函数

交换方法：

---
- 系统首先找到消息的接收对象，然后通过对象的isa找到它的类。
- 在它的类中查找method_list，是否有selector方法。
- 没有则查找父类的method_list。
- 找到对应的method，执行它的IMP。
- 转发IMP的return值。

实例对象的isa指针指向类，类的isa指针指向其元类（metaClass）。对象就是一个含isa指针的结构体。类存储实例对象的方法列表，元类存储类的方法列表，元类也是类对象。

SEL又叫选择器，是表示一个方法的selector的指针,映射方法的名字。Objective-C在编译时，会依据每一个方法的名字、参数序列，生成一个唯一的整型标识(Int类型的地址)

0.1-检查target是否为nil。如果为nil，直接cleanup，然后return。(这就是我们可以向nil发送消息的原因。) 如果方法返回值是一个对象，那么发送给nil的消息将返回nil；

如果方法返回值为指针类型，其指针大小为小于或者等于sizeof(void*)，float，double，long double 或者long long的整型标量，发送给nil的消息将返回0；

如果方法返回值为结构体,发送给nil的消息将返回0。结构体中各个字段的值将都是0；如果方法的返回值不是上述提到的几种情况，那么发送给nil的消息的返回值将是未定义的。

0.2-如果target非nil，在target的Class中根据Selector去找IMP。（因为同一个方法可能在不同的类中有不同的实现，所以我们需要依赖于接收者的类来找到的确切的实现）

1-首先它找到selector对应的方法实现:

1.1-在target类的方法缓存列表里检查有没有对应的方法实现，有的话，直接调用。

1.2-比较请求的selector和类方法列表中的selector，对应的话，直接调用。 

1.3-比较请求的selector和父类方法列表，父类的父类，直至根类，如果有对应，则直接调用。（方法重写拦截父类方法的原理）

2-调用方法实现，并将接收者对象及方法的所有参数传给它。

3-最后，将实现函数的返回值作为自己的返回值。



动态实现：
Method Swizzling可以在运行时通过修改类的方法列表中selector对应的函数或者设置交换方法实现，来动态修改方法。可以重写某个方法而不用继承，同时还可以调用原先的实现。通常应用于在category中添加一个方法。
为保证改变方法引起冲突，确保方法混用只能一次性：
比如，在+load方法或者dispatch_once中执行。
ISA Swizzling可以动态修改对象的isa指针，改变对象的类，类似于创建子类实现相同的功能。KVO即是同过ISA Swizzling实现的。