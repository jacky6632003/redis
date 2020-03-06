- Redis安裝

[Windows安裝路徑](https://github.com/MicrosoftArchive/redis/releases)

安裝完成之後,建議使用內建的redis-cli.exe工具進行測試

```<language>
redis-cli.exe -h 127.0.0.1 -p 6380
```


- Redis Cluster

1. 資料分片
  -    Redis Cluster將資料分散到16384個slot上
  -    Cluster中的不同節點負責一部分slot
2. 故障恢復
  -    Cluster中直接提供服務的節點為Master
  -    每個Master可以有一個或多個Slave
  -    當Master不能提供服務時，Slave會自動Failover

Redis Cluster有幾個主要的設定

```<language>
// 叢集開關，預設是不開啟
cluster-enabled yes

// 叢集設定文件的名稱，每個節點都有一個叢集相關的設定文件
cluster-config-file nodes.conf

// 節點互連timeout時間(毫秒)
cluster-node-timeout 5000

// 在進行故障轉移的時候，全部slave都會請求申請為master，但是有些slave可能與master斷線連接一段時間了，導致資料過於老舊，這樣的slave不應該被提升為master。該參數就是用來判斷slave節點與master斷線的時間是否過長。
// (node-timeout * slave-validity-factor) + repl-ping-slave-period
cluster-slave-validity-factor 10

// master的slave數量大於這個數量，slave才能移轉到其他master上
cluster-migration-barrier 1

// 預設情況下，叢集全部的slot有節點負責，叢集狀態才為ok，才能提供服務。設置為no，可以在slot沒有全部分配的時候提供服務。
cluster-require-full-coverage yes
```

需要特別注意的是要設定Redis Cluster至少要3個Nodes

啟動&註冊服務

```<language>
redis-server.exe --service-install redis.6380.conf --service-name redis6380
redis-server.exe --service-start --service-name Redis6380
```

安裝個redis的ruby函式庫

```<language>
gem install --http-proxy=http://secproxy.evertrust.com.tw:6588 redis
```

Redise官網提供透過[Ruby Script](http://download.redis.io/redis-stable/src/redis-trib.rb)來建立Cluster

建立叢集記得先把Nodes溝通的Port打開,Port分別是Redis服務的Port + 10000 (ex:Redis Port 6379 -> 對應的Port:16379)

```<language>
redis-trib.rb create --replicas 0 127.0.0.1:6380 127.0.0.1:6381 127.0.0.1:6382
```

如果很不幸的叢集啟動失敗,要重新建立請將redis.conf中定義的cluster-config-file刪除,即可重新建立

另外附加一些對叢集操作的指令

```<language>
// 檢查節點
redis-trib.rb check 0 127.0.0.1:6380

// 修復節點
redis-trib.rb fix 127.0.0.1:6380

// 新增節點
// redis-trib.rb add-node --slave --master-
redis-trib.rb add-node 127.0.0.1:6383 127.0.0.1:6380

// 重新分配 slot
redis-trib.rb reshard 127.0.0.1:6380
// 另一種重新分配寫法
redis-trib.rb reshard --from <node-id> --to <node-id> --slots <number of slots> --yes <host>:<port>

// 刪除節點
redis-trib.rb del-node 127.0.0.1:6380 dd5333e3927eb199564b54308dbaf71cd5ae0b5d

```
