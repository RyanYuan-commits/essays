```bash
docker run -it \
-v ~/Documents/volumes/myredis/conf:/usr/local/etc/redis \
-v ~/Documents/volumes/myredis/data:/data \
--name myredis \
-p 6379:6379 \
-d redis \
redis-server /usr/local/etc/redis/redis.conf
```