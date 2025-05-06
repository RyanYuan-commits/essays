```bash
docker run -it \
-v ~/Documents/volumes/redis_conf/redis_master/conf:/usr/local/etc/redis \
--name redis-master \
-p 6379:6379 \
-d redis \
redis-server /usr/local/etc/redis/redis.conf
```