---
name: scaffolding-backend-clean-architecture 
description: Use when user asks to scaffold, create, initialize, or set up a project with clean architecture folder structure. 
Triggers on: "clean architecture", "scaffold project", "create folder structure", "set up layers", "domain application infrastructure folders".
---

# Clean Architecture Scaffold

## Overview

Scaffold a project directory following clean architecture — three primary layers: `domain`, `application`, `infrastructure`. Dependencies flow inward: infrastructure depends on application, application depends on domain, domain depends on nothing.

----------

## Folder Structure

Folders are split into **core** (always created) and **conditional** (created only when the matching infrastructure is selected).

```
<project-root>/
├── domain/                          # innermost layer, zero framework dependencies
│   ├── entities/                    # business objects that carry identity and state
│   └── rules/                       # business rules: both concrete logic and domain interfaces
│
├── application/                     # orchestration layer — wires domain to infrastructure via interfaces
│   ├── usecases/                    # one file per use-case operation
│   ├── services/                    # application-level service interfaces (ports)
│   └── events/                      # [conditional: Publisher OR Subscriber] event payload structs
│
└── infrastructure/                  # outermost layer — all framework, DB, and I/O code
    ├── api/                         # [core] HTTP layer (e.g. Go Fiber)
    ├── configuration/               # [core] reads .env via Viper into AppConfig singleton
    ├── dependencies/                # [core] dependency injection wiring (e.g. Google Wire)
    ├── logging/                     # [core] structured logging setup
    ├── metric/                      # [core] metrics infrastructure
    ├── tracer/                      # [core] OTel TracerProvider with Jaeger OTLP HTTP exporter
    ├── security/                    # [core] auth, JWT, middleware guards
    ├── resiliency/                  # [core] circuit breaker, retry, timeout policies
    ├── validator/                   # [core] input validation logic
    ├── workers/                     # [core] background worker runners
    │
    ├── database/                    # [conditional: Database] — see substructure below
    │   ├── clients/                 # client interfaces (one per selected DB)
    │   │   └── mocks/              # testify mocks generated from client interfaces
    │   ├── connectors/              # concrete implementations of client interfaces
    │   ├── models/                  # ORM / document data model structs
    │   └── repositories/           # repository implementations (one file per entity×DB)
    │
    ├── cache/                       # [conditional: Cache] cache client initialization
    ├── publisher/                   # [conditional: Publisher] message publishing adapters
    ├── subscriber/                  # [conditional: Subscriber] message consuming adapters
    ├── queue/                       # [conditional: Queue] task queue adapters
    ├── workflow/                    # [conditional: Workflow] workflow engine client (e.g. Temporal)
    └── integrations/                # [conditional: Integrations] third-party API adapters

```

----------

## Steps

### Step 1 — Gather project info

Ask the user (if not already provided):

-   **Go module name** — used in all import paths (e.g. `mycompany/myservice`). Default: `myservice`.
-   **Base directory** — where to create the project root (e.g. `.`, `./myapp`). Default: `.`.

### Step 2 — Select infrastructure components

Present as a checklist. Display all option for each infrastructure category. Show each infrastructure category at distinct page. Record every selection — it determines which folders and files are created.

#### Database
| Option |       Name       | 
|:------:|------------------|
| **D1** | PostgreSQL       |
| **D2** | FerretDB         |
| **D3** | Redis            |
| **D4** | YugabyteDB       |
| **D5** | Elasticsearch    |
| **D6** | AerospikeDB      |


#### Cache
| Option |       Name       | 
|:------:|------------------|
| **C1** | Redis       	    |
| **C2** | Memcache         |


#### Publisher
| Option |       Name       | 
|:------:|------------------|
| **P1** | RabbitMQ         |
| **P2** | Redis            |


#### Subscriber
| Option |       Name       | 
|:------:|------------------|
| **S1** | RabbitMQ         |
| **S2** | Redis            |

#### Queue
| Option |       Name       | 
|:------:|------------------|
| **Q1** | RabbitMQ         |
| **Q2** | Asynq            |

#### Workflow
| Option |       Name       | 
|:------:|------------------|
| **W1** | Temporal         |

#### Integrations
| Option |       Name       | 
|:------:|------------------|
| **I1**  | Yes             |
| **I2**  | No              |


### Step 3 — Create folders

**Always create** (core):

```
domain/entities
domain/rules
application/usecases
application/services
infrastructure/api
infrastructure/configuration
infrastructure/dependencies
infrastructure/logging
infrastructure/metric
infrastructure/tracer
infrastructure/security
infrastructure/resiliency
infrastructure/validator
infrastructure/workers

```

**Create only if selected** (conditional):

Condition

Folders to create

Any Database selected

`infrastructure/database/clients` `infrastructure/database/clients/mocks` `infrastructure/database/connectors` `infrastructure/database/models` `infrastructure/database/repositories`

Cache selected

`infrastructure/cache`

Publisher selected

`infrastructure/publisher`

Subscriber selected

`infrastructure/subscriber`

Publisher OR Subscriber selected

`application/events`

Queue selected

`infrastructure/queue`

Workflow selected

`infrastructure/workflow`

Integrations selected

`infrastructure/integrations`

### Step 4 — Generate boilerplate files

For each selected database, create the 4 boilerplate files described in the **Database Boilerplate** section below.  
Replace `{module}` with the Go module name from Step 1.  
Place a `.gitkeep` in every leaf folder that has no generated file yet.

### Step 5 — Confirm

Print a `tree`-style output of the created structure so the user can verify it matches expectations.

----------

## Database Boilerplate

For each selected database, generate exactly **4 files**. All templates use `{module}` as the Go module name placeholder.

----------

### PostgreSQL

**`infrastructure/database/postgresql.go`**

```go
package database

import (
	"fmt"
	"log"
	"{module}/infrastructure/configuration"
	"strings"

	"github.com/uptrace/opentelemetry-go-extra/otelgorm"
	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/schema"
)

var PostgresqlClient *gorm.DB

func InitializePostgresql() {
	connstring := fmt.Sprintf("host=%s user=%s password=%s dbname=%s port=%d sslmode=disable",
		configuration.AppConfig.PostgreSQLHost,
		configuration.AppConfig.PostgreSQLUsername,
		configuration.AppConfig.PostgreSQLPassword,
		configuration.AppConfig.PostgreSQLDatabase,
		configuration.AppConfig.PostgreSQLPort,
	)
	db, err := gorm.Open(postgres.Open(connstring), &gorm.Config{
		NamingStrategy: schema.NamingStrategy{
			SingularTable: false,
			NameReplacer:  strings.NewReplacer("DataModel", ""),
		},
	})
	if err != nil {
		log.Fatal(err)
	} else {
		if err := db.Use(otelgorm.NewPlugin(otelgorm.WithDBName(configuration.AppConfig.PostgreSQLDatabase))); err != nil {
			log.Fatal(err)
		} else {
			PostgresqlClient = db
			fmt.Printf("Connecting To PostgreSQL : %+v \n", db)
		}
	}
}

```

**`infrastructure/database/clients/PostgresClient.go`**

```go
package clients

import "context"

type PostgresClient interface {
	// Create persists the given value
	Create(ctx context.Context, value interface{}) error

	// First loads the first matching record into dest
	First(ctx context.Context, dest interface{}, query interface{}, args ...interface{}) error

	// Save updates the given value
	Save(ctx context.Context, value interface{}) error
}

```

**`infrastructure/database/connectors/PostgresConnector.go`**

```go
package connectors

import (
	"context"

	"gorm.io/gorm"
)

type PostgresConnector struct {
	client *gorm.DB
}

func NewPostgresConnector(client *gorm.DB) *PostgresConnector {
	return &PostgresConnector{client: client}
}

func (c *PostgresConnector) Create(ctx context.Context, value interface{}) error {
	return c.client.WithContext(ctx).Create(value).Error
}

func (c *PostgresConnector) First(ctx context.Context, dest interface{}, query interface{}, args ...interface{}) error {
	if query == nil {
		return c.client.WithContext(ctx).First(dest, args...).Error
	}
	conds := append([]interface{}{query}, args...)
	return c.client.WithContext(ctx).First(dest, conds...).Error
}

func (c *PostgresConnector) Save(ctx context.Context, value interface{}) error {
	return c.client.WithContext(ctx).Save(value).Error
}

```

**`infrastructure/database/clients/mocks/MockPostgresClient.go`**

```go
// Code generated by mockery v2.53.5. DO NOT EDIT.

package clientmocks

import (
	context "context"

	mock "github.com/stretchr/testify/mock"
)

type MockPostgresClient struct {
	mock.Mock
}

type PostgresClient_Expecter struct {
	mock *mock.Mock
}

func (_m *MockPostgresClient) EXPECT() *PostgresClient_Expecter {
	return &PostgresClient_Expecter{mock: &_m.Mock}
}

func (_m *MockPostgresClient) Create(ctx context.Context, value interface{}) error {
	ret := _m.Called(ctx, value)
	if len(ret) == 0 {
		panic("no return value specified for Create")
	}
	var r0 error
	if rf, ok := ret.Get(0).(func(context.Context, interface{}) error); ok {
		r0 = rf(ctx, value)
	} else {
		r0 = ret.Error(0)
	}
	return r0
}

type PostgresClient_Create_Call struct{ *mock.Call }

func (_e *PostgresClient_Expecter) Create(ctx interface{}, value interface{}) *PostgresClient_Create_Call {
	return &PostgresClient_Create_Call{Call: _e.mock.On("Create", ctx, value)}
}

func (_c *PostgresClient_Create_Call) Run(run func(ctx context.Context, value interface{})) *PostgresClient_Create_Call {
	_c.Call.Run(func(args mock.Arguments) { run(args[0].(context.Context), args[1]) })
	return _c
}

func (_c *PostgresClient_Create_Call) Return(_a0 error) *PostgresClient_Create_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_c *PostgresClient_Create_Call) RunAndReturn(run func(context.Context, interface{}) error) *PostgresClient_Create_Call {
	_c.Call.Return(run)
	return _c
}

func (_m *MockPostgresClient) First(ctx context.Context, dest interface{}, query interface{}, args ...interface{}) error {
	var _ca []interface{}
	_ca = append(_ca, ctx, dest, query)
	_ca = append(_ca, args...)
	ret := _m.Called(_ca...)
	if len(ret) == 0 {
		panic("no return value specified for First")
	}
	var r0 error
	if rf, ok := ret.Get(0).(func(context.Context, interface{}, interface{}, ...interface{}) error); ok {
		r0 = rf(ctx, dest, query, args...)
	} else {
		r0 = ret.Error(0)
	}
	return r0
}

type PostgresClient_First_Call struct{ *mock.Call }

func (_e *PostgresClient_Expecter) First(ctx interface{}, dest interface{}, query interface{}, args ...interface{}) *PostgresClient_First_Call {
	return &PostgresClient_First_Call{Call: _e.mock.On("First", append([]interface{}{ctx, dest, query}, args...)...)}
}

func (_c *PostgresClient_First_Call) Run(run func(ctx context.Context, dest interface{}, query interface{}, args ...interface{})) *PostgresClient_First_Call {
	_c.Call.Run(func(args mock.Arguments) {
		variadicArgs := make([]interface{}, len(args)-3)
		for i, a := range args[3:] {
			variadicArgs[i] = a
		}
		run(args[0].(context.Context), args[1], args[2], variadicArgs...)
	})
	return _c
}

func (_c *PostgresClient_First_Call) Return(_a0 error) *PostgresClient_First_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_c *PostgresClient_First_Call) RunAndReturn(run func(context.Context, interface{}, interface{}, ...interface{}) error) *PostgresClient_First_Call {
	_c.Call.Return(run)
	return _c
}

func (_m *MockPostgresClient) Save(ctx context.Context, value interface{}) error {
	ret := _m.Called(ctx, value)
	if len(ret) == 0 {
		panic("no return value specified for Save")
	}
	var r0 error
	if rf, ok := ret.Get(0).(func(context.Context, interface{}) error); ok {
		r0 = rf(ctx, value)
	} else {
		r0 = ret.Error(0)
	}
	return r0
}

type PostgresClient_Save_Call struct{ *mock.Call }

func (_e *PostgresClient_Expecter) Save(ctx interface{}, value interface{}) *PostgresClient_Save_Call {
	return &PostgresClient_Save_Call{Call: _e.mock.On("Save", ctx, value)}
}

func (_c *PostgresClient_Save_Call) Run(run func(ctx context.Context, value interface{})) *PostgresClient_Save_Call {
	_c.Call.Run(func(args mock.Arguments) { run(args[0].(context.Context), args[1]) })
	return _c
}

func (_c *PostgresClient_Save_Call) Return(_a0 error) *PostgresClient_Save_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_c *PostgresClient_Save_Call) RunAndReturn(run func(context.Context, interface{}) error) *PostgresClient_Save_Call {
	_c.Call.Return(run)
	return _c
}

func NewPostgresClient(t interface {
	mock.TestingT
	Cleanup(func())
}) *MockPostgresClient {
	mock := &MockPostgresClient{}
	mock.Mock.Test(t)
	t.Cleanup(func() { mock.AssertExpectations(t) })
	return mock
}

```

----------

### FerretDB

**`infrastructure/database/ferretdb.go`**

```go
package database

import (
	"context"
	"fmt"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"go.opentelemetry.io/contrib/instrumentation/go.mongodb.org/mongo-driver/mongo/otelmongo"
)

var FerretDBClient *mongo.Client
var FerretDBDatabase *mongo.Database

func InitializeFerretDB() {
	uri := "mongodb://username:password@localhost:27017"
	opts := options.Client()
	opts.Monitor = otelmongo.NewMonitor()
	opts.ApplyURI(uri)

	client, err := mongo.Connect(context.TODO(), opts)
	if err != nil {
		fmt.Println("=== Load Database FerretDB ===")
		fmt.Printf("Error Connecting To FerretDB : %+v \n", err)
		fmt.Println("=== End Load Database FerretDB ===")
	} else {
		FerretDBClient = client
		FerretDBDatabase = FerretDBClient.Database("your-database-name")
		fmt.Println("=== Load Database FerretDB ===")
		fmt.Printf("Connecting To FerretDB : %+v \n", FerretDBClient)
		fmt.Println("=== End Load Database FerretDB ===")
	}
}

func CloseFerretDB() {
	if err := FerretDBClient.Disconnect(context.TODO()); err != nil {
		panic(err)
	}
}

```

**`infrastructure/database/clients/MongoClient.go`**

```go
package clients

import (
	"context"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

type MongoClient interface {
	InsertOne(ctx context.Context, collectionName string, doc interface{}) (*mongo.InsertOneResult, error)
	FindOne(ctx context.Context, collectionName string, filter interface{}, opts ...*options.FindOneOptions) *mongo.SingleResult
	UpdateOne(ctx context.Context, collectionName string, filter interface{}, update interface{}) (*mongo.UpdateResult, error)
}

```

**`infrastructure/database/connectors/MongoConnector.go`**

```go
package connectors

import (
	"context"

	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

type MongoConnector struct {
	client *mongo.Database
}

func NewMongoConnector(client *mongo.Database) *MongoConnector {
	return &MongoConnector{client: client}
}

func (m *MongoConnector) InsertOne(ctx context.Context, collectionName string, doc interface{}) (*mongo.InsertOneResult, error) {
	return m.client.Collection(collectionName).InsertOne(ctx, doc)
}

func (m *MongoConnector) FindOne(ctx context.Context, collectionName string, filter interface{}, opts ...*options.FindOneOptions) *mongo.SingleResult {
	return m.client.Collection(collectionName).FindOne(ctx, filter, opts...)
}

func (m *MongoConnector) UpdateOne(ctx context.Context, collectionName string, filter interface{}, update interface{}) (*mongo.UpdateResult, error) {
	return m.client.Collection(collectionName).UpdateOne(ctx, filter, update)
}

```

**`infrastructure/database/clients/mocks/MockMongoClient.go`**

```go
package clientmocks

import (
	"context"

	clients "{module}/infrastructure/database/clients"

	"github.com/stretchr/testify/mock"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

type MockMongoClient struct{ mock.Mock }

type MockMongoClient_Expecter struct{ mock *mock.Mock }

func (_m *MockMongoClient) EXPECT() *MockMongoClient_Expecter {
	return &MockMongoClient_Expecter{mock: &_m.Mock}
}

func (_m *MockMongoClient) InsertOne(ctx context.Context, collectionName string, doc interface{}) (*mongo.InsertOneResult, error) {
	ret := _m.Called(ctx, collectionName, doc)
	var r0 *mongo.InsertOneResult
	var r1 error
	if rf, ok := ret.Get(0).(func(context.Context, string, interface{}) (*mongo.InsertOneResult, error)); ok {
		return rf(ctx, collectionName, doc)
	}
	if rf, ok := ret.Get(0).(func(context.Context, string, interface{}) *mongo.InsertOneResult); ok {
		r0 = rf(ctx, collectionName, doc)
	} else if ret.Get(0) != nil {
		r0 = ret.Get(0).(*mongo.InsertOneResult)
	}
	if rf, ok := ret.Get(1).(func(context.Context, string, interface{}) error); ok {
		r1 = rf(ctx, collectionName, doc)
	} else {
		r1 = ret.Error(1)
	}
	return r0, r1
}

type MockMongoClient_InsertOne_Call struct{ *mock.Call }

func (_e *MockMongoClient_Expecter) InsertOne(ctx, collectionName, doc interface{}) *MockMongoClient_InsertOne_Call {
	return &MockMongoClient_InsertOne_Call{Call: _e.mock.On("InsertOne", ctx, collectionName, doc)}
}

func (_c *MockMongoClient_InsertOne_Call) Run(run func(ctx context.Context, collectionName string, doc interface{})) *MockMongoClient_InsertOne_Call {
	_c.Call.Run(func(args mock.Arguments) { run(args[0].(context.Context), args[1].(string), args[2]) })
	return _c
}

func (_c *MockMongoClient_InsertOne_Call) Return(_a0 *mongo.InsertOneResult, _a1 error) *MockMongoClient_InsertOne_Call {
	_c.Call.Return(_a0, _a1)
	return _c
}

func (_c *MockMongoClient_InsertOne_Call) RunAndReturn(run func(context.Context, string, interface{}) (*mongo.InsertOneResult, error)) *MockMongoClient_InsertOne_Call {
	_c.Call.Return(run)
	return _c
}

func (_m *MockMongoClient) FindOne(ctx context.Context, collectionName string, filter interface{}, opts ...*options.FindOneOptions) *mongo.SingleResult {
	var callArgs []interface{}
	callArgs = append(callArgs, ctx, collectionName, filter)
	for _, o := range opts {
		callArgs = append(callArgs, o)
	}
	ret := _m.Called(callArgs...)
	var r0 *mongo.SingleResult
	if rf, ok := ret.Get(0).(func(context.Context, string, interface{}, ...*options.FindOneOptions) *mongo.SingleResult); ok {
		return rf(ctx, collectionName, filter, opts...)
	}
	if ret.Get(0) != nil {
		r0 = ret.Get(0).(*mongo.SingleResult)
	}
	return r0
}

type MockMongoClient_FindOne_Call struct{ *mock.Call }

func (_e *MockMongoClient_Expecter) FindOne(ctx, collectionName, filter interface{}, opts ...interface{}) *MockMongoClient_FindOne_Call {
	return &MockMongoClient_FindOne_Call{Call: _e.mock.On("FindOne", append([]interface{}{ctx, collectionName, filter}, opts...)...)}
}

func (_c *MockMongoClient_FindOne_Call) Return(_a0 *mongo.SingleResult) *MockMongoClient_FindOne_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_m *MockMongoClient) UpdateOne(ctx context.Context, collectionName string, filter interface{}, update interface{}) (*mongo.UpdateResult, error) {
	ret := _m.Called(ctx, collectionName, filter, update)
	var r0 *mongo.UpdateResult
	var r1 error
	if rf, ok := ret.Get(0).(func(context.Context, string, interface{}, interface{}) (*mongo.UpdateResult, error)); ok {
		return rf(ctx, collectionName, filter, update)
	}
	if rf, ok := ret.Get(0).(func(context.Context, string, interface{}, interface{}) *mongo.UpdateResult); ok {
		r0 = rf(ctx, collectionName, filter, update)
	} else if ret.Get(0) != nil {
		r0 = ret.Get(0).(*mongo.UpdateResult)
	}
	if rf, ok := ret.Get(1).(func(context.Context, string, interface{}, interface{}) error); ok {
		r1 = rf(ctx, collectionName, filter, update)
	} else {
		r1 = ret.Error(1)
	}
	return r0, r1
}

type MockMongoClient_UpdateOne_Call struct{ *mock.Call }

func (_e *MockMongoClient_Expecter) UpdateOne(ctx, collectionName, filter, update interface{}) *MockMongoClient_UpdateOne_Call {
	return &MockMongoClient_UpdateOne_Call{Call: _e.mock.On("UpdateOne", ctx, collectionName, filter, update)}
}

func (_c *MockMongoClient_UpdateOne_Call) Return(_a0 *mongo.UpdateResult, _a1 error) *MockMongoClient_UpdateOne_Call {
	_c.Call.Return(_a0, _a1)
	return _c
}

var _ clients.MongoClient = (*MockMongoClient)(nil)

func NewMockMongoClient(t interface {
	mock.TestingT
	Cleanup(func())
}) *MockMongoClient {
	m := &MockMongoClient{}
	m.Mock.Test(t)
	t.Cleanup(func() { m.AssertExpectations(t) })
	return m
}

```

----------

### Redis

**`infrastructure/database/redis.go`**

```go
package database

import (
	"fmt"

	"github.com/redis/go-redis/extra/redisotel/v9"
	"github.com/redis/go-redis/v9"
)

var RedisClient *redis.Client

func InitializeRedis() {
	RedisClient = redis.NewClient(&redis.Options{
		Addr:     "localhost:6379",
		Password: "",
		DB:       0,
	})

	if err := redisotel.InstrumentTracing(RedisClient); err != nil {
		panic(err)
	}

	if err := redisotel.InstrumentMetrics(RedisClient); err != nil {
		panic(err)
	}

	fmt.Println("Redis Is Running : ", RedisClient)
}

```

**`infrastructure/database/clients/RedisClient.go`**

```go
package clients

import (
	"context"

	"github.com/redis/go-redis/v9"
)

type RedisClient interface {
	JSONSet(ctx context.Context, key string, path string, value interface{}) error
	JSONGet(ctx context.Context, key string, path ...string) *redis.JSONCmd
}

```

**`infrastructure/database/connectors/RedisConnector.go`**

```go
package connectors

import (
	"context"

	"github.com/redis/go-redis/v9"
)

type RedisConnector struct {
	client *redis.Client
}

func NewRedisConnector(client *redis.Client) *RedisConnector {
	return &RedisConnector{client: client}
}

func (r *RedisConnector) JSONSet(ctx context.Context, key string, path string, value interface{}) error {
	return r.client.JSONSet(ctx, key, path, value).Err()
}

func (r *RedisConnector) JSONGet(ctx context.Context, key string, path ...string) *redis.JSONCmd {
	return r.client.JSONGet(ctx, key, path...)
}

```

**`infrastructure/database/clients/mocks/MockRedisClient.go`**

```go
// Code generated by mockery v2.53.5. DO NOT EDIT.

package clientmocks

import (
	context "context"

	clients "{module}/infrastructure/database/clients"

	mock "github.com/stretchr/testify/mock"
	"github.com/redis/go-redis/v9"
)

type MockRedisClient struct{ mock.Mock }

type RedisClient_Expecter struct{ mock *mock.Mock }

func (_m *MockRedisClient) EXPECT() *RedisClient_Expecter {
	return &RedisClient_Expecter{mock: &_m.Mock}
}

func (_m *MockRedisClient) JSONSet(ctx context.Context, key string, path string, value interface{}) error {
	ret := _m.Called(ctx, key, path, value)
	if len(ret) == 0 {
		panic("no return value specified for JSONSet")
	}
	var r0 error
	if rf, ok := ret.Get(0).(func(context.Context, string, string, interface{}) error); ok {
		r0 = rf(ctx, key, path, value)
	} else {
		r0 = ret.Error(0)
	}
	return r0
}

type RedisClient_JSONSet_Call struct{ *mock.Call }

func (_e *RedisClient_Expecter) JSONSet(ctx, key, path, value interface{}) *RedisClient_JSONSet_Call {
	return &RedisClient_JSONSet_Call{Call: _e.mock.On("JSONSet", ctx, key, path, value)}
}

func (_c *RedisClient_JSONSet_Call) Run(run func(ctx context.Context, key string, path string, value interface{})) *RedisClient_JSONSet_Call {
	_c.Call.Run(func(args mock.Arguments) { run(args[0].(context.Context), args[1].(string), args[2].(string), args[3]) })
	return _c
}

func (_c *RedisClient_JSONSet_Call) Return(_a0 error) *RedisClient_JSONSet_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_c *RedisClient_JSONSet_Call) RunAndReturn(run func(context.Context, string, string, interface{}) error) *RedisClient_JSONSet_Call {
	_c.Call.Return(run)
	return _c
}

func (_m *MockRedisClient) JSONGet(ctx context.Context, key string, path ...string) *redis.JSONCmd {
	var _ca []interface{}
	_ca = append(_ca, ctx, key)
	for _, p := range path {
		_ca = append(_ca, p)
	}
	ret := _m.Called(_ca...)
	var r0 *redis.JSONCmd
	if rf, ok := ret.Get(0).(func(context.Context, string, ...string) *redis.JSONCmd); ok {
		return rf(ctx, key, path...)
	}
	if ret.Get(0) != nil {
		r0 = ret.Get(0).(*redis.JSONCmd)
	}
	return r0
}

type RedisClient_JSONGet_Call struct{ *mock.Call }

func (_e *RedisClient_Expecter) JSONGet(ctx, key interface{}, path ...interface{}) *RedisClient_JSONGet_Call {
	return &RedisClient_JSONGet_Call{Call: _e.mock.On("JSONGet", append([]interface{}{ctx, key}, path...)...)}
}

func (_c *RedisClient_JSONGet_Call) Return(_a0 *redis.JSONCmd) *RedisClient_JSONGet_Call {
	_c.Call.Return(_a0)
	return _c
}

var _ clients.RedisClient = (*MockRedisClient)(nil)

func NewMockRedisClient(t interface {
	mock.TestingT
	Cleanup(func())
}) *MockRedisClient {
	m := &MockRedisClient{}
	m.Mock.Test(t)
	t.Cleanup(func() { m.AssertExpectations(t) })
	return m
}

```

----------

### YugabyteDB

**`infrastructure/database/yugabytedb.go`**

```go
package database

import (
	"log"
	"strings"

	"gorm.io/driver/postgres"
	"gorm.io/gorm"
	"gorm.io/gorm/schema"
)

var YugabyteDBClient *gorm.DB

const connYugabyteDB = "host=localhost user=yugabyte password=yugabyte dbname=yugabyte port=5433 sslmode=disable"

func InitializeYugabyteDB() {
	db, err := gorm.Open(postgres.Open(connYugabyteDB), &gorm.Config{
		NamingStrategy: schema.NamingStrategy{
			SingularTable: false,
			NameReplacer:  strings.NewReplacer("DataModel", ""),
		},
	})
	if err != nil {
		log.Fatal(err)
	} else {
		YugabyteDBClient = db
	}
}

```

**`infrastructure/database/clients/YugabyteClient.go`**

```go
package clients

import "context"

type YugabyteClient interface {
	// Create persists the given value
	Create(ctx context.Context, value interface{}) error

	// First loads the first matching record into dest
	First(ctx context.Context, dest interface{}, query interface{}, args ...interface{}) error

	// Save updates the given value
	Save(ctx context.Context, value interface{}) error
}

```

**`infrastructure/database/connectors/YugabyteConnector.go`**

```go
package connectors

import (
	"context"

	"gorm.io/gorm"
)

type YugabyteConnector struct {
	client *gorm.DB
}

func NewYugabyteConnector(client *gorm.DB) *YugabyteConnector {
	return &YugabyteConnector{client: client}
}

func (c *YugabyteConnector) Create(ctx context.Context, value interface{}) error {
	return c.client.WithContext(ctx).Create(value).Error
}

func (c *YugabyteConnector) First(ctx context.Context, dest interface{}, query interface{}, args ...interface{}) error {
	if query == nil {
		return c.client.WithContext(ctx).First(dest, args...).Error
	}
	conds := append([]interface{}{query}, args...)
	return c.client.WithContext(ctx).First(dest, conds...).Error
}

func (c *YugabyteConnector) Save(ctx context.Context, value interface{}) error {
	return c.client.WithContext(ctx).Save(value).Error
}

```

**`infrastructure/database/clients/mocks/MockYugabyteClient.go`**

```go
// Code generated by mockery v2.53.5. DO NOT EDIT.

package clientmocks

import (
	context "context"

	clients "{module}/infrastructure/database/clients"

	mock "github.com/stretchr/testify/mock"
)

type MockYugabyteClient struct{ mock.Mock }

type YugabyteClient_Expecter struct{ mock *mock.Mock }

func (_m *MockYugabyteClient) EXPECT() *YugabyteClient_Expecter {
	return &YugabyteClient_Expecter{mock: &_m.Mock}
}

func (_m *MockYugabyteClient) Create(ctx context.Context, value interface{}) error {
	ret := _m.Called(ctx, value)
	if len(ret) == 0 {
		panic("no return value specified for Create")
	}
	var r0 error
	if rf, ok := ret.Get(0).(func(context.Context, interface{}) error); ok {
		r0 = rf(ctx, value)
	} else {
		r0 = ret.Error(0)
	}
	return r0
}

type YugabyteClient_Create_Call struct{ *mock.Call }

func (_e *YugabyteClient_Expecter) Create(ctx, value interface{}) *YugabyteClient_Create_Call {
	return &YugabyteClient_Create_Call{Call: _e.mock.On("Create", ctx, value)}
}

func (_c *YugabyteClient_Create_Call) Return(_a0 error) *YugabyteClient_Create_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_m *MockYugabyteClient) First(ctx context.Context, dest interface{}, query interface{}, args ...interface{}) error {
	var _ca []interface{}
	_ca = append(_ca, ctx, dest, query)
	_ca = append(_ca, args...)
	ret := _m.Called(_ca...)
	if len(ret) == 0 {
		panic("no return value specified for First")
	}
	var r0 error
	if rf, ok := ret.Get(0).(func(context.Context, interface{}, interface{}, ...interface{}) error); ok {
		r0 = rf(ctx, dest, query, args...)
	} else {
		r0 = ret.Error(0)
	}
	return r0
}

type YugabyteClient_First_Call struct{ *mock.Call }

func (_e *YugabyteClient_Expecter) First(ctx, dest, query interface{}, args ...interface{}) *YugabyteClient_First_Call {
	return &YugabyteClient_First_Call{Call: _e.mock.On("First", append([]interface{}{ctx, dest, query}, args...)...)}
}

func (_c *YugabyteClient_First_Call) Return(_a0 error) *YugabyteClient_First_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_m *MockYugabyteClient) Save(ctx context.Context, value interface{}) error {
	ret := _m.Called(ctx, value)
	if len(ret) == 0 {
		panic("no return value specified for Save")
	}
	var r0 error
	if rf, ok := ret.Get(0).(func(context.Context, interface{}) error); ok {
		r0 = rf(ctx, value)
	} else {
		r0 = ret.Error(0)
	}
	return r0
}

type YugabyteClient_Save_Call struct{ *mock.Call }

func (_e *YugabyteClient_Expecter) Save(ctx, value interface{}) *YugabyteClient_Save_Call {
	return &YugabyteClient_Save_Call{Call: _e.mock.On("Save", ctx, value)}
}

func (_c *YugabyteClient_Save_Call) Return(_a0 error) *YugabyteClient_Save_Call {
	_c.Call.Return(_a0)
	return _c
}

var _ clients.YugabyteClient = (*MockYugabyteClient)(nil)

func NewMockYugabyteClient(t interface {
	mock.TestingT
	Cleanup(func())
}) *MockYugabyteClient {
	m := &MockYugabyteClient{}
	m.Mock.Test(t)
	t.Cleanup(func() { m.AssertExpectations(t) })
	return m
}

```

----------

### Elasticsearch

**`infrastructure/database/elasticsearch.go`**

```go
package database

import (
	"crypto/tls"
	"fmt"
	"io"
	"log"
	"net/http"
	"strings"

	"github.com/elastic/go-elasticsearch/v9"
)

var ElasticClient *elasticsearch.Client

func InitializeElasticsearch() {
	client, err := elasticsearch.NewClient(elasticsearch.Config{
		Addresses: []string{"https://localhost:9200"},
		Username:  "elastic",
		Password:  "your-password",
		APIKey:    "your-api-key",
		Transport: &http.Transport{
			TLSClientConfig: &tls.Config{
				InsecureSkipVerify: true, // disable in production
			},
		},
	})
	if err != nil {
		fmt.Println("=== Load Database Elasticsearch ===")
		fmt.Printf("Error Connecting To Elasticsearch : %+v \n", err)
		fmt.Println("=== End Load Database Elasticsearch ===")
		return
	}

	res, err := client.Info()
	if err != nil {
		log.Fatal("Error getting response from Elasticsearch:", err)
	}
	defer res.Body.Close()

	if res.IsError() {
		log.Fatal("Elasticsearch returned an error:", res.String())
	}

	var buf strings.Builder
	if _, err = io.Copy(&buf, res.Body); err != nil {
		log.Fatal("Error reading response body:", err)
	}
	log.Println("Connected to Elasticsearch:", buf.String())

	ElasticClient = client
	fmt.Println("=== Load Database Elasticsearch ===")
	fmt.Printf("Connecting To Elasticsearch : %+v \n", ElasticClient)
	fmt.Println("=== End Load Database Elasticsearch ===")

	CreateIndexIfNotExists()
}

func CreateIndexIfNotExists() {
	// Register index creation calls here, e.g.:
	// CreateIndexYourEntity()
}

```

**`infrastructure/database/clients/ElasticsearchClient.go`**

```go
package clients

import "github.com/elastic/go-elasticsearch/v9/esapi"

type ElasticsearchClient interface {
	Transport() esapi.Transport
}

```

**`infrastructure/database/connectors/ElasticsearchConnector.go`**

```go
package connectors

import (
	"github.com/elastic/go-elasticsearch/v9"
	"github.com/elastic/go-elasticsearch/v9/esapi"
)

type ElasticsearchConnector struct {
	client *elasticsearch.Client
}

func NewElasticsearchConnector(client *elasticsearch.Client) *ElasticsearchConnector {
	return &ElasticsearchConnector{client: client}
}

func (c *ElasticsearchConnector) Transport() esapi.Transport {
	return c.client
}

```

**`infrastructure/database/clients/mocks/MockElasticsearchClient.go`**

```go
// Code generated by mockery v2.53.5. DO NOT EDIT.

package clientmocks

import (
	clients "{module}/infrastructure/database/clients"

	"github.com/elastic/go-elasticsearch/v9/esapi"
	mock "github.com/stretchr/testify/mock"
)

type MockElasticsearchClient struct{ mock.Mock }

type ElasticsearchClient_Expecter struct{ mock *mock.Mock }

func (_m *MockElasticsearchClient) EXPECT() *ElasticsearchClient_Expecter {
	return &ElasticsearchClient_Expecter{mock: &_m.Mock}
}

func (_m *MockElasticsearchClient) Transport() esapi.Transport {
	ret := _m.Called()
	var r0 esapi.Transport
	if rf, ok := ret.Get(0).(func() esapi.Transport); ok {
		r0 = rf()
	} else if ret.Get(0) != nil {
		r0 = ret.Get(0).(esapi.Transport)
	}
	return r0
}

type ElasticsearchClient_Transport_Call struct{ *mock.Call }

func (_e *ElasticsearchClient_Expecter) Transport() *ElasticsearchClient_Transport_Call {
	return &ElasticsearchClient_Transport_Call{Call: _e.mock.On("Transport")}
}

func (_c *ElasticsearchClient_Transport_Call) Run(run func()) *ElasticsearchClient_Transport_Call {
	_c.Call.Run(func(_ mock.Arguments) { run() })
	return _c
}

func (_c *ElasticsearchClient_Transport_Call) Return(_a0 esapi.Transport) *ElasticsearchClient_Transport_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_c *ElasticsearchClient_Transport_Call) RunAndReturn(run func() esapi.Transport) *ElasticsearchClient_Transport_Call {
	_c.Call.Return(run)
	return _c
}

var _ clients.ElasticsearchClient = (*MockElasticsearchClient)(nil)

func NewMockElasticsearchClient(t interface {
	mock.TestingT
	Cleanup(func())
}) *MockElasticsearchClient {
	m := &MockElasticsearchClient{}
	m.Mock.Test(t)
	t.Cleanup(func() { m.AssertExpectations(t) })
	return m
}

```

----------

### AerospikeDB

**`infrastructure/database/aerospikedb.go`**

```go
package database

import (
	"log"
	"{module}/infrastructure/configuration"

	"github.com/aerospike/aerospike-client-go/v8"
)

var AerospikeDBClient *aerospike.Client

func InitializeAerospikeDB() {
	clientPolicy := aerospike.NewClientPolicy()
	client, err := aerospike.NewClientWithPolicy(clientPolicy, configuration.AppConfig.AerospikeHost, configuration.AppConfig.AerospikePort)
	if err != nil {
		log.Fatalf("Failed to connect to Aerospike: %v", err)
	}
	AerospikeDBClient = client
}

```

**`infrastructure/database/clients/AerospikeClient.go`**

```go
package clients

import "github.com/aerospike/aerospike-client-go/v8"

type AerospikeClient interface {
	PutBins(writePolicy *aerospike.WritePolicy, key *aerospike.Key, bins ...*aerospike.Bin) aerospike.Error
	Get(policy *aerospike.BasePolicy, key *aerospike.Key, binNames ...string) (*aerospike.Record, aerospike.Error)
	Put(policy *aerospike.WritePolicy, key *aerospike.Key, bins aerospike.BinMap) aerospike.Error
}

```

**`infrastructure/database/connectors/AerospikeConnector.go`**

```go
package connectors

import "github.com/aerospike/aerospike-client-go/v8"

type AerospikeConnector struct {
	client *aerospike.Client
}

func NewAerospikeConnector(client *aerospike.Client) *AerospikeConnector {
	return &AerospikeConnector{client: client}
}

func (c *AerospikeConnector) PutBins(writePolicy *aerospike.WritePolicy, key *aerospike.Key, bins ...*aerospike.Bin) aerospike.Error {
	return c.client.PutBins(writePolicy, key, bins...)
}

func (c *AerospikeConnector) Get(policy *aerospike.BasePolicy, key *aerospike.Key, binNames ...string) (*aerospike.Record, aerospike.Error) {
	return c.client.Get(policy, key, binNames...)
}

func (c *AerospikeConnector) Put(policy *aerospike.WritePolicy, key *aerospike.Key, bins aerospike.BinMap) aerospike.Error {
	return c.client.Put(policy, key, bins)
}

```

**`infrastructure/database/clients/mocks/MockAerospikeClient.go`**

```go
// Code generated by mockery v2.53.5. DO NOT EDIT.

package clientmocks

import (
	clients "{module}/infrastructure/database/clients"

	"github.com/aerospike/aerospike-client-go/v8"
	mock "github.com/stretchr/testify/mock"
)

type MockAerospikeClient struct{ mock.Mock }

type AerospikeClient_Expecter struct{ mock *mock.Mock }

func (_m *MockAerospikeClient) EXPECT() *AerospikeClient_Expecter {
	return &AerospikeClient_Expecter{mock: &_m.Mock}
}

func (_m *MockAerospikeClient) PutBins(writePolicy *aerospike.WritePolicy, key *aerospike.Key, bins ...*aerospike.Bin) aerospike.Error {
	var _ca []interface{}
	_ca = append(_ca, writePolicy, key)
	for _, b := range bins {
		_ca = append(_ca, b)
	}
	ret := _m.Called(_ca...)
	var r0 aerospike.Error
	if rf, ok := ret.Get(0).(func(*aerospike.WritePolicy, *aerospike.Key, ...*aerospike.Bin) aerospike.Error); ok {
		return rf(writePolicy, key, bins...)
	}
	if ret.Get(0) != nil {
		r0 = ret.Get(0).(aerospike.Error)
	}
	return r0
}

type AerospikeClient_PutBins_Call struct{ *mock.Call }

func (_e *AerospikeClient_Expecter) PutBins(writePolicy, key interface{}, bins ...interface{}) *AerospikeClient_PutBins_Call {
	return &AerospikeClient_PutBins_Call{Call: _e.mock.On("PutBins", append([]interface{}{writePolicy, key}, bins...)...)}
}

func (_c *AerospikeClient_PutBins_Call) Return(_a0 aerospike.Error) *AerospikeClient_PutBins_Call {
	_c.Call.Return(_a0)
	return _c
}

func (_m *MockAerospikeClient) Get(policy *aerospike.BasePolicy, key *aerospike.Key, binNames ...string) (*aerospike.Record, aerospike.Error) {
	var _ca []interface{}
	_ca = append(_ca, policy, key)
	for _, b := range binNames {
		_ca = append(_ca, b)
	}
	ret := _m.Called(_ca...)
	var r0 *aerospike.Record
	var r1 aerospike.Error
	if rf, ok := ret.Get(0).(func(*aerospike.BasePolicy, *aerospike.Key, ...string) (*aerospike.Record, aerospike.Error)); ok {
		return rf(policy, key, binNames...)
	}
	if ret.Get(0) != nil {
		r0 = ret.Get(0).(*aerospike.Record)
	}
	if ret.Get(1) != nil {
		r1 = ret.Get(1).(aerospike.Error)
	}
	return r0, r1
}

type AerospikeClient_Get_Call struct{ *mock.Call }

func (_e *AerospikeClient_Expecter) Get(policy, key interface{}, binNames ...interface{}) *AerospikeClient_Get_Call {
	return &AerospikeClient_Get_Call{Call: _e.mock.On("Get", append([]interface{}{policy, key}, binNames...)...)}
}

func (_c *AerospikeClient_Get_Call) Return(_a0 *aerospike.Record, _a1 aerospike.Error) *AerospikeClient_Get_Call {
	_c.Call.Return(_a0, _a1)
	return _c
}

func (_m *MockAerospikeClient) Put(policy *aerospike.WritePolicy, key *aerospike.Key, bins aerospike.BinMap) aerospike.Error {
	ret := _m.Called(policy, key, bins)
	var r0 aerospike.Error
	if rf, ok := ret.Get(0).(func(*aerospike.WritePolicy, *aerospike.Key, aerospike.BinMap) aerospike.Error); ok {
		return rf(policy, key, bins)
	}
	if ret.Get(0) != nil {
		r0 = ret.Get(0).(aerospike.Error)
	}
	return r0
}

type AerospikeClient_Put_Call struct{ *mock.Call }

func (_e *AerospikeClient_Expecter) Put(policy, key, bins interface{}) *AerospikeClient_Put_Call {
	return &AerospikeClient_Put_Call{Call: _e.mock.On("Put", policy, key, bins)}
}

func (_c *AerospikeClient_Put_Call) Return(_a0 aerospike.Error) *AerospikeClient_Put_Call {
	_c.Call.Return(_a0)
	return _c
}

var _ clients.AerospikeClient = (*MockAerospikeClient)(nil)

func NewMockAerospikeClient(t interface {
	mock.TestingT
	Cleanup(func())
}) *MockAerospikeClient {
	m := &MockAerospikeClient{}
	m.Mock.Test(t)
	t.Cleanup(func() { m.AssertExpectations(t) })
	return m
}

```

----------

## Models

The `infrastructure/database/models/` folder holds ORM / document data model structs. Generate a placeholder `ExampleDataModel.go` on scaffold; the user replaces it per entity. Standard pattern for gorm-based DBs (PostgreSQL, YugabyteDB):

```go
package databasemodels

type ExampleDataModel struct {
	ID          string `gorm:"primaryKey;unique;not null"`
	CreatedDate int64
	CreatedUser string
	CreatedIp   string
	UpdatedDate int64
	UpdatedUser string
	UpdatedIp   string
	DeletedDate int64
	DeletedUser string
	DeletedIp   string
	DataStatus  string
	// Add domain-specific fields here
}

```

For FerretDB / Aerospike / Elasticsearch, models are plain structs with JSON or BSON tags instead of gorm tags.

----------

## Database Summary Table

Database

Init File

Client Interface

Driver / SDK

Global Var Type

PostgreSQL

`postgresql.go`

`PostgresClient`

`gorm.io/driver/postgres` + otelgorm

`*gorm.DB`

FerretDB

`ferretdb.go`

`MongoClient`

`go.mongodb.org/mongo-driver/mongo`

`*mongo.Client` + `*mongo.Database`

Redis

`redis.go`

`RedisClient`

`github.com/redis/go-redis/v9`

`*redis.Client`

YugabyteDB

`yugabytedb.go`

`YugabyteClient`

`gorm.io/driver/postgres` (port 5433)

`*gorm.DB`

Elasticsearch

`elasticsearch.go`

`ElasticsearchClient`

`github.com/elastic/go-elasticsearch/v9`

`*elasticsearch.Client`

AerospikeDB

`aerospikedb.go`

`AerospikeClient`

`github.com/aerospike/aerospike-client-go/v8`

`*aerospike.Client`

----------

## Layer Responsibilities

Layer

Contains

Depends On

`domain`

Entities, business rules, domain-level interfaces

Nothing

`application`

Use cases, service interfaces (ports), event payloads

`domain`

`infrastructure`

All adapters: DB, cache, HTTP, messaging, config, DI

`application`, `domain`

----------

## Common Mistakes

Mistake

Fix

Domain imports infrastructure

Domain defines interface; infrastructure implements it (dependency inversion)

Use cases contain business rules

Business rules belong in `domain/entities` or `domain/rules`

Repo implementations placed in domain

Interfaces go in `domain/rules`; implementations go in `infrastructure/database/repositories`

DTOs leak into domain

Domain uses its own value objects; DTOs live in `application` only

`application/events` created without messaging

Only create it when Publisher or Subscriber is selected

Missing mock when adding a new client method

Re-run mockery or manually add the method block to `Mock<DB>Client.go`

----------

## Variations

-   **No base-dir wrapper**: create `domain/`, `application/`, `infrastructure/` directly at the project root by setting `BASE="."`.
-   **With `presentation/` layer**: add `presentation/controllers`, `presentation/views`, and `presentation/middleware` as a sibling to the three core layers.
-   **Monorepo / bounded contexts**: each context gets its own three-layer sub-tree under `packages/<context>/`.