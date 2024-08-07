# etcd-learning

References:

[ETCD - IBM](https://www.ibm.com/topics/etcd)

[ETCD - facsiaginsa](https://facsiaginsa.com/etcd/how-to-setup-etcd-cluster)

[etcd.io](https://etcd.io/docs/v3.5/tutorials/how-to-setup-cluster/)

## Tổng quan vể etcd
Lưu trữ và duy trì hoạt động của các hệ thống phân tán
Là một trong những thành phần cốt lõi của k8s (lưu state data, configuration data, metadata, …

### Tính năng:

1. fully replicated
2. reliably consistent (nhất quán đáng tin cậy)
- Etcd được xây dựng dựa trên thuật toán Raft consensus algorithm để đảm bảo dữ liệu được lưu một cách nhất quán xuyên suốt tất cả các node trong cụm
- Thực hiện bầu ra leader node quản lý các replica cho các node khác gọi là followers. Leader sẽ nhận request từ client, sau đó chuyển tiếp cho follower node, Từng node followers cập nhật xong sau đó gửi phản hồi cập nhật thành công cho leader. Khi leader xác định được phần lớn các node followers đã lưu dữ liệu với , nó sẽ lưu cặp key-value đó vào local state machine của nó, rồi return kết quả về cho client. Nếu followers crash hoặc mất mát gói tin network, leader sẽ thử lại (retries) cho đến khi tất cả followers lưu log entries một cách nhất quán
Nếu một node fail khi nhận message từ leader trong một khoảng thời gian lặp lại cụ thể, sẽ có 1 buổi bầu cử leader mới. một followers sẽ được chọn làm leader mới dựa vào tính sẵn sàng của nó
- Điều này giúp cho etcd có tính highly availability

3. Highly available: etcd is designed to have no single point of failure and gracefully tolerate hardware failures and network partitions.
4. Fast: etcd has been benchmarked at 10,000 writes per second.
5. Secure: etcd supports automatic Transport Layer Security (TLS) and optional secure socket layer (SSL) client certificate authentication. Because etcd stores vital and highly sensitive configuration data, administrators should implement role-based access controls within the deployment and ensure that team members interacting with etcd are limited to the least-privileged level of access necessary to perform their jobs.


Simple: Any application, from simple web apps to highly complex container orchestration engines such as Kubernetes, can read or write data to etcd using standard HTTP/JSON  tools.
Trong kubernetes, sử dụng cơ chế “watch" khi so sánh actual với config khi có bất kỳ sự thay đổi nào xảy ra.


## Setup etcd cluster 3 node

- VM1: 192.168.144.129
- VM2: 192.168.144.133
- VM3: 192.168.144.136

Tại node 1

```
ETCD_NAME=etcd1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=http://192.168.144.129:2379,http://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=http://192.168.144.129:2380
ETCD_ADVERTISE_CLIENT_URLS=http://192.168.144.129:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.144.129:2380
ETCD_INITIAL_CLUSTER=etcd1=http://192.168.144.129:2380,etcd2=http://192.168.144.133:2380,etcd3=http://192.168.144.136:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster

```

Tại node 2

```
ETCD_NAME=etcd2
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=http://192.168.144.133:2379,http://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=http://192.168.144.133:2380
ETCD_ADVERTISE_CLIENT_URLS=http://192.168.144.133:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.144.133:2380
ETCD_INITIAL_CLUSTER=etcd1=http://192.168.144.129:2380,etcd2=http://192.168.144.133:2380,etcd3=http://192.168.144.136:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
```

Tại node 3

```
ETCD_NAME=etcd3
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=http://192.168.144.136:2379,http://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=http://192.168.144.136:2380
ETCD_ADVERTISE_CLIENT_URLS=http://192.168.144.136:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://192.168.144.136:2380
ETCD_INITIAL_CLUSTER=etcd1=http://192.168.144.129:2380,etcd2=http://192.168.144.133:2380,etcd3=http://192.168.144.136:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
```


Từ 1 trong 3 node chạy lệnh để kiểm tra danh sách thành viên etcd trong cluster

```
etcdctl --endpoints=http://192.168.144.129:2379 --write-out=table member list
```

Output:
```
+------------------+---------+-------+-----------------------------+-----------------------------+
|        ID        | STATUS  | NAME  |         PEER ADDRS          |        CLIENT ADDRS         |
+------------------+---------+-------+-----------------------------+-----------------------------+
|   ae11938136e019 | started | etcd1 | http://192.168.144.129:2380 | http://192.168.144.129:2379 |
| 1a36316d08590385 | started | etcd3 | http://192.168.144.136:2380 | http://192.168.144.136:2379 |
| 50aa2c95a5a6c74b | started | etcd2 | http://192.168.144.133:2380 | http://192.168.144.133:2379 |
+------------------+---------+-------+-----------------------------+-----------------------------+
```

Hoặc truy cập http://192.168.144.129:2380/members

```
[{"id":48995971173376025,"peerURLs":["http://192.168.144.129:2380"],"name":"etcd1","clientURLs":["http://192.168.144.129:2379"]},{"id":1888751438089749381,"peerURLs":["http://192.168.144.136:2380"],"name":"etcd3","clientURLs":["http://192.168.144.136:2379"]},{"id":5812507290315966283,"peerURLs":["http://192.168.144.133:2380"],"name":"etcd2","clientURLs":["http://192.168.144.133:2379"]}]
1
```

## Note
Nếu bị lỗi `No help topic for ...`

Fix: `export ETCDCTL_API=3`

### Service:
```
systemctl enable etcd
service etcd start
service etcd status
```

### Common command

1. Put a pair

 ```
 etcdctl put mykey myvalue
 ```

2. Get a value by key
 ```
 etcdctl get mykey
 ```

3. Get entries by prefix

 ```
 etcdctl get "example" --prefix
 ```

4. Delete one key
   
 ```
 etcdctl del mykey
 ```


## Kết nối API Server với etcd (VD: Golang)

### Connect

```
var Etcd_cli *clientv3.Client

func ConnectToEtcd() {

	var err error

	Etcd_cli, err = clientv3.New(clientv3.Config{
		Endpoints:   []string{"192.168.144.129:2379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		log.Fatal(err)
	}
}

```

### put, get (prefix), delete:

```
package etcd

import (
	"context"
	"go-api-server/initializers"
	"log"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func PutEntry(key string, value string) error {
	cli := initializers.Etcd_cli

	if cli == nil {
		log.Fatal("Etcd isn't connected!")
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)

	_, err := cli.Put(ctx, key, value)
	cancel()
	return err
}

func GetEntryByKey(key string) (*clientv3.GetResponse, error) {
	cli := initializers.Etcd_cli

	if cli == nil {
		log.Fatal("Etcd isn't connected!")
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)

	resp, err := cli.Get(ctx, key)
	cancel()
	if err != nil {
		log.Fatal(err)
	}

	return resp, err
}

func GetEntryByPrefix(prefix string) (*clientv3.GetResponse, error) {
	cli := initializers.Etcd_cli

	if cli == nil {
		log.Fatal("Etcd isn't connected!")
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)

	resp, err := cli.Get(ctx, prefix, clientv3.WithPrefix())
	cancel()
	if err != nil {
		log.Fatal(err)
	}

	return resp, err
}

func DeleteEntry(key string) error {
	cli := initializers.Etcd_cli

	if cli == nil {
		log.Fatal("Etcd isn't connected!")
	}
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)

	_, err := cli.Delete(ctx, key)
	cancel()
	if err != nil {
		log.Fatal(err)
	}
	return err
}

```
