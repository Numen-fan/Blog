# equals和hashCode

> 写Java项目有个默认的规则，重写equals方法时，要重写hashCode方法。面试有个老生常谈的题目：
>
> 1. 如果equals为true，那么hashCode一定相等吗？
> 2. 如果hashCode相等，那么对象equals吗？

先说答案吧：

1. 不一定，取决于是否重写hashCode方法。
2. 不一定。

## equals

`equals`定义在Object类中，所有的类都可以重写，默认的实现是比较两个对象的内存地址。

```java
public boolean equals(Object obj) {
        return (this == obj);
    }
```

### 重写equals的规则

重写equals需要满足以下几个原则：

**【自反性】:**  x.equals(x) 一定是true

**【对称性】**:  x.equals(y)  和 y.equals(x)结果一致

**【传递性】**:  a 和 b equals , b 和 c  equals，那么 a 和 c也一定equals。

**【一致性】：**  在某个运行时期间，2个对象的状态的改变不会不影响equals的决策结果，那么，在这个运行时期间，无论调用多少次equals，都返回相同的结果。

**【对null】**:  x.equals(null) 一定是false

---

## hashCode

hashCode同样定义在Object中，来看看默认的实现；

```java
/**
 * Returns a hash code value for the object. This method is
 * supported for the benefit of hash tables such as those provided by
 * {@link java.util.HashMap}.
 *	……………………
 */
public int hashCode() {
    return identityHashCode(this);
}
```

![](https://raw.githubusercontent.com/Numen-fan/BlogPicRepo/main/img/image-20220719151836102.png)

从官方的说明来看，默认实现就是对对象的内存地址做一次转换得到一个int值。简单说就是和内存地址相关。

### 重写hashCode的规则

从方法上的注释可以看出，对象的散列码是为了更好的支持基于哈希机制的Java集合类，例如 Hashtable, HashMap, HashSet 等。

最终的hash是个int值，而不能溢出。不同的对象的hash码应该尽量不同，避免hash冲突。

应该将equals中考虑的字段值都纳入hash计算，得到各个hash分量，最后累加，注意不是简单的累加，详细可见参考资料2。



---

## 对面试题的例证

- 如果equals为true，那么hashCode一定相等吗【不一定】

这点很好理解，如果一个类，只重写了equals，没有重写hashCode，那么就很容易导致，equals为true，但hashCode不同。

```java
static class Person {
    String name;
    int identify;
    public Person(String name, int identify) {
        this.name = name;
        this.identify = identify;
    }
    @Override
    public boolean equals(Object o) {
        if (this == o) {
            return true;
        }
        // 不建议用 o instanceof Person, 如果o是Person的的派生类就会有问题。
        if (o == null || getClass() != o.getClass()) {
            return false;
        }
        Person person = (Person) o;
        return identify == person.identify && Objects.equals(name, person.name);
    }
}
```

```java
Person person1 = new Person("张三", 123);
Person person2 = new Person("张三", 123);
System.out.println("equals:" + person1.equals(person2));
System.out.println("hashCode:" + (person1.hashCode() == person2.hashCode()));
```

```text
运行结果
equals:true
hashCode:false
```

- 如果hashCode相等，那么对象equals吗？ 【不一定】

```java
Integer i = 97 ;
String s = "a" ;
System.out.println(i.hashCode());
System.out.println(s.hashCode());
```

可以看看Integer和String的hashCode源码就清楚了。



### 只重写equals，不重写hashCode有什么影响呢？

hashCode返回一个散列值，如果对象不参与哈希运算，那么重写与否其实也就不那么重要了，但如果涉及到哈希计算，比如HashSet, Hashtable, HashMap这些数据结构，对象插入就必须考虑哈希值了。

可以利用API`Objects.hash(Object... values)`得到哈希值，看源码`result = result * 31 + value.hashCode()`，这是一个较强的哈希计算方式。

```java
public static int hashCode(Object a[]) {
    if (a == null)
        return 0;
    int result = 1;
    for (Object element : a)
        result = 31 * result + (element == null ? 0 : element.hashCode());
    return result;
}
```



## 参考资料

1. https://blog.csdn.net/cora_99/article/details/108383232
2. https://www.cnblogs.com/zhougongjin/p/15175706.html