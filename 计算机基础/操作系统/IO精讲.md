# 虚拟文件系统，文件描述符，IO重定向
## PageCache
![PageCache](assets/Pasted%20image%2020220927110938.png)


# 内核中PageCache、mmap作用、java文件系统io、nio、内存中缓冲区作用
**查看linux系统的dirty信息**
```shell
sysctl -a | grep dirty
```

**结果**
```shell
# 读取文件时，并且缓存页占了内存10%的时候，才会完成内存往磁盘写的过程
vm.dirty_background_ratio = 10
# 分配页到可用内存30%的时候，会阻塞，然后将脏数据写到磁盘。恢复后用LRU算法，总的来说，内存IO优先。脏的页要先写到磁盘当中去
vm.dirty_ratio = 30
vm.dirty_writeback_centisecs = 500
vm.dirty_expire_centisecs = 3000
```

**查看一个文件的pagecache信息**
```shell
pcstat 文件名 
```

**总结**
pagecache优点：优化IO效率
pagecache缺点：会丢失数据

## 磁盘IO
**dirty页的持久化**
![](assets/Pasted%20image%2020220927155124.png)

## 总结
![](assets/Pasted%20image%2020220927190423.png)


# Socket编程BIO及TCP参数
























