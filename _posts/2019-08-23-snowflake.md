---
layout: post
title:  "雪花算法"
date:   2019-08-23 10:50:01 +0800
categories: snowflake
tag: snowflake
---

* content
{:toc}


雪花算法优缺点		{#top}
==================

UUID（缺点：太长、没法排序、使数据库性能降低）<br>
Redis（缺点：必须依赖Redis）<br>
Oracle序列号（缺点：用Oracle才能使用）<br>
Snowflake雪花算法，优点：生成有顺序的id，提高数据库的性能<br>

Snowflake雪花算法解析		{#Algorithmic_Analysis}
===================

![/styles/images/common/snowflake.png]({{ '/styles/images/common/snowflake.png' | prepend: site.baseurl  }})

雪花算法解析 结构 snowflake的结构如下(每部分用-分开):<br>
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000<br>
第一位为未使用，接下来的41位为毫秒级时间(41位的长度可以使用69年)，<br>
然后是5位datacenterId和5位workerId(10位的长度最多支持部署1024个节点），<br>
最后12位是毫秒内的计数（12位的计数顺序号支持每个节点每毫秒产生4096个ID序号）<br>
一共加起来刚好64位，为一个Long型。(转换成字符串长度为18)。<br>
Snowflake算法核心把时间戳，工作机器id，序列号组合在一起。<br>
整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞（由datacenter和机器ID作区分），<br>
并且效率较高，经测试，snowflake每秒能够产生26万ID左右，完全满足需要。<br>

依赖包			{#Dependency_Packages}
===================

&nbsp;\<dependency\><br>
&nbsp;&nbsp;&nbsp;&nbsp;\<groupId\>org.apache.commons\</groupId\><br>
&nbsp;&nbsp;&nbsp;&nbsp;\<artifactId\>commons-lang3\</artifactId\><br>
&nbsp;&nbsp;&nbsp;&nbsp;\<version\>3.8\</version\><br>
&nbsp;\</dependency\><br>

实现方式-Java			{#Implementation}
===================

/**<br>
&nbsp;* Twitter_Snowflake<br>
&nbsp;* SnowFlake的结构如下(每部分用-分开):<br>
&nbsp;* 0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000 <br>
&nbsp;* 1位标识，由于long基本类型在Java中是带符号的，最高位是符号位，正数是0，负数是1，所以id一般是正数，最高位是0<br>
&nbsp;* 41位时间截(毫秒级)，注意，41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截)
&nbsp;* 得到的值），这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下下面程序IdWorker类的startTime属性）。41位的时间截，可以使用69年，年T = (1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69<br>
&nbsp;* 10位的数据机器位，可以部署在1024个节点，包括5位datacenterId和5位workerId<br>
&nbsp;* 12位序列，毫秒内的计数，12位的计数顺序号支持每个节点每毫秒(同一机器，同一时间截)产生4096个ID序号<br>
&nbsp;* 加起来刚好64位，为一个Long型。<br>
&nbsp;* SnowFlake的优点是，整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞(由数据中心ID和机器ID作区分)，并且效率较高，经测试，SnowFlake每秒能够产生26万ID左右。<br>
&nbsp;*/<br>
class SnowflakeIdWorker {<br>
&nbsp;&nbsp;&nbsp;&nbsp;/** 开始时间截 (2015-01-01) */<br>
&nbsp;&nbsp;&nbsp;&nbsp;private final long twepoch = 1489111610226L;<br>

&nbsp;&nbsp;&nbsp;&nbsp;/** 机器id所占的位数 */<br>
&nbsp;&nbsp;&nbsp;&nbsp;private final long workerIdBits = 5L;<br>

&nbsp;&nbsp;&nbsp;&nbsp;/** 数据标识id所占的位数 */<br>
&nbsp;&nbsp;&nbsp;&nbsp;private final long dataCenterIdBits = 5L;<br>

&nbsp;&nbsp;&nbsp;&nbsp;/** 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数) */<br>
&nbsp;&nbsp;&nbsp;&nbsp;private final long maxWorkerId = -1L ^ (-1L << workerIdBits);<br>

&nbsp;&nbsp;&nbsp;&nbsp;/** 支持的最大数据标识id，结果是31 */<br>
&nbsp;&nbsp;&nbsp;&nbsp;private final long maxDataCenterId = -1L ^ (-1L << dataCenterIdBits);<br>

&nbsp;&nbsp;&nbsp;&nbsp;/** 序列在id中占的位数 */<br>
&nbsp;&nbsp;&nbsp;&nbsp;private final long sequenceBits = 12L;<br>

&nbsp;&nbsp;&nbsp;&nbsp;/** 机器ID向左移12位 */<br>
&nbsp;&nbsp;&nbsp;&nbsp;private final long workerIdShift = sequenceBits;<br>

    /** 数据标识id向左移17位(12+5) */<br>
    private final long dataCenterIdShift = sequenceBits + workerIdBits;<br>

    /** 时间截向左移22位(5+5+12) */<br>
    private final long timestampLeftShift = sequenceBits + workerIdBits + dataCenterIdBits;<br>

    /** 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095) */<br>
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);<br>

    /** 工作机器ID(0~31) */<br>
    private long workerId;<br>

    /** 数据中心ID(0~31) */<br>
    private long dataCenterId;<br>

    /** 毫秒内序列(0~4095) */<br>
    private long sequence = 0L;<br>

    /** 上次生成ID的时间截 */<br>
    private long lastTimestamp = -1L;<br>

    private static SnowflakeIdWorker idWorker;<br>

    static {<br>
        idWorker = new SnowflakeIdWorker(getWorkId(),getDataCenterId());<br>
    }<br>

    //=============Constructors==============<br>
    /**<br>
     * 构造函数<br>
     * @param workerId 工作ID (0~31)<br>
     * @param dataCenterId 数据中心ID (0~31)<br>
     */<br>
    public SnowflakeIdWorker(long workerId, long dataCenterId) {<br>
        if (workerId > maxWorkerId || workerId < 0) {<br>
            throw new IllegalArgumentException(String.format("workerId can't be greater than %d or less than 0", maxWorkerId));<br>
        }<br>
        if (dataCenterId > maxDataCenterId || dataCenterId < 0) {<br>
            throw new IllegalArgumentException(String.format("dataCenterId can't be greater than %d or less than 0", maxDataCenterId));<br>
        }<br>
        this.workerId = workerId;<br>
        this.dataCenterId = dataCenterId;<br>
    }<br>

    // =========Methods=========<br>
    /**<br>
     * 获得下一个ID (该方法是线程安全的)<br>
     * @return SnowflakeId<br>
     */<br>
    public synchronized long nextId() {<br>
        long timestamp = timeGen();<br>

        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常<br>
        if (timestamp < lastTimestamp) {<br>
            throw new RuntimeException(<br>
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));<br>
        }<br>

        //如果是同一时间生成的，则进行毫秒内序列<br>
        if (lastTimestamp == timestamp) {<br>
            sequence = (sequence + 1) & sequenceMask;<br>
            //毫秒内序列溢出<br>
            if (sequence == 0) {<br>
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);<br>
            }<br>
        }<br>
        //时间戳改变，毫秒内序列重置<br>
        else {<br>
            sequence = 0L;<br>
        }<br>

        //上次生成ID的时间截<br>
        lastTimestamp = timestamp;<br>

        //移位并通过或运算拼到一起组成64位的ID<br>
        return ((timestamp - twepoch) << timestampLeftShift)<br>
                | (dataCenterId << dataCenterIdShift)<br>
                | (workerId << workerIdShift)<br>
                | sequence;<br>
    }<br>

    /**<br>
     * 阻塞到下一个毫秒，直到获得新的时间戳<br>
     * @param lastTimestamp 上次生成ID的时间截<br>
     * @return 当前时间戳<br>
     */<br>
    protected long tilNextMillis(long lastTimestamp) {<br>
        long timestamp = timeGen();<br>
        while (timestamp <= lastTimestamp) {<br>
            timestamp = timeGen();<br>
        }<br>
        return timestamp;<br>
    }<br>

    /**<br>
     * 返回以毫秒为单位的当前时间<br>
     * @return 当前时间(毫秒)<br>
     */<br>
    protected long timeGen() {<br>
        return System.currentTimeMillis();<br>
    }<br>

    private static Long getWorkId(){<br>
        try {<br>
            String hostAddress = Inet4Address.getLocalHost().getHostAddress();<br>
            int[] ints = StringUtils.toCodePoints(hostAddress);<br>
            int sums = 0;<br>
            for(int b : ints){<br>
                sums += b;<br>
            }<br>
            return (long)(sums % 32);<br>
        } catch (UnknownHostException e) {<br>
            // 如果获取失败，则使用随机数备用<br>
            return RandomUtils.nextLong(0,31);<br>
        }<br>
    }<br>

    private static Long getDataCenterId(){<br>
        int[] ints = StringUtils.toCodePoints(SystemUtils.getHostName());<br>
        int sums = 0;<br>
        for (int i: ints) {<br>
            sums += i;<br>
        }<br>
        return (long)(sums % 32);<br>
    }<br>


    /**<br>
     * 静态工具类<br>
     *<br>
     * @return<br>
     */<br>
    public static Long generateId(){<br>
        long id = idWorker.nextId();<br>
        return id;<br>
    }<br>

    //==========Test=========<br>
    /** 测试 */<br>
    public static void main(String[] args) {<br>
        System.out.println(System.currentTimeMillis());<br>
        long startTime = System.nanoTime();<br>
        for (int i = 0; i < 50000; i++) {<br>
            long id = SnowflakeIdWorker.generateId();<br>
            System.out.println(id);<br>
        }<br>
        System.out.println((System.nanoTime()-startTime)/1000000+"ms");<br>
    }<br>
}<br>