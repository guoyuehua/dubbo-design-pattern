### Template Method模板方法模式

定义一个操作中的算法骨架，而将一些步骤延迟到子类中，使得子类可以不改变一个算法的结构即可重定义该算法的某些特定步骤。

##### 适应性：

1. 一次性实现一个算法的不变的部分，并将可变的行为留给子类来实现；
2. 各子类中公共的行为应被提取出来并集中到一个公共父类中以避免代码重复；
3. 控制子类扩展。

##### 效果：

1. 模板方法是一种代码复用的基本技术；
2. 模板方法导致一种反向的控制结构，有时称为“好莱坞法则”，即“别找我们，我们找你”。

##### 结构：



##### Dubbo中的应用：
