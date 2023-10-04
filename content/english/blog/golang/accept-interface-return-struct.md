---
title: "Accept interface return struct"
meta_title: ""
description: "The why and how of Accept interface return struct"
date: 2023-10-03T05:00:00Z
categories: ["golang", "design", "interface"]
author: "Hugo Sjoberg"
tags: ["golang", "design", "interface"]
draft: false
---

# Accept interface return struct

The term `Accept interface return struct` was first coined by Jack Lindamood in this [article](https://medium.com/@cep21/preemptive-interface-anti-pattern-in-go-54c18ac0668a).

# Real world example

Let's use an example where a we set a user session in a cache and get a user session in a cache. The cache in this case can be what-ever, Redis, Memcached, Postgres(please don't), in-memory etc. But from a session perspective we want to be able to set the session and get the session.

# Bad example

Let's first see a bad example which causes strong coupling of the storage medium(A pattern I'm also guilty of using)

```golang
// cache/cache.go
type Cache struct {
	client *redis.Client
}

func (c *Cache) Set(key string, value string) {
	ctx := context.Background()
	c.client.Set(ctx, key, value, 0)
}

func (c *Cache) Get(key string) string {
	ctx := context.Background()
	value, _ := c.client.Get(ctx, key).Result()
	return value
}

func NewCache() *Cache {
	return &Cache{redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "", // no password set
		DB:       0,  // use default DB
	})}
}
```

And the session package using the cache:

```golang
// session/session.go
type SessionService struct {
	cache *cache.Cache
}

func NewSessionService(c *cache.Cache) *SessionService {
	return &SessionService{
		cache: c,
	}
}

func (s *SessionService) SetSession(user string, session string) {
	s.cache.Set(user, session)
}

func (s *SessionService) GetSession(user string) string {
	return s.cache.Get(user)
}
```

Here the interface is defined by the producer(cache) instead of defined by the session. Which means the producer is preemptively defining an interface before it is actually being used, this is called *preemptive interface*.

This is a fairly common pattern seen across go code, it's not ncessarily bad but I think we can do it differently which will enable us to easier switch out the cache for a different type of cache if needed as well as doing some easy mocking.

# Good Example

The cache-package will in this case be the producer and provide a service, in this case the service is to set and get a session

The session-package will in this case be the consumer of the service.

Let's look at an example implementation of the cache and session:

We create a consumer which accepts an interface and returns a struct.

```golang
// session/session.go

type SessionStorer interface {
	Get(string) string
	Set(string, string)
}

type SessionService struct {
	store SessionStorer
}

func NewSessionService(s SessionStorer) *SessionService {
	return &SessionService{
		store: s,
	}
}

func (s *SessionService) SetSession(user string, session string) {
	s.store.Set(user, session)
}

func (s *SessionService) GetSession(user string) string {
	return s.store.Get(user)
}
```

Lets create a cache which acts as the provider, returning a struct with concrete types.

```golang
// cache/cache.go

type Cache struct {
	mu    sync.RWMutex
	cache map[string]string
}

func (c *Cache) Set(key string, value string) {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.cache[key] = value
}

func (c *Cache) Get(key string) string {
	c.mu.RLock()
	defer c.mu.RUnlock()
	value, _ := c.cache[key]
	return value
}

func NewCache() *Cache {
	return &Cache{
		cache: make(map[string]string),
	}
}
```

Using the two packages together can look something like this:

```golang
func main() {
	c := cache.NewCache()
	s := session.NewSessionService(c)

	s.SetSession("john", "john-session")
	fmt.Println(s.GetSession("john"))
}
```

The session-service accepts an interface which consists of two functions `Get` and `Set` which is something which the cache provided(Cool!).

This is a pretty novel example showing what *Accepting Interfaces* is all about, letting the consumer define what they want in an interface. By using this pattern we get some pretty sweet gains:

- Easy to test, we can easily change the cache so we count how many times `Set` and `Get` has been called which is great in testing. We can easily change the cache type from Redis to in-memory which also simplifies testing.
- Looser coupling, consumers are not coupled with their dependencies. The consumer can use what-ever storage, this helps us if we want to change storage type in the future.

# Code

All the code used in the blog can be found here:

https://github.com/hugosjoberg/blog-code/tree/main/interface-struct

https://github.com/hugosjoberg/blog-code/tree/main/interface-struct-bad-example



# Additional resources
[The official Go repo review-guide](https://github.com/golang/go/wiki/CodeReviewComments#interfaces)
