# 1. 运行时数据区
![输入图片说明](/imgs/2025-03-25/sOyOJsJGPSPUHAkO.png)

# 2. 对象的创建
Step1:类加载检查

虚拟机遇到一条 new 指令时，首先将去检查这个指令的参数是否能在常量池中定位到这个类的符号引用，并且检查这个符号引用代表的类是否已被加载过、解析和初始化过。如果没有，那必须先执行相应的类加载过程

step2：分配内存

step3：初始化零值

step4：设置对象头

step5：执行init方法

# 3. 对象的访问定位
句柄：
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0NjQzNjAwNzddfQ==
-->