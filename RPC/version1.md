# V1.0
1. 客户端获取ClientProxy对象获取代理对象，ClientProxy通过反射构建request对象，通过IOClient的sendRequest方法和服务端进行数据传输
2. 服务端指定监听端口，如果没有链接会一直处于堵塞状态，有链接来了会创建一个新的线程WorkThread
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTc1NDUwNTE4MF19
-->