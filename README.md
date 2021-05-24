# clickhouse

1. docker 启动单节点
```bash
$ docker run -d --name clickhouse-server \
	--ulimit nofile=262144:262144 \
	-p 9000:9000 \
    -p 8123:8123 \
    -p 9004:9004 \
	-v $PWD/data/test:/test \
	-v $PWD/data/node0/etc:/etc/clickhouse-server \
	-v $PWD/data/node0/data:/var/lib/clickhouse \
	--privileged=true --user=root \
	yandex/clickhouse-server:21.3
```

2. 生成并设置密码

```bash
$ PASSWORD="密码"; echo "$PASSWORD"; echo -n "$PASSWORD" | sha256sum | tr -d '-'
```

3. 在 users.xml 中设置密码
```xml
<password_double_sha1_hex>6bb4837eb74329105ee4568dda7dc67ed2ca2ad9</password_double_sha1_hex>
```

4.  使用 client 连接节点进行测试
```bash
$ clickhouse-client -h 127.0.0.1 --port 9000 -u default --password 123456 --query "CREATE DATABASE IF NOT EXISTS tutorial"

$ clickhouse-client --query "INSERT INTO tutorial.hits_v1 FORMAT TSV" --max_insert_block_size=100000 < hits_v1.tsv

$ clickhouse-client -h 127.0.0.1 --port 9000 -u default --password 123456 --query "INSERT INTO tutorial.visits_v1 FORMAT TSV" --max_insert_block_size=100000 < visits_v1.tsv

$ clickhouse-client -m -h 127.0.0.1 --port 9000 -u default --password 123456

SELECT
    StartURL AS URL,
    AVG(Duration) AS AvgDuration
FROM tutorial.visits_v1
WHERE StartDate BETWEEN '2014-03-23' AND '2014-03-30'
GROUP BY URL
ORDER BY AvgDuration DESC
LIMIT 10
```

> 由于本机配置较低，可能会报内存不足等错误,可以设置允许使用磁盘进行查询
```text
set max_bytes_before_external_group_by=20000000000;
```

# 搭建集群

1. 准备

这里使用 docker compose 在本机搭建进行测试(1 zookeeper + 3 clickhouse)

2. 参考单节点的密码在`users.xml`中设置密码
```xml
<password_double_sha1_hex>6bb4837eb74329105ee4568dda7dc67ed2ca2ad9</password_double_sha1_hex>
```

3. 在 `config.xml` 中设置分片配置和 zookeeper地址
```xml

<remote_servers>
        <!-- 集群名 -->
    <perftest_3shards_1replicas>
        <!-- 分片地址 -->
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
               <!-- 我这里使用的是 docker compose  故可以直接使用容器名字 -->
                <host>clickhouse01</host>
                <port>9000</port>
                <user>default</user>
                <password>123456</password>
            </replica>
        </shard>
        <shard>
            <internal_replication>true</internal_replication>
            <replica> 
                <host>clickhouse02</host>
                <port>9000</port>
                <user>default</user>
                <password>123456</password>
            </replica>
        </shard>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>clickhouse03</host>
                <port>9000</port>
                <user>default</user>
                <password>123456</password>
            </replica>
        </shard>
    </perftest_3shards_1replicas>
</remote_servers>

<zookeeper>
    <node>
        <!-- zookeeper 地址 -->
        <host>zookeeper</host>
        <port>2181</port>
    </node>
</zookeeper>

<!-- 宏配置，用于分布式建表时做替换，每个节点配置不能相同 -->
<macros>
    <shard>03</shard>
    <replica>03</replica>
</macros>
```

4. 启动集群
```bash
$ docker compose up -d
```

5. 登陆不同的节点进行测试
```bash
$ clickhouse-client -h 127.0.0.1 --port 9000 -u default --password 123456 --query "CREATE DATABASE IF NOT EXISTS tutorial"

$ clickhouse-client -m -h 127.0.0.1 --port 9000 -u default --password 123456

CREATE TABLE tutorial.visits_v1 on cluster perftest_3shards_1replicas
(
    `CounterID` UInt32,
    `StartDate` Date,
    `Sign` Int8,
    `IsNew` UInt8,
    `VisitID` UInt64,
    `UserID` UInt64,
    `StartTime` DateTime,
    `Duration` UInt32,
    `UTCStartTime` DateTime,
    `PageViews` Int32,
    `Hits` Int32,
    `IsBounce` UInt8,
    `Referer` String,
    `StartURL` String,
    `RefererDomain` String,
    `StartURLDomain` String,
    `EndURL` String,
    `LinkURL` String,
    `IsDownload` UInt8,
    `TraficSourceID` Int8,
    `SearchEngineID` UInt16,
    `SearchPhrase` String,
    `AdvEngineID` UInt8,
    `PlaceID` Int32,
    `RefererCategories` Array(UInt16),
    `URLCategories` Array(UInt16),
    `URLRegions` Array(UInt32),
    `RefererRegions` Array(UInt32),
    `IsYandex` UInt8,
    `GoalReachesDepth` Int32,
    `GoalReachesURL` Int32,
    `GoalReachesAny` Int32,
    `SocialSourceNetworkID` UInt8,
    `SocialSourcePage` String,
    `MobilePhoneModel` String,
    `ClientEventTime` DateTime,
    `RegionID` UInt32,
    `ClientIP` UInt32,
    `ClientIP6` FixedString(16),
    `RemoteIP` UInt32,
    `RemoteIP6` FixedString(16),
    `IPNetworkID` UInt32,
    `SilverlightVersion3` UInt32,
    `CodeVersion` UInt32,
    `ResolutionWidth` UInt16,
    `ResolutionHeight` UInt16,
    `UserAgentMajor` UInt16,
    `UserAgentMinor` UInt16,
    `WindowClientWidth` UInt16,
    `WindowClientHeight` UInt16,
    `SilverlightVersion2` UInt8,
    `SilverlightVersion4` UInt16,
    `FlashVersion3` UInt16,
    `FlashVersion4` UInt16,
    `ClientTimeZone` Int16,
    `OS` UInt8,
    `UserAgent` UInt8,
    `ResolutionDepth` UInt8,
    `FlashMajor` UInt8,
    `FlashMinor` UInt8,
    `NetMajor` UInt8,
    `NetMinor` UInt8,
    `MobilePhone` UInt8,
    `SilverlightVersion1` UInt8,
    `Age` UInt8,
    `Sex` UInt8,
    `Income` UInt8,
    `JavaEnable` UInt8,
    `CookieEnable` UInt8,
    `JavascriptEnable` UInt8,
    `IsMobile` UInt8,
    `BrowserLanguage` UInt16,
    `BrowserCountry` UInt16,
    `Interests` UInt16,
    `Robotness` UInt8,
    `GeneralInterests` Array(UInt16),
    `Params` Array(String),
    `Goals` Nested(
        ID UInt32,
        Serial UInt32,
        EventTime DateTime,
        Price Int64,
        OrderID String,
        CurrencyID UInt32),
    `WatchIDs` Array(UInt64),
    `ParamSumPrice` Int64,
    `ParamCurrency` FixedString(3),
    `ParamCurrencyID` UInt16,
    `ClickLogID` UInt64,
    `ClickEventID` Int32,
    `ClickGoodEvent` Int32,
    `ClickEventTime` DateTime,
    `ClickPriorityID` Int32,
    `ClickPhraseID` Int32,
    `ClickPageID` Int32,
    `ClickPlaceID` Int32,
    `ClickTypeID` Int32,
    `ClickResourceID` Int32,
    `ClickCost` UInt32,
    `ClickClientIP` UInt32,
    `ClickDomainID` UInt32,
    `ClickURL` String,
    `ClickAttempt` UInt8,
    `ClickOrderID` UInt32,
    `ClickBannerID` UInt32,
    `ClickMarketCategoryID` UInt32,
    `ClickMarketPP` UInt32,
    `ClickMarketCategoryName` String,
    `ClickMarketPPName` String,
    `ClickAWAPSCampaignName` String,
    `ClickPageName` String,
    `ClickTargetType` UInt16,
    `ClickTargetPhraseID` UInt64,
    `ClickContextType` UInt8,
    `ClickSelectType` Int8,
    `ClickOptions` String,
    `ClickGroupBannerID` Int32,
    `OpenstatServiceName` String,
    `OpenstatCampaignID` String,
    `OpenstatAdID` String,
    `OpenstatSourceID` String,
    `UTMSource` String,
    `UTMMedium` String,
    `UTMCampaign` String,
    `UTMContent` String,
    `UTMTerm` String,
    `FromTag` String,
    `HasGCLID` UInt8,
    `FirstVisit` DateTime,
    `PredLastVisit` Date,
    `LastVisit` Date,
    `TotalVisits` UInt32,
    `TraficSource` Nested(
        ID Int8,
        SearchEngineID UInt16,
        AdvEngineID UInt8,
        PlaceID UInt16,
        SocialSourceNetworkID UInt8,
        Domain String,
        SearchPhrase String,
        SocialSourcePage String),
    `Attendance` FixedString(16),
    `CLID` UInt32,
    `YCLID` UInt64,
    `NormalizedRefererHash` UInt64,
    `SearchPhraseHash` UInt64,
    `RefererDomainHash` UInt64,
    `NormalizedStartURLHash` UInt64,
    `StartURLDomainHash` UInt64,
    `NormalizedEndURLHash` UInt64,
    `TopLevelDomain` UInt64,
    `URLScheme` UInt64,
    `OpenstatServiceNameHash` UInt64,
    `OpenstatCampaignIDHash` UInt64,
    `OpenstatAdIDHash` UInt64,
    `OpenstatSourceIDHash` UInt64,
    `UTMSourceHash` UInt64,
    `UTMMediumHash` UInt64,
    `UTMCampaignHash` UInt64,
    `UTMContentHash` UInt64,
    `UTMTermHash` UInt64,
    `FromHash` UInt64,
    `WebVisorEnabled` UInt8,
    `WebVisorActivity` UInt32,
    `ParsedParams` Nested(
        Key1 String,
        Key2 String,
        Key3 String,
        Key4 String,
        Key5 String,
        ValueDouble Float64),
    `Market` Nested(
        Type UInt8,
        GoalID UInt32,
        OrderID String,
        OrderPrice Int64,
        PP UInt32,
        DirectPlaceID UInt32,
        DirectOrderID UInt32,
        DirectBannerID UInt32,
        GoodID String,
        GoodName String,
        GoodQuantity Int32,
        GoodPrice Int64),
    `IslandID` FixedString(16)
)
ENGINE = CollapsingMergeTree(Sign)
PARTITION BY toYYYYMM(StartDate)
ORDER BY (CounterID, StartDate, intHash32(UserID), VisitID)
SAMPLE BY intHash32(UserID)

CREATE TABLE tutorial.visits_all AS tutorial.visits_v1 on cluster perftest_3shards_1replicas
ENGINE = Distributed(perftest_3shards_1replicas, tutorial, visits_v1, rand());

INSERT INTO tutorial.visits_all SELECT * FROM tutorial.visits_v1;


clickhouse-client -h 127.0.0.1 --port 9000 -u default --password 123456 --query "INSERT INTO tutorial.visits_all FORMAT TSV" --max_insert_block_size=100000 < visits_v1.tsv


SELECT
    StartURL AS URL,
    AVG(Duration) AS AvgDuration
FROM tutorial.visits_all
WHERE StartDate BETWEEN '2014-03-23' AND '2014-03-30'
GROUP BY URL
ORDER BY AvgDuration DESC
LIMIT 10
```
