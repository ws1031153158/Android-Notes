# Enum
最终实现原理还是类，编译完成后，为每一种类型生成静态对象，对象需要的内存空间大于静态常量，对象中默认有一个字符数组空间的申请。  
可通过 ProGuard 将简单枚举优化为一个整型（int），也有限制，以下无法优化：  
1.枚举实现了自定义接口，并被调用  
2.发生类型转换（使用了不同签名存储枚举、使用 instanceOf 指令判断、对枚举强转）  
3.使用枚举加锁操作  
4.调用静态方法 valueOf  
5.定义可以外部访问的方法  