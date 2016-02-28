最近在java眼社区看到2014年一个朋友关于hash均衡性的一个提问，刚好最近自己也在看这方面的知识，遂写下自己的一些思考。

### 故事背景

在"JavaEye"上看到一篇2010年的一篇关于一致性hash算法的文章，文章中引用的是Memcached客户端SpyMemcached中寻找MemcachedNode的算法展开讲解hash一致性算法。文章地址[Ketama一致性Hash算法](http://langyu.iteye.com/blog/684087)

翻看评论时看到一个朋友提问，**如果key的hash冲突了怎么办？**
我看到之后特别想回复，所以就写了两句：
>如何理解key的哈希冲突？
>
>* 不同的key经过hash运算之后得到同一个hash值？
>
>* 应用中某一个或某几个key非常热？
>
>对于第一种情况输入hash算法问题，hash算法有一个特性是均衡性。均衡性都无法保证的hash算法是不合格的。
>
>对于第二种情况，可以在原有hash基础上再次做一次hash，通过二次hash来重建均衡性。ConcurrentHashMap中选择segment就采用了二次hash。

### SpyMemcached均衡性测试

先看一下SpyMemcached里key的均衡性表现的测试，下边是完整的测试代码：
````java
import org.apache.commons.lang3.RandomStringUtils;

import java.io.UnsupportedEncodingException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.util.*;

/**
 * Rehash algorithm
 * <p>
 * Created by Sam on 2016/2/28.
 */
public class RehashTest {

    private static MessageDigest md5Digest = null;

    static {
        try {
            md5Digest = MessageDigest.getInstance("MD5");
        } catch (NoSuchAlgorithmException e) {
            throw new RuntimeException("MD5 not supported", e);
        }
    }

    /**
     * Get the md5 of the given key.
     */
    public static byte[] computeMd5(String k) {
        MessageDigest md5;
        try {
            md5 = (MessageDigest) md5Digest.clone();
        } catch (CloneNotSupportedException e) {
            throw new RuntimeException("clone of MD5 not supported", e);
        }
        try {
            md5.update(k.getBytes("UTF-8"));
        } catch (UnsupportedEncodingException uee) {
            throw new RuntimeException("UTF-8 encoding not supported", uee);
        }
        return md5.digest();
    }

    /**
     * Ketama虚拟节点Position计算方法。
     *
     * @param nodeKey   节点Key
     * @param iteration 虚拟节点编号
     * @return 虚拟节点Hash值
     */
    private static List<Long> ketamaNodePositionsAtIteration(String nodeKey, int iteration) {
        List<Long> positions = new ArrayList<Long>();
        byte[] digest = computeMd5(nodeKey + "-" + iteration);
        for (int h = 0; h < 4; h++) {
            Long k = ((long) (digest[3 + h * 4] & 0xFF) << 24)
                    | ((long) (digest[2 + h * 4] & 0xFF) << 16)
                    | ((long) (digest[1 + h * 4] & 0xFF) << 8)
                    | (digest[h * 4] & 0xFF);
            positions.add(k);
        }
        return positions;
    }

    /**
     * Ketama hash algorithm
     *
     * @param k key
     * @return hash code
     */
    public static long hash(String k) {
        long rv;
        byte[] bKey = computeMd5(k);
        rv = ((long) (bKey[3] & 0xFF) << 24)
                | ((long) (bKey[2] & 0xFF) << 16)
                | ((long) (bKey[1] & 0xFF) << 8)
                | (bKey[0] & 0xFF);
        return rv & 0xffffffffL; /* Truncate to 32-bits */
    }

    /**
     * 为了方便测试，对源码包里的方法进行部分修改，主体逻辑未做改变。
     *
     * @param hash    key的hash值
     * @param nodeMap 虚拟节点集合
     */
    static void getNodeForKey(long hash, TreeMap<Long, Long> nodeMap) {
        //final MemcachedNode rv;
        if (!nodeMap.containsKey(hash)) {
            // Java 1.6 adds a ceilingKey method, but I'm still stuck in 1.5
            // in a lot of places, so I'm doing this myself.
            SortedMap<Long, Long> tailMap = nodeMap.tailMap(hash);
            if (tailMap.isEmpty()) {
                hash = nodeMap.firstKey();
            } else {
                hash = tailMap.firstKey();
            }
        }
        // 对命中的虚拟节点+1
        nodeMap.put(hash, nodeMap.get(hash) + 1);
    }


    public static void main(String[] args) {
        // 写死两个memcached服务地址。真实情况下SpyMemcached也采用这种形式来构造缓存节点key
        String[] nodeKeys = {"192.168.199.200:18000", "192.168.199.201:18000", "192.168.199.202:18000", "192.168.199.203:18000", "192.168.199.204:18000"};
//        String[] nodeKeys = { "192.168.199.203:18000", "192.168.199.204:18000"};

        //为了方便统计，构造三个Map，分别存储实际节点，虚拟节点和虚拟节点与实际节点的映射
        // 命中的计数在虚拟节点上，统计需要统计实际节点的命中数，实际节点又会被分成多个虚拟节点
        Map<String, Long> actualNodeMap = new HashMap<>(3);
        TreeMap<Long, Long> virtualNodeMap = new TreeMap<>();
        Map<Long, String> mappingMap = new HashMap<>();

        // 构造虚拟节点，repetition表示节点的复用数，测试发现，repetition越大，均衡性越好。
        int repetition = 256;
        for (String nodeKey : nodeKeys) {
            actualNodeMap.put(nodeKey, 0L);
            // 虚拟节点数量
            for (int i = 0; i < repetition; i++) { //非常重要
                for (Long hashedKey : ketamaNodePositionsAtIteration(nodeKey, i)) {
                    virtualNodeMap.put(hashedKey, 0L);
                    mappingMap.put(hashedKey, nodeKey);
                }
            }
        }

        Long requestCount = 1000000L; // 模拟请求次数

        // 模拟虚拟节点的命中
        for (long i = 0L; i < requestCount; i++) {
            getNodeForKey(hash(RandomStringUtils.random(32)), virtualNodeMap);
        }

        // 汇总虚拟节点的命中数
        for (Map.Entry<Long, Long> entry : virtualNodeMap.entrySet()) {
            actualNodeMap.put(mappingMap.get(entry.getKey()), actualNodeMap.get(mappingMap.get(entry.getKey())) + entry.getValue());
        }

        //统计物理节点的命中数，以及计算命中百分比
        for (Map.Entry<String, Long> entry : actualNodeMap.entrySet()) {
            System.out.println(entry.getKey() + "-" + (entry.getValue() * 100.0 / requestCount) + "-" + entry.getValue());
        }
    }
}
````
测试结果如下：
````
192.168.199.203:18000-24.7474-247474
192.168.199.201:18000-17.6973-176973
192.168.199.200:18000-16.5573-165573
192.168.199.202:18000-19.5754-195754
192.168.199.204:18000-21.4226-214226
````
从测试的结果来看，均衡性不是非常理想。

### 修改物理节点的repetition实现理想的均衡效果。

测试代码中构造虚拟节点处有一个变量repetition，这个变量代表实际节点的复用数。也就是说，如果repetition=4（spymemcached包里的默认值），那么在hash环中，每个实际节点对应4*4个虚拟节点，命中其中任何一个虚拟节点就等同于命中实际节点。

现在将repetition调大，看看命中的均衡性如何？ 我将repetition调整到256，也就是意味着环中每个实际节点有256*4个虚拟节点。
````java
    // 构造虚拟节点，repetition表示节点的复用数，测试发现，repetition越大，均衡性越好。
    int repetition = 256;
    for (String nodeKey : nodeKeys) {
        actualNodeMap.put(nodeKey, 0L);
        // 虚拟节点数量
        for (int i = 0; i < repetition; i++) { //非常重要
            for (Long hashedKey : ketamaNodePositionsAtIteration(nodeKey, i)) {
                virtualNodeMap.put(hashedKey, 0L);
                mappingMap.put(hashedKey, nodeKey);
            }
        }
    }
````
测试结果如下：
````
192.168.199.203:18000-19.4207-194207
192.168.199.201:18000-20.7831-207831
192.168.199.200:18000-19.7188-197188
192.168.199.202:18000-19.6237-196237
192.168.199.204:18000-20.4537-204537
````
