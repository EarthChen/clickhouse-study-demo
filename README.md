# docker 部署单节点

```bash
$ docker run -d --name clickhouse-server \
	--ulimit nofile=262144:262144 \
	-p 9000:9000 \
    -p 8123:8123 \
    -p 9004:9004 \
	-v $PWD/data/node0/etc:/etc/clickhouse-server \
	-v $PWD/data/node0/data:/var/lib/clickhosue \
	--privileged=true --user=root \
	yandex/clickhouse-server:21.3
```