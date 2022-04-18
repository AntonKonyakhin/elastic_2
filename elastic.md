###Задача 1
В этом задании вы потренируетесь в:

- установке elasticsearch
- первоначальном конфигурировании elastcisearch
- запуске elasticsearch в docker

Используя докер образ centos:7 как базовый и документацию по установке и запуску Elastcisearch:

- составьте Dockerfile-манифест для elasticsearch
- соберите docker-образ и сделайте push в ваш docker.io репозиторий
- запустите контейнер из получившегося образа и выполните запрос пути / c хост-машины

Требования к elasticsearch.yml:

- данные path должны сохраняться в /var/lib
- имя ноды должно быть netology_test


В ответе приведите:

- текст Dockerfile манифеста
- ссылку на образ в репозитории dockerhub
- ответ elasticsearch на запрос пути / в json виде

Dockerfile:
```
from centos:centos7
MAINTAINER konyakhin anton
ENV https_proxy="http://***:****"
ENV http_proxy="http://***:****"
RUN ["/bin/bash", "-c" , "yum update -y && yum install java-11-openjdk -y && yum install wget -y && yum install perl-Digest-SHA -y"]
RUN ["/bin/bash", "-c", "wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.1.2-linux-x86_64.tar.gz &&\
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-8.1.2-linux-x86_64.tar.gz.sha512 &&\
shasum -a 512 -c elasticsearch-8.1.2-linux-x86_64.tar.gz.sha512 &&\
tar -xzf elasticsearch-8.1.2-linux-x86_64.tar.gz"]
RUN ["/bin/bash", "-c", "rm -rf elasticsearch-8.1.2-linux-x86_64.tar.gz"]
ENV http_proxy=
ENV https_proxy=
RUN mkdir -p /var/lib/elasticsearch &&\
mkdir -p /var/lib/elasticsearch_logs
RUN groupadd elasticsearch && useradd elasticsearch -g elasticsearch -p elasticsearch &&\
chown -R elasticsearch:elasticsearch /elasticsearch-8.1.2 &&\
chmod o+x /elasticsearch-8.1.2/ &&\
chown -R elasticsearch:elasticsearch /var/lib/elasticsearch &&\
chown -R elasticsearch:elasticsearch /var/lib/elasticsearch_logs
COPY elasticsearch.yml /elasticsearch-8.1.2/config/
USER elasticsearch
CMD ["/elasticsearch-8.1.2/bin/elasticsearch"]

```

```
root@anton-v-m:~/elastic-docker# docker build -t test/elastic-dockerfile .
```

ссылка на образ в репозитории dockerhub
```
docker pull akonyakhin/elastic-dockerfile
```

ответ elasticsearch на запрос пути / в json виде:
```
docker run -d -p 9300:9200 test/elastic-dockerfile

root@anton-v-m:~# curl -X GET http://localhost:9300 -H "Accept: application/json"
{
  "name" : "netology_test",
  "cluster_name" : "elasticsearsh",
  "cluster_uuid" : "W_ZxHtE3TV2jXvjqkh3jTg",
  "version" : {
    "number" : "8.1.2",
    "build_flavor" : "default",
    "build_type" : "tar",
    "build_hash" : "31df9689e80bad366ac20176aa7f2371ea5eb4c1",
    "build_date" : "2022-03-29T21:18:59.991429448Z",
    "build_snapshot" : false,
    "lucene_version" : "9.0.0",
    "minimum_wire_compatibility_version" : "7.17.0",
    "minimum_index_compatibility_version" : "7.0.0"
  },
  "tagline" : "You Know, for Search"
}

```

### Задача 2
В этом задании вы научитесь:

- создавать и удалять индексы
- изучать состояние кластера
- обосновывать причину деградации доступности данных

Ознакомтесь с документацией и добавьте в elasticsearch 3 индекса, в соответствии со таблицей:

| Имя	| Количество реплик	| Количество шард|
|---|---|---|
|ind-1|	0	|1|
|ind-2	|1	|2|
|ind-3|	2|	4|

ind-1:
```
root@anton-v-m:~/elastic-docker# curl -X PUT "localhost:9300/ind-1?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 1,  
>       "number_of_replicas": 0 
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-1"
}

```
ind-2:
```
root@anton-v-m:~/elastic-docker# curl -X PUT "localhost:9300/ind-2?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 2,  
>       "number_of_replicas": 1 
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-2"
}

```
ind-3:
```
root@anton-v-m:~/elastic-docker# curl -X PUT "localhost:9300/ind-3?pretty" -H 'Content-Type: application/json' -d'
> {
>   "settings": {
>     "index": {
>       "number_of_shards": 4,  
>       "number_of_replicas": 2 
>     }
>   }
> }
> '
{
  "acknowledged" : true,
  "shards_acknowledged" : true,
  "index" : "ind-3"
}

```
Получите список индексов и их статусов, используя API и приведите в ответе на задание.

```
root@anton-v-m:~/elastic-docker# curl -X GET http://localhost:9300/_cat/indices?v
health status index uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   ind-1 4cDWMipYTISKseclUQJY1w   1   0          0            0       225b           225b
yellow open   ind-3 dYlNTx_hRyCRoHajoxl2nQ   4   2          0            0       413b           413b
yellow open   ind-2 BAP1Eyc9TOW8ZFHnlS3TwA   2   1          0            0       450b           450b


```

Получите состояние кластера elasticsearch, используя API.
```
root@anton-v-m:~/elastic-docker# curl -X GET http://localhost:9300/_cat/health?v
epoch      timestamp cluster       status node.total node.data shards pri relo init unassign pending_tasks max_task_wait_time active_shards_percent
1650228513 20:48:33  elasticsearsh yellow          1         1      7   7    0    0       10             0                  -                 41.2%

```

Как вы думаете, почему часть индексов и кластер находится в состоянии yellow?

```
из-за того, что у нас кластер из одного узла
```

Удалите все индексы.

```
root@anton-v-m:~/elastic-docker# curl -X DELETE "localhost:9300/ind-1?pretty"
{
  "acknowledged" : true
}

```

```
root@anton-v-m:~/elastic-docker# curl -X DELETE "localhost:9300/ind-2?pretty"
{
  "acknowledged" : true
}

```

```
root@anton-v-m:~/elastic-docker# curl -X DELETE "localhost:9300/ind-3?pretty"
{
  "acknowledged" : true
}

```

### Задача 3
В данном задании вы научитесь:

- создавать бэкапы данных
- восстанавливать индексы из бэкапов
- Создайте директорию {путь до корневой директории с elasticsearch в образе}/snapshots.

Используя API зарегистрируйте данную директорию как snapshot repository c именем netology_backup.

Приведите в ответе запрос API и результат вызова API для создания репозитория.
```
root@anton-v-m:~# curl -X PUT "localhost:9300/_snapshot/netology_backup?pretty" -H 'Content-Type: application/json' -d'
> {
>   "type": "fs",
>   "settings": {
>     "location": "snapshots"
>   }
> }
> '
{
  "acknowledged" : true
}

```

Создайте индекс test с 0 реплик и 1 шардом и приведите в ответе список индексов.

Создайте snapshot состояния кластера elasticsearch.
```
root@anton-v-m:~# curl -X PUT "localhost:9300/_snapshot/netology_backup/elastic_snapshot?pretty"
{
  "accepted" : true
}

```

Приведите в ответе список файлов в директории со snapshotами.
```
[elasticsearch@183a53de0111 snapshots]$ ls -la
total 44
drwxr-xr-x 3 elasticsearch elasticsearch  4096 Apr 18 21:18 .
drwxrwxr-x 3 elasticsearch elasticsearch  4096 Apr 18 21:11 ..
-rw-r--r-- 1 elasticsearch elasticsearch   592 Apr 18 21:18 index-0
-rw-r--r-- 1 elasticsearch elasticsearch     8 Apr 18 21:18 index.latest
drwxr-xr-x 3 elasticsearch elasticsearch  4096 Apr 18 21:18 indices
-rw-r--r-- 1 elasticsearch elasticsearch 17974 Apr 18 21:18 meta-Ixiz8skgTBiNDtyWobDUHQ.dat
-rw-r--r-- 1 elasticsearch elasticsearch   306 Apr 18 21:18 snap-Ixiz8skgTBiNDtyWobDUHQ.dat

```
Удалите индекс test и создайте индекс test-2. Приведите в ответе список индексов.

```
root@anton-v-m:~# curl -X GET http://localhost:9300/_cat/indices?v
health status index  uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test-2 K0WEE18DSNybRvMe6dQZ6A   1   0          0            0       225b           225b

```

Восстановите состояние кластера elasticsearch из snapshot, созданного ранее.

Приведите в ответе запрос к API восстановления и итоговый список индексов.
```
root@anton-v-m:~# curl -X POST "localhost:9300/_snapshot/netology_backup/elastic_snapshot/_restore?pretty"
{
  "accepted" : true
}

```

```
root@anton-v-m:~# curl -X GET http://localhost:9300/_cat/indices?v
health status index  uuid                   pri rep docs.count docs.deleted store.size pri.store.size
green  open   test-2 K0WEE18DSNybRvMe6dQZ6A   1   0          0            0       225b           225b
green  open   test   5-Pg11pfQ--2Z7pb5vOAeg   1   0          0            0       225b           225b

```