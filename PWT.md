对于类class来说,每个类型都会创建虚函数表指针,指向一个叫做V-Table的表.拥有继承关系的子类会在虚函数表内通过继承顺序(C++可以实现多继承)去展示虚函数表指针.

但是对于swift来说,class类和struct结构体的实现是不同的,而属于结构体的协议Protocol,可以拥有属性和实现方法,管理Protocol Type方法分派的表就叫做Protocol Witness Table

和V-table一样,Protocol Witness Table(简称PWT)内存储的是方法数组,里面包含了方法实现的指针地址,一般我们调用方法时,是通过获取对象的内存地址和方法的位移offset去查找的

value witness table的结构如上,是用于管理遵守了协议的Protocol Type实例的初始化,拷贝,内存消减和销毁的.

value witness table还可以拆分为%relative_vwtable和%absolute_vwtable,我们这里先不做展开,之后会对这部分内容进行补充.

value witness table和protocol witness table通过分工,去管理Protocol Type实例的内存管理(初始化,拷贝,销毁)和方法调用。