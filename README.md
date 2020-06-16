#  SnowFlake 雪花算法


对于分布式系统环境，主键ID的设计很关键，什么自增intID那些是绝对不用的，比较早的时候，**大部分系统都用UUID/GUID来作为主键**，**优点**是方便又能解决问题，**缺点**是插入时因为UUID/GUID的不规则导致每插入一条数据就需要重新排列一次，性能低下；也有人提出用UUID/GUID转long的方式，可以很明确的告诉你，这种方式long不能保证唯一，大并发下会有重复long出现，所以也不可取，这个主键设计问题曾经是很多公司系统设计的一个头疼点，所以大部分公司愿意牺牲一部分性能而直接采用简单粗暴的UUID/GUID来作为分布式系统的主键；

　　twitter开源了一个**snowflake算法，俗称雪花算法**；就是为了解决分布式环境下生成不同ID的问题；该算法会生成19位的long型有序数字，MySQL中用bigint来存储（bigint长度为20位）；该算法应该是目前分布式环境中主键ID最好的解决方案之一了；

SnowFlake 算法其核心思想就是：使用一个 64 bit 的 long 型的数字作为全局唯一 id。在分布式系统中的应用十分广泛，且ID 引入了时间戳，基本上保持自增的，后面的代码中有详细的注解。

这 64 个 bit 中，其中 1 个 bit 是不用的，然后用其中的 41 bit 作为毫秒数，用 10 bit 作为工作机器 id，12 bit 作为序列号。

![在这里插入图片描述](https://img-blog.csdnimg.cn/2020061610430219.png)

给大家举个例子吧，比如下面那个 64 bit 的 long 型数字：

- 第一个部分，是 1 个 bit：0，这个是无意义的。

- 第二个部分是 41 个 bit：表示的是时间戳。

- 第三个部分是 5 个 bit：表示的是机房 id，10001。

- 第四个部分是 5 个 bit：表示的是机器 id，1 1001。

- 第五个部分是 12 个 bit：表示的序号，就是某个机房某台机器上这一毫秒内同时生成的 id 的序号，0000 00000000。

 

①1 bit：是不用的，为啥呢？

> 因为二进制里第一个 bit 为如果是 1，那么都是负数，但是我们生成的 id 都是正数，所以第一个 bit 统一都是 0。

 

②41 bit：表示的是时间戳，单位是毫秒。 

> 41 bit 可以表示的数字多达 2^41 - 1，也就是可以标识 2 ^ 41 - 1 个毫秒值，换算成年就是表示 69 年的时间。

 

③10 bit：记录工作机器 id，代表的是这个服务最多可以部署在 2^10 台机器上，也就是 1024 台机器。 

> 但是 10 bit 里 5 个 bit 代表机房 id，5 个 bit 代表机器 id。意思就是最多代表 2 ^ 5 个机房（32 个机房），每个机房里可以代表 2 ^ 5 个机器（32 台机器），也可以根据自己公司的实际情况确定。

 

④12 bit：这个是用来记录同一个毫秒内产生的不同 id。

> 12 bit 可以代表的最大正整数是 2 ^ 12 - 1 = 4096，也就是说可以用这个 12 bit 代表的数字来区分同一个毫秒内的 4096 个不同的 id。

	简单来说，你的某个服务假设要生成一个全局唯一 id，那么就可以发送一个请求给部署了 SnowFlake 算法的系统，由这个 SnowFlake 算法系统来生成唯一 id。

这个 SnowFlake 算法系统首先肯定是知道自己所在的机房和机器的，比如机房 id = 17，机器 id = 12。

接着 SnowFlake 算法系统接收到这个请求之后，首先就会用二进制位运算的方式生成一个 64 bit 的 long 型 id，64 个 bit 中的第一个 bit 是无意义的。

接着 41 个 bit，就可以用当前时间戳（单位到毫秒），然后接着 5 个 bit 设置上这个机房 id，还有 5 个 bit 设置上机器 id。

最后再判断一下，当前这台机房的这台机器上这一毫秒内，这是第几个请求，给这次生成 id 的请求累加一个序号，作为最后的 12 个 bit。

最终一个 64 个 bit 的 id 就出来了，类似于：

这个算法可以保证说，一个机房的一台机器上，在同一毫秒内，生成了一个唯一的 id。可能一个毫秒内会生成多个 id，但是有最后 12 个 bit 的序号来区分开来。

下面我们简单看看这个 SnowFlake 算法的一个代码实现，这就是个示例，大家如果理解了这个意思之后，以后可以自己尝试改造这个算法。



总之就是用一个 64 bit 的数字中各个 bit 位来设置不同的标志位，区分每一个 id。

 

SnowFlake 算法的实现代码如下：

```java
/** 描述: Twitter的分布式自增ID雪花算法snowflake (Java版) */
public class SnowFlake {

  // 因为二进制里第一个 bit 为如果是 1，那么都是负数，但是我们生成的 id 都是正数，所以第一个 bit 统一都是 0

  /** 起始的时间戳 */
  private static final long START_STMP = 1480166465631L;

  /** 每一部分占用的位数 */
  private static final long SEQUENCE_BIT = 12; // 序列号占用的位数

  private static final long MACHINE_BIT = 5; // 机器标识占用的位数
  private static final long DATACENTER_BIT = 5; // 数据中心占用的位数

  /** 每一部分的最大值 */
  // 这个是一个意思，就是5 bit最多只能有31个数字，机房id最多只能是32以内
  private static final long MAX_DATACENTER_NUM = -1L ^ (-1L << DATACENTER_BIT);
  // 这个是二进制运算，就是5 bit最多只能有31个数字，也就是说机器id最多只能是32以内
  private static final long MAX_MACHINE_NUM = -1L ^ (-1L << MACHINE_BIT);
  // 每毫秒内产生的id数 2 的 12次方
  private static final long MAX_SEQUENCE = -1L ^ (-1L << SEQUENCE_BIT);

  /** 每一部分向左的位移 */
  private static final long MACHINE_LEFT = SEQUENCE_BIT;

  private static final long DATACENTER_LEFT = SEQUENCE_BIT + MACHINE_BIT;
  private static final long TIMESTMP_LEFT = DATACENTER_LEFT + DATACENTER_BIT;

  // 机房ID 2进制5位  32位减掉1位 31个
  private long datacenterId; // 数据中心、机房ID
  // 机器ID  2进制5位  32位减掉1位 31个
  private long machineId; // 机器标识
  // 代表一毫秒内生成的多个id的最新序号  12位 4096 -1 = 4095 个
  private long sequence = 0L; // 序列号
  // 记录产生时间毫秒数，判断是否是同1毫秒
  private long lastStmp = -1L; // 上一次时间戳

  public SnowFlake(long datacenterId, long machineId) {
    // 检查机房id和机器id是否超过31 不能小于0
    if (datacenterId > MAX_DATACENTER_NUM || datacenterId < 0) {
      throw new IllegalArgumentException(
          "datacenterId can't be greater than MAX_DATACENTER_NUM or less than 0");
    }
    if (machineId > MAX_MACHINE_NUM || machineId < 0) {
      throw new IllegalArgumentException(
          "machineId can't be greater than MAX_MACHINE_NUM or less than 0");
    }
    this.datacenterId = datacenterId;
    this.machineId = machineId;
  }

  /**
   * // 这个是核心方法，通过调用nextId()方法，让当前这台机器上的snowflake算法程序生成一个全局唯一的id
   *
   * @return
   */
  public synchronized long nextId() {
    long currStmp = getNewstmp();
    if (currStmp < lastStmp) {
      throw new RuntimeException("Clock moved backwards.  Refusing to generate id");
    }

    // 下面是说假设在同一个毫秒内，又发送了一个请求生成一个id
    // 这个时候就得把seqence序号给递增1，最多就是4096
    if (currStmp == lastStmp) {
      // 这个意思是说一个毫秒内最多只能有4096个数字，无论你传递多少进来，
      // 这个位运算保证始终就是在4096这个范围内，避免你自己传递个sequence超过了4096这个范围
      sequence = (sequence + 1) & MAX_SEQUENCE;
      // 当某一毫秒的时间，产生的id数 超过4095，系统会进入等待，直到下一毫秒，系统继续产生ID
      if (sequence == 0L) {
        currStmp = getNextMill();
      }
    } else {
      // 不同毫秒内，序列号置为0
      sequence = 0L;
    }

    lastStmp = currStmp;
    // 最核心的二进制位运算操作，生成一个64bit的id
    // 先将当前时间戳左移，放到41 bit那儿；将机房id左移放到5 bit那儿；将机器id左移放到5 bit那儿；将序号放最后12 bit
    // 最后拼接起来成一个64 bit的二进制数字，转换成10进制就是个long型
    return (currStmp - START_STMP) << TIMESTMP_LEFT // 时间戳部分
        | datacenterId << DATACENTER_LEFT // 数据中心部分
        | machineId << MACHINE_LEFT // 机器标识部分
        | sequence; // 序列号部分
  }

  private long getNextMill() {
    long mill = getNewstmp();
    while (mill <= lastStmp) {
      mill = getNewstmp();
    }
    return mill;
  }

  private long getNewstmp() {
    return System.currentTimeMillis();
  }

  public static void main(String[] args) {
    SnowFlake snowFlake = new SnowFlake(5, 9);

    long start = System.currentTimeMillis();
    for (int i = 0; i < 1000000; i++) {
      System.out.println(snowFlake.nextId());
    }

    System.out.println(System.currentTimeMillis() - start);

    System.out.println("MAX_MACHINE_NUM");
    System.out.println(MAX_MACHINE_NUM);
    System.out.println(MAX_DATACENTER_NUM);
    System.out.println(MAX_SEQUENCE);
    System.out.println("MAX_DATACENTER_NUM");
  }
}

```



## SnowFlake算法的优点：

- （1）高性能高可用：生成时不依赖于数据库，完全在内存中生成（由于完基于位运算，所以性能比随机数运算要高）。

- （2）容量大：每秒中能生成数百万的自增ID。

- （3）ID自增：存入数据库中，索引效率高。

 

## SnowFlake算法的缺点：


依赖与系统时间的一致性，如果系统时间被回调，或者改变，可能会造成id冲突或者重复。

实际中我们的机房并没有那么多，我们可以改进改算法，将10bit的机器id优化，成业务表或者和我们系统相关的业务。

## 测试生成100万个数据算只用了约4s时间
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200616103850892.png)