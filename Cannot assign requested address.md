## 现象
使用redigo访问redis，压测时当并发达到1500以上时，会报“Cannot assign requested address”错误。
## 问题追查
- 网上查到说是由于TCP端口被占满导致的，原来当redis连接关闭后，TCP端口并没有马上释放，而是处于TIME_WAIT状态，需要过50s左右才释放。当并发较大时，端口被占满就会导致没有可用的连接
**解决方案：**
执行命令修改如下2个内核参数  
```shell
sysctl -w net.ipv4.tcp_timestamps=1  开启对于TCP时间戳的支持,若该项设置为0，则下面一项设置不起作用
sysctl -w net.ipv4.tcp_tw_recycle=1  表示开启TCP连接中TIME-WAIT sockets的快速回收
```

- 但设置后发现仍会出现上述错误，怀疑是redis的最大连接数设置的比较低，通过下述命令查询，发现最大连接数是10240，排除
```shell
redis > config get maxclients
```

- 由于使用的是redigo的连接池，怀疑设置有问题，因为如果是连接池的话，应该会重用连接，而不是创建那么多新连接，由于初始未设定MaxAcitve参数，即不限制最大的活跃连接，然后设置`MaxActive:10000`，仍失败；查看源码发现当调用conn.close时，会将connection放入连接池中，但如果idle的连接大于MaxIdel，则会释放该连接，但我MaxIdle设置的是3，所以导致大部分连接都被释放，从而无法重用。设置MaxIdle=1000后问题解决，相关代码如下：
```go
redis.Pool{
		MaxIdle:     1000,
		IdleTimeout: 600 * time.Second,
		Dial: func() (redis.Conn, error) {
			c, err := redis.Dial("tcp", server)
			if err != nil {
				return nil, err
			}
			if password != "" {
				if _, err := c.Do("AUTH", password); err != nil {
					c.Close()
					return nil, err
				}
			}
			return c, err
		},
		TestOnBorrow: func(c redis.Conn, t time.Time) error {
			_, err := c.Do("PING")
			return err
		},
	}
```
