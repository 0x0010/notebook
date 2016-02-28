最近在java眼社区看到2014年一个朋友关于hash均衡性的一个提问，刚好最近自己也在看这方面的知识，遂写下自己的一些思考。

### 故事背景

在"JavaEye"上看到一篇2010年的一篇关于一致性hash算法的博客，博客中引用的是Memcached客户端SpyMemcached中寻找MemcachedNode的算法展开讲解hash一致性算法。文章地址[Ketama一致性Hash算法](http://langyu.iteye.com/blog/684087)

翻看评论时看到一个朋友提问，**如果key的hash冲突了怎么办？**
我看到之后特别想回复，所以就写了两句：
>如何理解key的哈希冲突？
>
>* 不同的key经过hash运算之后得到同一个hash值？
>
>* 应用中某一个或某几个key非常热？
>
>对于第一种情况输入hash算法问题，hash算法有一个特性是均衡性。均衡性很难保证的hash算法是不合格的。
>
>对于第二种情况，可以在原有hash基础上再次做一次hash，通过二次hash来重建均衡性。ConcurrentHashMap中选择segment就采用了二次hash。
