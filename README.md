# examples-redigo

Examples on using redigo library. First file contains initialized redis pool
that is stored in `Pool` variable. Pool connects by default `localhost:6379`.
This can be changed by passing `REDIS_HOST` environment variable.

**redis/pool.go** file

```go
package redis

import (
    "github.com/garyburd/redigo/redis"
    "os"
    "os/signal"
    "syscall"
    "time"
)

var (
    Pool *redis.Pool
)

func init() {
    redisHost := os.Getenv("REDIS_HOST")
    if redisHost == "" {
        redisHost = ":6379"
    }
    Pool = newPool(redisHost)
    cleanupHook()
}

func newPool(server string) *redis.Pool {

    return &redis.Pool{

        MaxIdle:     3,
        IdleTimeout: 240 * time.Second,

        Dial: func() (redis.Conn, error) {
            c, err := redis.Dial("tcp", server)
            if err != nil {
                return nil, err
            }
            return c, err
        },

        TestOnBorrow: func(c redis.Conn, t time.Time) error {
            _, err := c.Do("PING")
            return err
        },
    }
}

func cleanupHook() {

    c := make(chan os.Signal, 1)
    signal.Notify(c, os.Interrupt)
    signal.Notify(c, syscall.SIGTERM)
    signal.Notify(c, syscall.SIGKILL)
    go func() {
        <-c
        Pool.Close()
        os.Exit(0)
    }()
}
```

Util file provides basic (most common) redis operations. This is supposed to be
an example not full list of all operations.

**redis/util.go** file

```go
package redis

import (
    "fmt"
    "github.com/garyburd/redigo/redis"
)

func Ping() error {

    conn := Pool.Get()
    defer conn.Close()

    _, err := redis.String(conn.Do("PING"))
    if err != nil {
        return fmt.Errorf("cannot 'PING' db: %v", err)
    }
    return nil
}

func Get(key string) ([]byte, error) {

    conn := Pool.Get()
    defer conn.Close()

    var data []byte
    data, err := redis.Bytes(conn.Do("GET", key))
    if err != nil {
        return data, fmt.Errorf("error getting key %s: %v", key, err)
    }
    return data, err
}

func Set(key string, value []byte) error {

    conn := Pool.Get()
    defer conn.Close()

    _, err := conn.Do("SET", key, value)
    if err != nil {
        v := string(value)
        if len(v) > 15 {
            v = v[0:12] + "..."
        }
        return fmt.Errorf("error setting key %s to %s: %v", key, v, err)
    }
    return err
}

func Exists(key string) (bool, error) {

    conn := Pool.Get()
    defer conn.Close()

    ok, err := redis.Bool(conn.Do("EXISTS", key))
    if err != nil {
        return ok, fmt.Errorf("error checking if key %s exists: %v", key, err)
    }
    return ok, err
}

func Delete(key string) error {

    conn := Pool.Get()
    defer conn.Close()

    _, err := conn.Do("DEL", key)
    return err
}

func GetKeys(pattern string) ([]string, error) {

    conn := Pool.Get()
    defer conn.Close()

    iter := 0
    keys := []string{}
    for {
        arr, err := redis.Values(conn.Do("SCAN", iter, "MATCH", pattern))
        if err != nil {
            return keys, fmt.Errorf("error retrieving '%s' keys", pattern)
        }

        iter, _ = redis.Int(arr[0], nil)
        k, _ := redis.Strings(arr[1], nil)
        keys = append(keys, k...)

        if iter == 0 {
            break
        }
    }

    return keys, nil
}

func Incr(counterKey string) (int, error) {

    conn := Pool.Get()
    defer conn.Close()

    return redis.Int(conn.Do("INCR", counterKey))
}
```
