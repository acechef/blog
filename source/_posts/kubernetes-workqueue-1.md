---
title: kubernetes中的限速器
date: 2022-05-18 15:21:20
tags: kubernetes
---

简介
---
`Kubernetes`中的`client-go`包是`k8s`开发者必须要了解的一块内容，我也认为这是`Kubernetes`代码的天花板，其中的一些设计思想和实现非常值得我们学习和借鉴。

接下来，我们来看下`client-go`中的`workqueue`包的限速器(`rate_limiter`)的实现。

接口
---
在`workqueue`包下的`default_rate_limiters.go`下有好几种限速器，他们都实现了如下接口：
```go
type RateLimiter interface {
	// When gets an item and gets to decide how long that item should wait
	// 根据传入的item返回要等待的时间，传入的item对象会放入到一个map中，该map键为对象item，值为int类型的计数，每调用When，都会增加一次计数
	When(item interface{}) time.Duration
	// Forget indicates that an item is finished being retried.  Doesn't matter whether it's for failing
	// or for success, we'll stop tracking it
	// 清除map中的键为item的元素
	Forget(item interface{})
	// NumRequeues returns back how many failures the item has had
	// 获得map中键为item的数量，即返回map对应键的value
	NumRequeues(item interface{}) int
}
```
业务的区分主要在`when`方法的实现上，可以根据不同的`when`方法的实现返回不同的延迟时间

实现
---
我们主要来介绍以下这几钟`RateLimiter`的实现：

- BucketRateLimiter
- ItemExponentialFailureRateLimiter
- ItemFastSlowRateLimiter
- MaxOfRateLimiter

BucketRateLimiter
---
主要是官方包rate.Limiter令牌桶的封装，官方包在我的[上一篇文章](https://acechef.github.io/2022/05/13/golang-token-bucket/) 里有分析，这里不再阐述。

ItemExponentialFailureRateLimiter
---

该`RateLimiter`特殊点主要在`When`函数上，从函数名上可以看出，它是有关指数增长的，这在错误重试、控制器`Reconcile`的时候非常有用
```go
// ItemExponentialFailureRateLimiter does a simple baseDelay*2^<num-failures> limit
// dealing with max failures and expiration are up to the caller
// 实现了baseDelay*2^x 的指数增长限制
type ItemExponentialFailureRateLimiter struct {
	failuresLock sync.Mutex
	// 对象计数map
	failures     map[interface{}]int
    
	// 基础延迟
	baseDelay time.Duration
	// 最大延迟
	maxDelay  time.Duration
}

var _ RateLimiter = &ItemExponentialFailureRateLimiter{}

func NewItemExponentialFailureRateLimiter(baseDelay time.Duration, maxDelay time.Duration) RateLimiter {
	return &ItemExponentialFailureRateLimiter{
		failures:  map[interface{}]int{},
		baseDelay: baseDelay,
		maxDelay:  maxDelay,
	}
}

func DefaultItemBasedRateLimiter() RateLimiter {
	return NewItemExponentialFailureRateLimiter(time.Millisecond, 1000*time.Second)
}

// 可以看到这边实现了指数级时间的返回
func (r *ItemExponentialFailureRateLimiter) When(item interface{}) time.Duration {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	exp := r.failures[item]
	r.failures[item] = r.failures[item] + 1

	// The backoff is capped such that 'calculated' value never overflows.
	backoff := float64(r.baseDelay.Nanoseconds()) * math.Pow(2, float64(exp))
	if backoff > math.MaxInt64 {
		return r.maxDelay
	}

	calculated := time.Duration(backoff)
	if calculated > r.maxDelay {
		return r.maxDelay
	}

	return calculated
}

// 返回指定对象的请求次数
func (r *ItemExponentialFailureRateLimiter) NumRequeues(item interface{}) int {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	return r.failures[item]
}

// 删除一个对象
func (r *ItemExponentialFailureRateLimiter) Forget(item interface{}) {
	r.failuresLock.Lock()
	defer r.failuresLock.Unlock()

	delete(r.failures, item)
}
```

ItemFastSlowRateLimiter
---
快慢限速器，可以从实现中看到，当尝试次数小于`maxFastAttempts`时，`when`值返回的是`fastDelay`时间，否则，返回的是`slowDelay`时间
```go
// ItemFastSlowRateLimiter does a quick retry for a certain number of attempts, then a slow retry after that
type ItemFastSlowRateLimiter struct {
	failuresLock sync.Mutex
	failures     map[interface{}]int

	maxFastAttempts int
	fastDelay       time.Duration
	slowDelay       time.Duration
}

var _ RateLimiter = &ItemFastSlowRateLimiter{}

func NewItemFastSlowRateLimiter(fastDelay, slowDelay time.Duration, maxFastAttempts int) RateLimiter {
    return &ItemFastSlowRateLimiter{
        failures:        map[interface{}]int{},
        fastDelay:       fastDelay,
        slowDelay:       slowDelay,
        maxFastAttempts: maxFastAttempts,
    }
}

func (r *ItemFastSlowRateLimiter) When(item interface{}) time.Duration {
    r.failuresLock.Lock()
    defer r.failuresLock.Unlock()
    
    r.failures[item] = r.failures[item] + 1
    
    if r.failures[item] <= r.maxFastAttempts {
        return r.fastDelay
    }
    
    return r.slowDelay
}
```


MaxOfRateLimiter
---
该类型里面以数组的形式存放了多个限速器对象，并且在执行`when`方法的时候返回延迟最大的那个，其他方法的实现也是类似
```go
type MaxOfRateLimiter struct {
	limiters []RateLimiter
}

func (r *MaxOfRateLimiter) When(item interface{}) time.Duration {
	ret := time.Duration(0)
	for _, limiter := range r.limiters {
		curr := limiter.When(item)
		if curr > ret {
			ret = curr
		}
	}

	return ret
}
```
结语
---
今天我们看了`workqueue`包中限速器的几种实现，他们主要在限速队列那边被使用，限速队列会在之后的文章提及。



