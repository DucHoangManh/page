---
title: "How to write integration tests with test container in Go"
date: 2023-12-27T18:31:05+07:00
tags: ["go", "efective-go"]
---

The significance of integration testing in assuring the reliability and seamless operation of software systems cannot be understated.
This guide will help you get started with using containers for integration testing and learn how to set up and execute test cases effectively.

### 1. Use container for testing  
Let's consider a small example program where I use postgres as the underlying database to store product records:
```SQL
create table product  
(  
    id    serial8      not null primary key,  
    name  varchar(100) not null,  
    type  varchar(50)  not null,  
    code  varchar(50)  not null,  
    price int4         not null  
);
```
The program can create, update and list products via a repository:
```go
type Product struct {  
    ID    int64  `db:"id"`  
    Name  string `db:"name"`  
    Type  string `db:"type"`  
    Code  string `db:"code"`  
    Price int64  `db:"price"`  
}
```
```go
...
type ProductPersistenceRepository struct {  
    db *sqlx.DB  
}

func (r *ProductPersistenceRepository) Create(product Product) (Product, error) {
    created := Product{}
    rows, err := r.db.NamedQuery(
        "INSERT INTO product (name, type, code, price) VALUES (:name, :type, :code, :price) RETURNING *", product,
    )
    if err != nil {
        return created, fmt.Errorf("ProductPersistenceRepository Create: %w", err)
    }
    for rows.Next() {
        if err := rows.StructScan(&created); err != nil {
            return created, fmt.Errorf("ProductPersistenceRepository Create: %w", err)
        }
    }
    return created, nil
}
...
// update and list functions
```
To test these functions against a real database, we'll need a clean database instance.  
With a bit of research, we can find that starting and cleaning up containerized dependencies can be done with ease by running database within a container as part of the test itself.  
I have come across some libraries that help to effectively spin up testing container, such as [dockertest](https://github.com/ory/dockertest) or [testcontainer](https://golang.testcontainers.org/), which I will utilize in this guide.
To begin, let's write some code that runs a postgres container and returns the container address and port for later connection.
```go
func NewTestDatabase(t *testing.T) (string, string) {
    ctx, cancel := context.WithTimeout(context.Background(), time.Minute)
    defer cancel()
    req := testcontainers.ContainerRequest{
        Image:        "postgres:12",
        ExposedPorts: []string{"5432/tcp"},
        HostConfigModifier: func(config *container.HostConfig) {
            config.AutoRemove = true
        },
        Env: map[string]string{
            "POSTGRES_USER":     "denishoang",
            "POSTGRES_PASSWORD": "pgpassword",
            "POSTGRES_DB":       "products",
        },
        WaitingFor: wait.ForListeningPort("5432/tcp"),
    }
    postgres, err := testcontainers.GenericContainer(
        ctx, testcontainers.GenericContainerRequest{
            ContainerRequest: req,
            Started:          true,
        },
    )
    require.NoError(t, err)
    mappedPort, err := postgres.MappedPort(ctx, "5432")
    require.NoError(t, err)
    
    hostIP, err := postgres.Host(ctx)
    require.NoError(t, err)
    
    return hostIP, mappedPort.Port()
}
```
The function above runs a postgres container like you normally do with docker cli by interacting with the docker api.

Notice the request's param `WaitingFor: wait.ForListeningPort("5432/tcp")`, with this configuration, our tests will check if the container is listening to a specific port (`5432/tcp` in this case) before proceeding to the next part, makes sure that the database is ready before performing any tests. You can take a look at other wait strategies supported by testcontainer [here](https://golang.testcontainers.org/features/wait/introduction/).

This way, we can spin up a clean database for each test and remove it after the test is done.

Let's get our hands dirty with the integration test, we'll use the address and port returned from the `NewTestDatabase` function to connect to the database and perform the tests.
```go
func Test_Product(t *testing.T) {
    host, port := NewTestDatabase(t)
    db, err := sqlx.Connect(
        "postgres",
        fmt.Sprintf(
            "postgres://denishoang:pgpassword@%s:%s/products?sslmode=disable", host,
            port,
        ),
    )
    require.Nil(t, err)
    repo := repository.NewProductPersistenceRepository(db)
    t.Run(
        "test create product", func(t *testing.T) {
            created, err := repo.Create(
                repository.Product{
                    Name:  "cake",
                    Type:  "food",
                    Code:  "c1",
                    Price: 100,
                },
            )
            require.Nil(t, err)
            require.Equal(t, "cake", created.Name)
        },
    )
}
```
try to run the test, and you'll see in the logs that the postgres container is being created and started, and the test is being executed.
```
✅ Container created: ec1c30ac431b
🐳 Starting container: ec1c30ac431b
✅ Container started: ec1c30ac431b
🚧 Waiting for container id ec1c30ac431b image: postgres:12. Waiting for: &{Port:5432/tcp timeout:<nil> PollInterval:100ms}
```
then the test failed, is there something wrong?
```
=== RUN   Test_Product/test_create_product
    product_test.go:34: 
        	Error Trace:	product_test.go:34
        	Error:      	Expected nil, but got: &fmt.wrapError{msg:"ProductPersistenceRepository Create: pq: relation \"product\" does not exist", err:(*pq.Error)(0x14000442b40)}
        	Test:       	Test_Product/test_create_product
    --- FAIL: Test_Product/test_create_product (0.00s)
```
The error message tells us that the table `product` does not exist, since we are using a clean database, it's necessary to perform some migrations before running the test.

I'm using [golang-migrate](github.com/golang-migrate) to do the migrations, this involves placing all the migration files in a directory and applying them before the tests.
```go
//go:embed "migrations"
var EmbeddedFiles embed.FS
```
```go
package persistence

func MigrationUp(completeDsn string) error {
    iofsDriver, err := iofs.New(EmbeddedFiles, "migrations")
    if err != nil {
        return err
    }
    
    migrator, err := migrate.NewWithSourceInstance("iofs", iofsDriver, completeDsn)
    if err != nil {
        return err
    }
    
    return migrator.Up()
}
```
use this function after the database is ready:
```go
func NewTestDatabase(t *testing.T) (string, string) {
    ...
    err = persistence.MigrationUp(
        fmt.Sprintf(
            "postgres://denishoang:pgpassword@%s:%s/products?sslmode=disable", hostIP,
            mappedPort.Port(),
        ),
    )
    require.NoError(t, err)
    return hostIP, mappedPort.Port()
}
```
Now, the test should pass.

### 2. Database per test 

However, there are still some things to consider: We don't really need an entire database instance for each test as it can be resource-consuming and unnecessary.

Ideally, we can reuse the previously created database instance and create a new database for each test.

Let's rewrite the `NewTestDatabase` function, instead of directly return the address and port of the database instance, we just create a new database instance that can be reused later.
```go
var postgresContainer testcontainers.Container

func StartDatabase() {
    ctx, cancel := context.WithTimeout(context.Background(), time.Minute)
    defer cancel()
    req := testcontainers.ContainerRequest{
        Image:        "postgres:12",
        ExposedPorts: []string{"5432/tcp"},
        HostConfigModifier: func(config *container.HostConfig) {
            config.AutoRemove = true
        },
        Env: map[string]string{
            "POSTGRES_USER":     "denishoang",
            "POSTGRES_PASSWORD": "pgpassword",
            "POSTGRES_DB":       "products",
        },
        WaitingFor: wait.ForListeningPort("5432/tcp"),
    }
    postgres, err := testcontainers.GenericContainer(
        ctx, testcontainers.GenericContainerRequest{
            ContainerRequest: req,
            Started:          true,
        },
    )
    if err != nil {
        os.Exit(1)
    }
    postgresContainer = postgres
}
```
And a function to create a new database for each test:
```go
func NewDatabase(t *testing.T) *sqlx.DB {
    ctx := context.Background()
    if postgresContainer == nil {
        t.Fatal("postgres is not yet started")
    }
    mappedPort, err := postgresContainer.MappedPort(ctx, "5432")
    if err != nil {
        t.Fatal("err get mapped port from container")
    }
    
    hostIP, err := postgresContainer.Host(ctx)
    // open connection to postgres instance in order to create other databases
    baseDb, err := sqlx.Open(
        "postgres", fmt.Sprintf(
            "postgres://denishoang:pgpassword@%s:%s/products?sslmode=disable", hostIP, mappedPort.Port(),
        ),
    )
    defer func() {
        if err := baseDb.Close(); err != nil {
            t.Fatal("err close connection to db")
        }
    }()
    // a naive scheme to generate random database names
    dbName := fmt.Sprintf("%s_%d", "products", rand.Int63())
    if _, err := baseDb.Exec(fmt.Sprintf("CREATE DATABASE %s;", dbName)); err != nil {
        t.Fatal("err creating postgres database")
    }
    connString := fmt.Sprintf(
        "postgres://denishoang:pgpassword@%s:%s/%s?sslmode=disable", hostIP, mappedPort.Port(), dbName,
    )
    // apply migrations to the newly created database
    if err = persistence.MigrationUp(connString); err != nil {
        t.Fatal("err connect postgres database")
    }
    // connect to new database
    db, err := sqlx.Open("postgres", connString)
    if err != nil {
        t.Fatal("err connect postgres database")
    }
    return db
}
```
As you can see, there is quite a bit happening here; let me explain:
- We have started the postgres container in the `StartDatabase` function, and stored the container instance in a global variable.
- The `NewDatabase` function will create a new database for each test, by connecting to the postgres instance and creating a new database with a random name. It then applies migrations to the new database and returns the connection to the new database.
- The test that calls `NewDatabase` function will have its own database so, there is no need to worry about cleaning up the database after each test.
- Another positive aspect is that our tests can run in parallel without any issues.

But we still have a problem, how to run `StartDatabase` before any tests?   
Luckily, go's build in test utility provides the `TestMain` function, which allows us to run some setup code before or after all the tests residing in a package.
```go
func TestMain(m *testing.M) {
    StartDatabase()
    code := m.Run()
    // go test will decide whether the tests failed or not by exit code
    os.Exit(code)
```
Put everything together by modifying our tests:
```go
func Test_Product(t *testing.T) {
    t.Parallel()
    t.Run(
        "test create product", func(t *testing.T) {
            t.Parallel()
            db := NewDatabase(t)
            repo := repository.NewProductPersistenceRepository(db)
            created, err := repo.Create(
                repository.Product{
                    Name:  "cake",
                    Type:  "food",
                    Code:  "c1",
                    Price: 100,
                },
            )
            require.Nil(t, err)
            require.Equal(t, "cake", created.Name)
        },
    )
    
    t.Run(
        "test get all products", func(t *testing.T) {
            t.Parallel()
            db := NewDatabase(t)
            repo := repository.NewProductPersistenceRepository(db)
            _, err := repo.Create(
                repository.Product{
                    Name:  "cake",
                    Type:  "food",
                    Code:  "c1",
                    Price: 100,
                },
            )
            require.Nil(t, err)
            products, err := repo.GetAll()
            require.Nil(t, err)
            require.Len(t, products, 1)
        },
    )
    
    t.Run(
        "test update product", func(t *testing.T) {
            t.Parallel()
            db := NewDatabase(t)
            repo := repository.NewProductPersistenceRepository(db)
            created, err := repo.Create(
                repository.Product{
                    Name:  "cake",
                    Type:  "food",
                    Code:  "c1",
                    Price: 100,
                },
            )
            require.Nil(t, err)
            created.Name = "new cake"
            updated, err := repo.Update(created)
            require.Nil(t, err)
            require.Equal(t, "new cake", updated.Name)
        },
    )
}
```
We have learned how to use container for integration testing, and how to set up and execute test cases effectively.   
You can see the full code [here](https://github.com/duchng/go-test-container-postgres)

### 3. References
- [https://golang.testcontainers.org/](https://golang.testcontainers.org/)