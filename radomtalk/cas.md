# CAS

```
public class CASTest { 
    public static void main(String[] args) { 
        //新建一个原子整型实例，初始值设置为6，不设置则默认为0 
        AtomicInteger integer = new AtomicInteger(6); 
        System.out.println(integer.compareAndSet(6, 8)+"\t 当前值="+integer.get()); 
        System.out.println(integer.compareAndSet(6, 66)+"\t 当前值="+integer.get()); 
    }
}

--- - - - - - - - - - - --- - - - - -
true	 当前值=8
false	 当前值=8
```

**先比较后交换**   核心方法就是上面代码中的 integer.compareAndSet(6, 8)；第一个参数代表期望值，第二个参数表示最后更新成的数值。上面代码在第一次打印的时候，期望值是6，更新值是8，由于该原子整型在声明时设置的初始值就是6，所以**先比较**，发现初始值和期望值是相等的，然**后交换**该值为8；但在第二次打印时，此时内存中的数值已经被更新成8，这时再比较发现8和期望值6不相等，所以无法更新，最后的值还是8。
