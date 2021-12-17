## 分布式ID的生成

```python
一：需要满足条件：
    1 分布式的系统中，生成id号，按当前时间生成id号，可能会重复，全局唯一ID的系统是非常必要的
    2 不同机器上生成的id号不能重复


3 生成id号的要求
	-全局唯一
    -趋势递增
    -单调递增
    -安全（不要被猜出来）
二：方案
    1 mysql的自增（可以做，但是性能跟不上）
    2 uuid：全局唯一，安全，没有递增的趋势
    
    3 redis：自增+时间+用户id  比较好的方案

	4 分布式系统最常用的生成全局唯一ID的方法是雪花算法snowFlake
    雪花算法（业界普遍选择的id生成方案）跟语言无关（任何语言都有实现），不依赖于服务（redis，mysql）
    
    介绍：
    雪花算法生成的结果是一位64bit的整数，为一个long类型（转换成字符串最多为19位）
    1bit: 一般是符号位，代表正负数的所以这一位不做处理
	41bit：这个部分用来记录时间戳，如果从1970-01-01 00:00:00来计算开始时间的话，它可以记录到2039年，足够我们用了，并且后续我们可以设置起始时间，这样就不用担心不够的问题， 这一个部分是保证我们生辰的id趋势递增的关键。
	10bit：这是用来记录机器id的， 默认情况下这10bit会分成两部分前5bit代表数据中心，后5bit代表某个数据中心的机器id，默认情况下计算大概可以支持32*32 - 1= 1023台机器。
	12bit：循环位，来对应1毫秒内产生的不同的id， 大概可以满足1毫秒并发生成2^12-1=4095次id的要求。也就是在同一个机器同一毫秒最多记录4095个，多余的需要进行等待下毫秒。


上面只是一个将64bit划分的标准，当然也不一定这么做，可以根据不同业务的具体场景来划分，比如下面给出一个业务场景：
    服务目前QPS10万，预计几年之内会发展到百万。
    当前机器三地部署，上海，北京，深圳都有。
    当前机器10台左右，预计未来会增加至百台。
    这个时候我们根据上面的场景可以再次合理的划分62bit,QPS几年之内会发展到百万，那么每毫秒就是千级的请求，目前10台机器那么每台机器承担百级的请求，为了保证扩展，后面的循环位可以限制到1024，也就是2^10，那么循环位10位就足够了。
    机器三地部署我们可以用3bit总共8来表示机房位置，当前的机器10台，为了保证扩展到百台那么可以用7bit 128来表示，时间位依然是41bit,那么还剩下64-10-3-7-41-1 = 2bit,还剩下2bit可以用来进行扩展。


由上可见用雪花算法生成的ID既是全局唯一的，又是递增的，又不需要任何维护成本，所以雪花算法相比之下是最好生成全局唯一ID的方法。

```


```python
雪花算法python，
缺点：时钟回拨：因为机器的原因会发生时间回拨，我们的雪花算法是强依赖我们的时间的，如果时间发生回拨，有可能会生成重复的ID，在我们上面的nextId中我们用当前时间和上一次的时间进行判断，如果当前时间小于上一次的时间那么肯定是发生了回拨，算法会直接抛出异常.

import time
import logging

# 64位ID的划分
WORKER_ID_BITS = 5
DATACENTER_ID_BITS = 5
SEQUENCE_BITS = 12
# 最大取值计算
MAX_WORKER_ID = -1 ^ (-1 << WORKER_ID_BITS)  # 2**5-1 0b11111
MAX_DATACENTER_ID = -1 ^ (-1 << DATACENTER_ID_BITS)
# 移位偏移计算
WOKER_ID_SHIFT = SEQUENCE_BITS
DATACENTER_ID_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS
TIMESTAMP_LEFT_SHIFT = SEQUENCE_BITS + WORKER_ID_BITS + DATACENTER_ID_BITS
# 序号循环掩码
SEQUENCE_MASK = -1 ^ (-1 << SEQUENCE_BITS)
# Twitter元年时间戳
TWEPOCH = 1288834974657
logger = logging.getLogger('flask.app')


class IdWorker(object):
    """
    用于生成IDs
    """
    def __init__(self, datacenter_id, worker_id, sequence=0):
        """
        初始化
        :param datacenter_id: 数据中心（机器区域）ID
        :param worker_id: 机器ID
        :param sequence: 其实序号
        """
        # sanity check
        if worker_id > MAX_WORKER_ID or worker_id < 0:
            raise ValueError('worker_id值越界')

        if datacenter_id > MAX_DATACENTER_ID or datacenter_id < 0:
            raise ValueError('datacenter_id值越界')

        self.worker_id = worker_id
        self.datacenter_id = datacenter_id
        self.sequence = sequence

        self.last_timestamp = -1  # 上次计算的时间戳

    def _gen_timestamp(self):
        """
        生成整数时间戳
        :return:int timestamp
        """
        return int(time.time() * 1000)

    def get_id(self):
        """
        获取新ID
        :return:
        """
        timestamp = self._gen_timestamp()

        # 时钟回拨
        if timestamp < self.last_timestamp:
            logging.error('clock is moving backwards. Rejecting requests until{}'.format(self.last_timestamp))
            raise Exception

        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & SEQUENCE_MASK
            if self.sequence == 0:
                timestamp = self._til_next_millis(self.last_timestamp)
        else:
            self.sequence = 0

        self.last_timestamp = timestamp

        new_id = ((timestamp - TWEPOCH) << TIMESTAMP_LEFT_SHIFT) | (self.datacenter_id << DATACENTER_ID_SHIFT) | \
                 (self.worker_id << WOKER_ID_SHIFT) | self.sequence
        return new_id

    def _til_next_millis(self, last_timestamp):
        """
        等到下一毫秒
        """
        timestamp = self._gen_timestamp()
        while timestamp <= last_timestamp:
            timestamp = self._gen_timestamp()
        return timestamp

def test():
    for i in range(10):
        id = worker.get_id()
        print(id)

if __name__ == '__main__':
    from threading import Thread
    worker = IdWorker(1, 1, 0)
    l = list()
    for i in range(2):
        t = Thread(target=test)
        t.start()
        l.append(t)
    for t in l:
        t.join()
```