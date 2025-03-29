```
docker run \
-p 3306:3306 \
--restart=always \
--name mysql \
--privileged=true \
-e MYSQL_ROOT_PASSWORD=my-secret-pw \
-d mysql:8.3.0
```

```
docker run \
-p 3306:3306 \
--restart=always \
--name mysql \
--privileged=true \
-v /home/mysql/log:/var/log/mysql \
-v /home/mysql/data:/var/lib/mysql \
-v /home/mysql/conf/my.cnf:/etc/mysql/my.cnf \
-e MYSQL_ROOT_PASSWORD=my-secret-pw \
-d mysql:8.3.0
```