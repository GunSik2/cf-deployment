


### Resolving bosh lock ###
BOSH uses Redis for locks. To find the location and password for Redis, look in the Director's configuration:
```
$ cat /var/vcap/jobs/director/config/director.yml | grep redis -A3
redis:  
  host: 127.0.0.1
  port: 25255
  password: redis
  logging:
    level: info
```

To connected to Redis, add the redis-cli to the $PATH:
```
export PATH=$PATH:/var/vcap/packages/redis/bin  
redis-cli -p 25255 -a redis  
```

To find the lock, look up all current locks:
```
> keys "lock:*"
1) "lo
```

Delete it with Redis command del:
```
> del lock:deployment:my-locked-deployment
(integer) 1
```

Reference: https://blog.starkandwayne.com/2015/06/08/unlocking-bosh-locks/
