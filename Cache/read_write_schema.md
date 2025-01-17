# 读写策略

我们有三个实体：App Cache DB

## Cache Aside

以数据库中的数据为准，缓存中的数据按需加载

```
// Write
App write to DB;
    Get response from DB
App invalidate Cache
    invalidate response

// Read
App read from Cache
    if HIT:
        Get value from Cache
    else:
        App read from DB
            Get value from DB
        App write to Cache
            write response
```

* 不严格保证一致性
* 频繁的写入会导致命中率下降

## Read/Write Through

缓存机制对使用者透明

```
// Write
App look up Cache
    if HIT:
        App update Cache
            update response
                   Cache update DB(maybe async)
                         update response
    else:
        App update DB
            update response

// Read
App read from Cache
    if HIT:
        read response
    else:
        Cache look up DB
            Get response from DB
            respond to App
```

* 常见的客户端只读缓存策略，对使用者透明，指定过期、刷新时间
* 不会造成脏独，缓存命中率更高，一致性保障难度大

## Write Back

```
App write to Cache
    write response
             
             Cache sync data with DB (periodically or triggered)
```

* 写入性能大幅提升，一致性保障难度更大
