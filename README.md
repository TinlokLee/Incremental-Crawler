# Incremental-Crawler

1. 应用场景
    页面抓取量大时：通过并行，分布式，集群提高抓取速度；
                   增量式爬取提高系统的工作效率，降低算法复杂度

2. 增量式爬虫：只抓取上次出现之后页面更新的部分，另一种原本存在网页内容更新；
              不论是产生新页面，还是原页面更新都是增量

3. 爬虫的工作流程：发送URL请求 -- 获取响应 -- 解析内容 -- 存储内容

4. 增量爬虫思路：
            在发送请求之前判断这个URL是不是之前爬取过
            在解析内容后判断这部分内容是不是之前爬取过
            写入存储介质时判断内容是不是已经在介质中存在

5. 实现增量爬虫：核心去重
   去重方法：数据指纹(给URL或内容做一个标识) -- hash值，作为数据库索引
            发送请求设置304状态码，last-modified字段，文件大小，MD5签名等

            数据量小时，函数或set()去重

            数据指纹+Bloomfilter 节省空间 + Redis数据库持久化存储
            布朗过滤器可以通过计算来判断某项数据是否存在于集合中，
            它是一个典型且高效的空间换时间的例子

6. Redis的集合就是Redis数据库中的集合类型，它具有无序不重复的特点
   关于Redis数据库几个关键词：key-value，高性能，数据持久化，数据备份，原子操作以及支持集合数据类型
   Redis可以将内存中的内容持久化到磁盘，并且其每一次操作都是原子操作，这就保证了爬虫的可靠性，即爬虫不会应为意外停止而损失数据
   常用操作命令两个：SADD 和 SISMEMBER

>>> import redis
>>> r = redis.StrictRedis(host='192.168.153.131', port=6379, db=0)
>>> r.sadd('1','aa')
1
>>> r.sismember('1','aa')
True


00. 如何将这一特性融入到爬虫中?

如果是自己写的爬虫代码，添加上述代码即可；

如果使用的是scrapy框架，我们可以在middleware上下功夫
在spider模块收到要处理的URL时，写一个Spider中间件用来判断这条URL的指纹是否在Redis数据库中存在,如果存在的话，就直接舍弃这条URL；
如果是要判断页面的内容是否更新，可以在Download中间件中添加代码来校验，原理一样。
数据库的操作可以用类似write()和query()的方法进行封装


7. BloomFilter：使用pybloom库，需要pip或者源码安装
from pybloom import BloomFilter

# 新建一个过滤器，长度为m，错误率为0.1%
bf = BloomFilter(capacity=1000, error_rate=0.001)
[bf.add(x) for x in range(bf.capacity)]
print(0 in bf, 5 in bf, 10 in bf)

# 计算错误率
count = 0
amount = bf.capacity
for i in range(bf.capacity, bf.capacity + amount + 1):
    if i in bf:
        coiunt -= 1
print('FP: {: 2.4f}'.format(count / float(amount)))

8. Filter 使用
    
# 新建
bf1 = BloomFilter(capacity=1000, error_rate=0.001)
bf2 = BloomFilter(capacity=1000, error_rate=0.001)

# 添加
[bf1.add(x) for x in range(3)]
[bf2.add(x) for x in range(3,6)]

# 复制
bf3 = bf1.copy()

# | 操作，三种都行
bf3.union(bf1)
bf3 = bf3 | bf2
bf3 = bf3 or bf2

# & 操作, 三种都行
bf3.intersection(bf1)
bf3 = bf3 & bf1
bf3 = bf3 and bf1

# 成员变量和支持的操作符
len(bf3)
3 in bf3
bf3.capacity
bf3.error_rate

可扩展的过滤器

# 两个属性（装饰器）
bf1.capacity
bf1.count

# 新建, mode目前只有2种
# SMALL_SET_GROWTH = 2, LARGE_SET_GROWTH = 4
# 前者占内存少但速度慢，后者消耗内存快但速度快
