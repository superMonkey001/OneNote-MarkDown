有一个类A
```java
public class A{  
    public void m(Callable c) {  
        System.out.println("call");  
    }  
  
    public void m(Runnable r) {  
        System.out.println("run");  
    }  
}
```
主程序
```java
public static void main(String[] args) {  
    A a = new A();  
    a.m(() -> 1);  
}
```
执行结果是
 ![](assets/Pasted%20image%2020221003140959.png)

如果主程序改成如下代码： 
```java
public static void main(String[] args) {  
    A a = new A();  
    a.m(() -> System.out.println("..."));  
}
``` 
执行结果是：
 ![](assets/Pasted%20image%2020221003141114.png)

因此，由于Runnable接口和Callable接口中的方法，可以唯一确定（Runnable接口的方法run()没有返回值，而Callable接口的方法有返回值)，因此Lambda表达式可以被正确执行。


 