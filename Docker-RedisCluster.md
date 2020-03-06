啟動並確認服務狀態
```<language>
sudo systemctl start docker
sudo systemctl status docker.service
```

docker設定下載proxy
```<language>
mkdir -p /etc/systemd/system/docker.service.d
vi /etc/systemd/system/docker.service.d/http-proxy.conf
	[Service]
	Environment="HTTP_PROXY=xxxxxxxxx"
	[Service]
	Environment="HTTPS_PROXY=xxxxxxxxxxxx"
```

重新啟動docker
```<language>
sudo systemctl daemon-reload
sudo systemctl restart docker
```

確認環境變數
```<language>
systemctl show --property=Environment docker
```

下載docker images (建議暫時不要使用redis 4.0.0以上版本,C# Client有點問題)
```<language>
sudo docker pull redis:3.2.9
```

建立redis設定檔
```<language>
sudo mkdir -p /myredis/conf
sudo touch /myredis/conf/redis7001.conf
sudo touch /myredis/conf/redis7002.conf
sudo touch /myredis/conf/redis7003.conf
sudo touch /myredis/conf/redis7004.conf
sudo touch /myredis/conf/redis7005.conf
sudo touch /myredis/conf/redis7006.conf
sudo touch /myredis/conf/redis7007.conf
sudo touch /myredis/conf/redis7008.conf
```

安裝docker redis container
```<language>
sudo docker run -v /myredis/conf/redis7001.conf:/usr/local/etc/redis/redis.conf --net=host -p 7001:7001 -p 17001:17001 --name redis01 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
sudo docker run -v /myredis/conf/redis7002.conf:/usr/local/etc/redis/redis.conf --net=host -p 7002:7002 -p 17002:17002 --name redis02 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
sudo docker run -v /myredis/conf/redis7003.conf:/usr/local/etc/redis/redis.conf --net=host -p 7003:7003 -p 17003:17003 --name redis03 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
sudo docker run -v /myredis/conf/redis7004.conf:/usr/local/etc/redis/redis.conf --net=host -p 7004:7004 -p 17004:17004 --name redis04 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
sudo docker run -v /myredis/conf/redis7005.conf:/usr/local/etc/redis/redis.conf --net=host -p 7005:7005 -p 17005:17005 --name redis05 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
sudo docker run -v /myredis/conf/redis7006.conf:/usr/local/etc/redis/redis.conf --net=host -p 7006:7006 -p 17006:17006 --name redis06 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
sudo docker run -v /myredis/conf/redis7007.conf:/usr/local/etc/redis/redis.conf --net=host -p 7007:7007 -p 17007:17007 --name redis07 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
sudo docker run -v /myredis/conf/redis7008.conf:/usr/local/etc/redis/redis.conf --net=host -p 7008:7008 -p 17008:17008 --name redis08 -d  redis:3.2.9 redis-server /usr/local/etc/redis/redis.conf
```
先把6台服務一起啟動
```<language>
docker start redis01 redis02 redis03 redis05 redis06 redis07
```
確認啟動狀態
```<language>
docker ps -a
```

啟動redis叢集 (啟動安裝請參考[善心人士提供](https://hub.docker.com/r/zvelo/redis-trib/)的建立Cluster方式)
```<language>
redis-trib.rb create --replicas 1 10.101.1.110:7001 10.101.1.110:7002 10.101.1.110:7003 10.101.1.110:7005 10.101.1.110:7006 10.101.1.110:7007
```

確認叢集狀態
```<language>
redis-trib.rb check 10.101.1.110:7001
```

叢集測試案例：
1. 加入master

```<language>
add-node 10.101.1.110:7004 10.101.1.110:7001
```

   
1. 加入slave

```<language>
add-node --slave --master-id b0f03581f787c991ef0ee641d06be9bb1aa6606f 10.101.1.110:7008 10.101.1.110:7004
```


3. 為新加入節點分配slot

```<language>
reshard 10.101.1.110:7001 (之後會詢問要移動多少slots、接收節點是誰、要從哪些節點移動slot)
```


1. master failover確認是否成為slave   



2. 延續3. 還原成master確認



3. slave failover

![

1. 刪除master、slave

```<language>
/// Master (移除master之前,請記得先清空已經分配到的slot,嘗試使用案例3的方式重新分配slot)
del-node 10.101.1.110:7004 8bc21197b94ed9ba3703d3643d5ed8a3399463ed

/// Slave (不需要重新分配slot)
del-node 10.101.1.110:7008 dc05c7005e664b24709475bb17114a2ecf6bca71
```




延續問題再展開...

如何使用工具分析key in memory size?

在軟體自由界總是有很多善心人士([請看這裡](https://github.com/sripathikrishnan/redis-rdb-tools))

我們使用的工具是一套redis-rdb-tools,主要是利用redis產生的rdb檔案進行分析

這樣可以避免掉直接對主機記憶體掃描產生的效能問題

首先我們先把container的rdb給複製到本機來

```<language>
sudo docker cp redis01:/data /hsyu
```

接著使用redis-rdb-tools產生csv檔案

```<language>
sudo rdb -c memory /hsyu/data/dump7001.rdb --byte 128 -f memory.csv
```

可以產生出幾個欄位
- database
- type
- key
- size_in_bytes
- encoding
- num_elements
- len_largest_element
