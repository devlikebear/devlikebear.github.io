---
layout: post
title: "go-redis Proxy 모듈 구현"
categories: golang go-redis redis
---

## go-redis Proxy를 구현하는 이유

go-redis Proxy는 왜 작성할까요?

그냥 바로 go-redis 패키지를 import해서 사용해도 됩니다.

하지만 저는 가급적 3rd-party 패키지에 의존하는 코드를 구현하는 코드에 드러내는 걸 좋아하지 않습니다.

이유는 이후 해당 패키지를 다른 패키지로 변경하거나, 버전업의 이유로 업그레이드를 할 때, 직접 import한 모듈이 여기저기 흩어져 있을 경우 수정하기가 어렵기 때문입니다.

Proxy 패턴을 사용해 실제 모듈은 Proxy 모듈의 뒤로 숨겨두고 사용은 Proxy 인터페이스를 사용하는 방식으로 작성합니다. 

## go-redis 의존성 추가

redis 서버 버전이 7 이상일 경우 goredis는 v9을 사용해야 합니다.

```bash
go get github.com/go-redis/redis/v9
```

6 이하일 경우에는 v8을 사용합니다.

## 테스트 코드 작성

먼저 테스트코드를 작성합니다.

```go
package redisproxy_test

import (
	"context"
	"testing"
	"time"

	"github.com/stretchr/testify/assert"
	"www.devforest.net/pkg/redisproxy"
)

var (
	ctx        context.Context
	expiration time.Duration
	key, value string
	redis      redisproxy.Proxy
)

func init() {
	ctx = context.Background()
	expiration = 10 * time.Second
	key, value = "foo", "bar"

	conf := &redisproxy.Config{
		Addr: "localhost:6379",
	}
	redisproxy.Init(conf)
	redis = redisproxy.GetRedis()
}

func TestGet(t *testing.T) {
	// given
	assert.NoError(t, redis.Set(ctx, key, value, expiration))

	// when
	v, err := redis.Get(ctx, key)

	// then
	assert.NoError(t, err)
	assert.Equal(t, v, value)
}

func TestSet(t *testing.T) {
	// given
	redis.Del(ctx, key)
	_, err := redis.Get(ctx, key)
	assert.Equal(t, redisproxy.ErrNotFound, err)

	// when
	assert.NoError(t, redis.Set(ctx, key, value, expiration))
	v, err := redis.Get(ctx, key)

	// then
	assert.NoError(t, err)
	assert.Equal(t, v, value)
}
```

Get()과 Set() 기능을 테스트하는 코드를 작성합니다.

## Redis Proxy 인터페이스 정의

redisproxy.Proxy는 다음과 같이 Redis 인터페이스를 정의합니다.

```go
package redisproxy

import (
	"context"
	"errors"
	"time"
)

var (
	// ErrNotFound 에러
	ErrNotFound = errors.New("not found")
)

// Proxy Redis Proxy 인터페이스 정의
type Proxy interface {
	// Get 키에 해당하는 값을 조회
	Get(ctx context.Context, key string) (string, error)

	// Set 키에 해당하는 값을 저장
	// 유효시간을 설정할 경우 해당 시간 이후 삭제된다.
	Set(ctx context.Context, key, value string, expiration time.Duration) error

	// Del 지정된 키들 삭제
	// 성공할 경우 삭제된 키의 개수를 반환한다.
	Del(ctx context.Context, keys ...string) (int, error)
}
```

지금은 단순히 키&값을 저장하고 조회하는 기능만 있으면 되기때문에 Get(), Set(), Del() 함수만 포함하지만, 추후 해시를 다룰 필요가 있을 경우 Proxy 인터페이스를 확장할 예정입니다.

## Proxy 인터페이스 구현

Proxy 인터페이스를 구현하는 코드를 다음과 같이 작성합니다.

```go
package redisproxy

import (
	"context"
	"sync"
	"time"

	"github.com/go-redis/redis/v9"
)

var (
	once       sync.Once
	redisProxy Proxy
)

type proxy struct {
	client *redis.Client
}

func Init(conf ...*Config) {
	once.Do(func() {
		if len(conf) != 1 {
			panic("not given redis config")
		}

		redisConf := conf[0]

		client := redis.NewClient(&redis.Options{
			Addr:     redisConf.Addr,
			Password: redisConf.Password,
			DB:       redisConf.DB,
		})
		redisProxy = &proxy{
			client: client,
		}
	})
}

func GetRedis() Proxy {
	return redisProxy
}

func (clt proxy) Get(ctx context.Context, key string) (string, error) {
	strCmd := clt.client.Get(ctx, key)
	if strCmd.Err() != nil {
		if strCmd.Err() == redis.Nil {
			return "", ErrNotFound
		}
		return "", strCmd.Err()
	}

	return strCmd.Val(), nil
}

func (clt proxy) Set(ctx context.Context, key, value string, expiration time.Duration) error {
	statusCmd := clt.client.Set(ctx, key, value, expiration)
	if statusCmd.Err() != nil {
		return statusCmd.Err()
	}

	return nil
}

func (clt proxy) Del(ctx context.Context, keys ...string) (int, error) {
	intCmd := clt.client.Del(ctx, keys...)
	if intCmd.Err() != nil {
		return 0, intCmd.Err()
	}

	return int(intCmd.Val()), nil
}
```

Redis 서버에 연결할 때 사용할 설정을 다루는 Config도 go-redis를 노출하지 않기 위해 다음과 같이 Config 구조체를 만들어 정의합니다.

```go
package redisproxy

// Config Redis 서버 연결 설정
type Config struct {
	Addr     string	// 서버 주소. 예) localhost:6379
	Password string	// 서버 연결 암호
	DB       int	// DB 번호
}
```
