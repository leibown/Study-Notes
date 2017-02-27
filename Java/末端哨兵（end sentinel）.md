# 末端哨兵（end sentinel）

```java
public class LinkedStack<T> {

    private static class Node<U> {
        U item;
        Node<U> next;

        Node() {
            item = null;
            next = null;
        }

        Node(U item, Node<U> next) {
            this.item = item;
            this.next = next;
        }

        boolean end() {
            return item == null && next == null;
        }
    }

    //创建一个成员变量来充当栈顶元素，此时top对象里面的成员属性都为空。
    private Node<T> top = new Node<>();

    public void push(T item) {
        //把原有的空top放入新对象中的next，这时item就是top对象里面的item，原有的空top就成了现有的top的next
        top = new Node<>(item, top);
    }

    public T pop() {
        T result = top.item;
        //如果top里面的item和next都为空了，说明当前元素为原有的top元素，也就是栈顶元素
        if (top.end())
            top = top.next;
        return result;
    }

}
```

这个例子使用了一个末端哨兵（end sentinel）来判断堆栈何时为空。这个末端哨兵是在构造LinkedStack时创建的。然后，每调用一次push()方法，就会创建一个Node<T>对象，并将其链接到前一个Node<T>对象。当你调用pop()方法时，总是返回top.item，然后丢弃当前top所指的Node<T>，并将top转移到下一个Node<T>，除非你已经碰到了末端哨兵，这时候就不再移动top了。如果已经到了末端，客户端程序还继续调用pop()方法，它只能得到null，说明堆栈已经空了。