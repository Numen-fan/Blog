# Android源码—Handler

## post/sendMessage方法

最终会走到handler中的enqueueMessage方法，然后走走到quene.enqueneMessage(msg, uptimeMillis)，这里面将msg放到消息队列中，这个队列是链表实现的，按照msg的when进行的排序，并且进行了synchronized加锁，确保添加数据的线程安全。

之所以采用链表的数据结构，原因是链表方便插入。

