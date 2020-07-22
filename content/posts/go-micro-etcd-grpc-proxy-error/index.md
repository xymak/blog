---
title: "etcd grpc proxy缓存引发的问题"
date: 2020-07-22T00:44:54+08:00
---

最近把go-micro升级到v2，上线后遇到一个问题：网关偶尔会把流量解析到已经下线的服务上去。
```shell script
http: proxy error: dial tcp xxx.xxx.xxx.xxx:xxx: connect: connection refused
```
导致大量的502产生，而且重启网关或者重启服务都不管用，于是怀疑是etcd出了问题。
我们服务发现用的是etcd集群，集群前面有3个etcd grpc proxy做反向代理，反向代理前面是一个lvs。
查看go micro etcd服务发现的相关代码：
```go
func (e *etcdRegistry) GetService(name string, opts ...registry.GetOption) ([]*registry.Service, error) {
	ctx, cancel := context.WithTimeout(context.Background(), e.options.Timeout)
	defer cancel()

	rsp, err := e.client.Get(ctx, servicePath(name)+"/", clientv3.WithPrefix(), clientv3.WithSerializable())
	if err != nil {
		return nil, err
	}

	if len(rsp.Kvs) == 0 {
		return nil, registry.ErrNotFound
	}

	...
}
```
通过debug发现rsp真的返回了已经下线的版本注册信息。但是通过etcdctl --endpoints={lvs地址} get --prefix命令，数据都是最新的。这就很诡异了，同一个地址，怎么可能数据不一样？
再仔细看发现client.Get那一行有个option我没见过。进一步查看代码：
```go
// WithSerializable makes 'Get' request serializable. By default,
// it's linearizable. Serializable requests are better for lower latency
// requirement.
func WithSerializable() OpOption {
	return func(op *Op) { op.serializable = true }
}
```
注释几乎没说什么，通过查看这篇知乎这篇文章[《etcd raft如何实现Linearizable Read》](https://zhuanlan.zhihu.com/p/27869566) 了解到etcd做读请求，分Linearizable Read和Serializable Read。Linearizable Read情况下，当时的leader节点是要请求其它member节点做一致性校验（自己是不是leader之类），校验通过后，然后再返回数据的。而Serializable Read是不做校验，直接返回结果。
所以上面代码注释说Serializable Read对低延时的场景更为友好。
我分析了一下，难道是没有用一致性更高的Linearizable Read导致，读取了老数据吗？也不可能，有老数据，以为着etcd多个节点数据不一致或者切换了leader，也不可能一直切换leader啊。
我把Serializable Read选项去掉了。debug发现能获取到最新数据，而且线上部分错误少了，但是又不是还是有错误。这个情况好像是某个节点缓存被清了，其它节点还是有缓存的感觉。

所以现在怀疑grpc proxy有缓存，而且这个缓存跟Serializable Read和Linearizable Read有关系。直接去看grpc proxy的代码。发现真的是有cache机制，定位到[kv.go](https://github.com/etcd-io/etcd/blob/master/proxy/grpcproxy/kv.go)这个文件：
```go
func (p *kvProxy) Range(ctx context.Context, r *pb.RangeRequest) (*pb.RangeResponse, error) {
	if r.Serializable {
		resp, err := p.cache.Get(r)
		switch err {
		case nil:
			cacheHits.Inc()
			return resp, nil
		case cache.ErrCompacted:
			cacheHits.Inc()
			return nil, err
		}

		cachedMisses.Inc()
	}

	resp, err := p.kv.Do(ctx, RangeRequestToOp(r))
	if err != nil {
		return nil, err
	}

	// cache linearizable as serializable
	req := *r
	req.Serializable = true
	gresp := (*pb.RangeResponse)(resp.Get())
	p.cache.Add(&req, gresp)
	cacheKeys.Set(float64(p.cache.Size()))

	return gresp, nil
}
```
这个range函数是处理get请求用的，上面有个条件如果是Serializable Read，会先去缓存里面找，如果没找到或者是Linearizable Read就直接请求etcd，然后再把缓存打上。缓存是怎么失效的呢？
```go
func (p *kvProxy) Put(ctx context.Context, r *pb.PutRequest) (*pb.PutResponse, error) {
	p.cache.Invalidate(r.Key, nil)
	cacheKeys.Set(float64(p.cache.Size()))

	resp, err := p.kv.Do(ctx, PutRequestToOp(r))
	return (*pb.PutResponse)(resp.Put()), err
}

func (p *kvProxy) DeleteRange(ctx context.Context, r *pb.DeleteRangeRequest) (*pb.DeleteRangeResponse, error) {
	p.cache.Invalidate(r.Key, r.RangeEnd)
	cacheKeys.Set(float64(p.cache.Size()))

	resp, err := p.kv.Do(ctx, DelRequestToOp(r))
	return (*pb.DeleteRangeResponse)(resp.Del()), err
}
```
修改和删除的时候，会调用Invalidate使缓存失效。而且是没有自动watch一个key来刷新缓存的。

现在就能定位问题了，etcd集群前面是有3个grpc proxy，一个写请求只能让一个节点缓存刷新，所以其它节点还存在老的数据。把2个grpc proxy权重改为0后，问题消失了。
本身我只是用grpc proxy让etcd集群无状态，外部客户端不需要感知集群成员变更，3个节点让可用性更高一点，没想到grpc proxy是有缓存的。