---
title: TinyKV系列(2)-standaloneKV

# Summary for listings and search engines
summary: TinyKV系列课程(2)-standaloneKV

# Link this post with a project
projects: []

# Date published
date: '2023-9-13T00:00:00Z'

# Date updated
lastmod: '2023-9-13T00:00:00Z'

# Is this an unpublished draft?
draft: false

# Show this page in the Featured widget?
featured: false

authors:
- admin

tags:
- TinyKV
- 分布式存储
- Raft

categories:
- TinyKV
- 分布式存储
- Raft
---

## 项目任务
整个项目分成了4个lab，
- standalone KV
  - 实施独立的存储引擎。
  - 实施原始键值服务处理程序。
- raft KV
  - 实现基本的 Raft 算法。
  - 在 Raft 之上构建容错 KV 服务器。
  - 添加Raft日志垃圾收集和快照的支持。
- multi raft KV
  - 对 Raft 算法实施成员变更和领导层变更。
  - 在 Raft 存储上实现conf更改和区域分割。
  - 实现一个基本的调度程序。
- 事务
  - 实现多版本并发控制层。
  - KvGet实现、KvPrewrite和请求的处理程序KvCommit。
  - 实现KvScan、KvCheckTxnStatus、KvBatchRollback和KvResolveLock请求的处理程序。
## standalone kv
在lab1中，要实现的是单机的kv存储，只有一个节点，所以不用考虑raft、分布式等问题，
这个服务支持Put、Delete、Get、Scan（查询特定cf下连续一系列key）四种操作 ，细分为两个子任务：
- 实现存储引擎engine
- 实现服务处理程序（handler后面的service层，即对上述引擎的增删改查进行封装）
## 具体实现
### 1、standaloneStorage结构体
通过提示2知道，engine_util对badger进行了封装，StandAloneStorage就是要在engine_util的基础上再封装一层。
另外需要用到conf的参数进行初始化，所以将conf也传递进来
```go
type StandAloneStorage struct {
	engine *engine_util.Engines
	conf   *config.Config
}
```
### 2、实现Storage接口
自定义的standaloneStorage要实现如下的接口
```go
type Storage interface {
	Start() error
	Stop() error
	Write(ctx *kvrpcpb.Context, batch []Modify) error
	Reader(ctx *kvrpcpb.Context) (StorageReader, error)
}
```
首先关注Write，可以注意到Modify通过类型断言对put和delete来进行区分，
而且engine_util对方法进行了封装 
```go
func PutCF(engine *badger.DB, cf string, key []byte, val []byte) error
func DeleteCF(engine *badger.DB, cf string, key []byte) error
```
所以只要在Write中调用这两个方法即可：
```go
func (s *StandAloneStorage) Write(ctx *kvrpcpb.Context, batch []storage.Modify) error {
	var err error
	for _, m := range batch {
		key, val, cf := m.Key(), m.Value(), m.Cf()
		if _, ok := m.Data.(storage.Put); ok {
			err = engine_util.PutCF(s.engine.Kv, cf, key, val)
		} else {
			err = engine_util.DeleteCF(s.engine.Kv, cf, key)
		}
		if err != nil {
			return err
		}
	}
	return nil
}
```
接下来实现Reader：

Reader返回的是一个```StorageReader```接口，
```go
Reader(ctx *kvrpcpb.Context) (StorageReader, error)

type StorageReader interface {
	GetCF(cf string, key []byte) ([]byte, error)
	IterCF(cf string) engine_util.DBIterator
	Close()
}
```
所以我们只需要实现这三个方法即可，注意到engine_util提供了
```go
func GetCFFromTxn(txn *badger.Txn, cf string, key []byte) (val []byte, err error)
func NewCFIterator(cf string, txn *badger.Txn) *BadgerIterator
```
所以只需要创建txn实例，他是badger的事务，可以通过如下函数获得
```go
func (db *DB) NewTransaction(update bool) *Txn
```
这样我们就实现了Reader接口，完整Reader代码如下:
```go
func (s *StandAloneStorage) Reader(ctx *kvrpcpb.Context) (storage.StorageReader, error) {
	txn := s.engine.Kv.NewTransaction(false)
	return NewStandAloneStorageReader(txn), nil
}

//实现 type StorageReader interface 接口

type StandAloneStorageReader struct {
	KvTxn *badger.Txn
}

func NewStandAloneStorageReader(txn *badger.Txn) *StandAloneStorageReader {
	return &StandAloneStorageReader{
		KvTxn: txn,
	}
}

func (s *StandAloneStorageReader) GetCF(cf string, key []byte) ([]byte, error) {
	val, err := engine_util.GetCFFromTxn(s.KvTxn, cf, key)
	if err == badger.ErrKeyNotFound {
		return nil, nil
	}
	return val, err
}

func (s *StandAloneStorageReader) IterCF(cf string) engine_util.DBIterator {
	return engine_util.NewCFIterator(cf, s.KvTxn)
}

func (s *StandAloneStorageReader) Close() {
	s.KvTxn.Discard()
}

```
到目前为止我们已经完成了第一个任务：实现standalonestorage引擎
### 3、实现RawAPI层
这一层比较简单，就是将get、scan请求调用上述实现的reader方法、
将put和delete请求调用write方法
详细代码如下
```go
func (server *Server) RawGet(_ context.Context, req *kvrpcpb.RawGetRequest) (*kvrpcpb.RawGetResponse, error) {
	reader, err := server.storage.Reader(req.Context)
	if err != nil {
		return nil, err
	}
	val, err := reader.GetCF(req.Cf, req.Key)
	if err != nil {
		return nil, err
	}
	resp := &kvrpcpb.RawGetResponse{
		Value:    val,
		NotFound: false,
	}
	if val == nil {
		resp.NotFound = true
	}
	return resp, nil
}

// RawPut puts the target data into storage and returns the corresponding response
func (server *Server) RawPut(_ context.Context, req *kvrpcpb.RawPutRequest) (*kvrpcpb.RawPutResponse, error) {
	// Hint: Consider using Storage.Modify to store data to be modified
	put := storage.Put{
		Key:   req.Key,
		Value: req.Value,
		Cf:    req.Cf,
	}
	batch := storage.Modify{Data: put}
	err := server.storage.Write(req.Context, []storage.Modify{batch})
	if err != nil {
		return nil, err

	}
	return &kvrpcpb.RawPutResponse{}, nil
}

// RawDelete delete the target data from storage and returns the corresponding response
func (server *Server) RawDelete(_ context.Context, req *kvrpcpb.RawDeleteRequest) (*kvrpcpb.RawDeleteResponse, error) {
	// Hint: Consider using Storage.Modify to store data to be deleted
	del := storage.Delete{
		Key: req.Key,
		Cf:  req.Cf,
	}
	batch := storage.Modify{Data: del}
	err := server.storage.Write(req.Context, []storage.Modify{batch})
	if err != nil {
		return nil, err
	}
	return &kvrpcpb.RawDeleteResponse{}, nil
}

// RawScan scan the data starting from the start key up to limit. and return the corresponding result
func (server *Server) RawScan(_ context.Context, req *kvrpcpb.RawScanRequest) (*kvrpcpb.RawScanResponse, error) {
	// Hint: Consider using reader.IterCF
	reader, err := server.storage.Reader(req.Context)
	if err != nil {
		return nil, err
	}
	var Kvs []*kvrpcpb.KvPair
	limit := req.Limit

	iter := reader.IterCF(req.Cf)
	defer iter.Close()


	for iter.Seek(req.StartKey); iter.Valid(); iter.Next() {
		item := iter.Item()
		key := item.Key()
		val, _ := item.Value()

		Kvs = append(Kvs, &kvrpcpb.KvPair{
			Key:   key,
			Value: val,
		})
		limit--
		if limit == 0 {
			break
		}
	}
	resp := &kvrpcpb.RawScanResponse{
		Kvs: Kvs,
	}
	return resp, nil
}
```
### 4、单元测试
```shell
=== RUN   TestRawGet1
--- PASS: TestRawGet1 (0.08s)
=== RUN   TestRawGetNotFound1
--- PASS: TestRawGetNotFound1 (0.09s)
=== RUN   TestRawPut1
--- PASS: TestRawPut1 (0.07s)
=== RUN   TestRawGetAfterRawPut1
--- PASS: TestRawGetAfterRawPut1 (0.07s)
=== RUN   TestRawGetAfterRawDelete1
--- PASS: TestRawGetAfterRawDelete1 (0.07s)
=== RUN   TestRawDelete1
--- PASS: TestRawDelete1 (0.09s)
=== RUN   TestRawScan1
--- PASS: TestRawScan1 (0.07s)
=== RUN   TestRawScanAfterRawPut1
--- PASS: TestRawScanAfterRawPut1 (0.07s)
=== RUN   TestRawScanAfterRawDelete1
--- PASS: TestRawScanAfterRawDelete1 (0.09s)
=== RUN   TestIterWithRawDelete1
--- PASS: TestIterWithRawDelete1 (0.09s)
PASS
ok  	github.com/pingcap-incubator/tinykv/kv/server	3.412s
```
### 5、心得体会
- 不要着急写代码，先对照文档梳理逻辑，对项目整体上有一个认识，明白每个目录下的代码的作用
- 万事开头难，然后中间难，最后结尾难，重在坚持
- 不要忽略测试用例
