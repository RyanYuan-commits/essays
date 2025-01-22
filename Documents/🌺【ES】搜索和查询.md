### 使用 Docker 安装 ES
```bash
docker network create elastic
docker pull docker.elastic.co/elasticsearch/elasticsearch:7.15.2
docker run -di --name es 
--net elastic
-p 10.37.112.12:9200:9200 
-p 10.37.112.12:9300:9300 
-e "discovery.type=single-node"
docker.elastic.co/elasticsearch/elasticsearch:7.15.2
```

