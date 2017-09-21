# 内置的Redis支持

Core包内建对Redis的支持。使用redis，需要在项目的配置目录的```core/redis.neon```中配置连接参数：

```neon
default:
  connect:          # can use 'pconnect' as key
    - 127.0.0.1     # can be unix socket path
    - 6379          # port, ignore when use unix socket
    # - 1           # timeout
    # - null        # reserved, value in seconds
    # - 100         # retry_interval, value in milliseconds
    # - 0           # read_timeout, value in seconds
```

使用的时候：

```php
$redis = rt::core()->redis();
```

上面默认取出是```default```实例，如果配置文件中配置了多个连接实例，可以给参数取出，比如定义了一个叫```cache``的连接：

```php
$redis = rt::core()->redis('cache');
```

返回的```$redis```是原生phpredis扩展的连接对象，请参考[相关文档](https://github.com/phpredis/phpredis)使用。

