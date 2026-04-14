# Extending Shuttle Resource

Shuttle Resource is designed to be extended. The most common extension point is adding a new storage backend, though the overall architecture also makes it straightforward to add new operation modes or enhance the version model.

---

## Adding a New Storage Backend

The storage layer is abstracted behind the `Driver` interface defined in `driver/driver.go`:

```go
type Driver interface {
    Read(version models.Version) (*models.Payload, error)
    Write(version models.Version, payload models.Payload) error
    Versions() (models.VersionList, error)
    Clean(version models.Version) error
}
```

To add a new backend, follow these steps:

### Step 1: Create the Driver File

Add a new file in the `driver/` package. The file name should reflect the backend:

```
driver/
  driver.go      # Interface definition
  s3.go          # Existing S3 implementation
  redis.go       # Example: new Redis backend
```

### Step 2: Implement the Interface

Your struct must implement all four methods. Here is a skeleton:

```go
package driver

import "github.com/thomasmitchell/shuttle-resource/models"

type redisDriver struct {
    // connection fields
}

func newRedisDriver(cfg models.Source) (*redisDriver, error) {
    // initialize connection
    return &redisDriver{}, nil
}

func (r *redisDriver) Read(version models.Version) (*models.Payload, error) {
    // retrieve payload for version
    return nil, nil
}

func (r *redisDriver) Write(version models.Version, payload models.Payload) error {
    // store payload for version
    return nil
}

func (r *redisDriver) Versions() (models.VersionList, error) {
    // list all known versions
    return models.VersionList{}, nil
}

func (r *redisDriver) Clean(version models.Version) error {
    // delete the version after it has been consumed
    return nil
}
```

### Step 3: Register the Driver

Update the `New` function in `driver/driver.go` to instantiate your driver based on source configuration. Consider introducing a `Driver` field in `models.Source` to select the backend:

```go
func New(cfg models.Source) (Driver, error) {
    switch cfg.Driver {
    case "redis":
        return newRedisDriver(cfg)
    default:
        return newS3Driver(cfg)
    }
}
```

### Step 4: Extend the Source Model

If your backend requires additional configuration fields, add them to `models.Source` in `models/models.go`:

```go
type Source struct {
    // existing fields ...
    Driver    string `json:"driver"`
    RedisAddr string `json:"redis_addr"`
}
```

---

## Adding a New Operation Mode

Operation modes are parsed in `in/main.go` by the `parseMode` function. To add a new mode:

1. Define a new constant in the `modeT` iota block.
2. Add the corresponding field to the `Params` struct.
3. Handle the new mode in the `parseMode` function.
4. Implement the mode's logic in the `main` function's `switch` statement.

---

## Extending the Payload

The `Payload` struct (`models/models.go`) can be extended with additional fields. Because payloads are stored as JSON, new fields are backward compatible as long as existing fields are not removed or renamed.

```go
type Payload struct {
    Caller   string `json:"caller"`
    Done     bool   `json:"done"`
    Passed   bool   `json:"passed"`
    // New field — optional, zero value is safe
    Message  string `json:"message,omitempty"`
}
```

---

## Recommended Practices

- Keep new drivers in the `driver/` package and name them after the backend they implement.
- Return typed errors (see `driver/errors.go`) to allow callers to distinguish not-found conditions from other errors.
- Update `docs/integrations.md` when adding a backend that requires specific infrastructure.
- Add at minimum a brief description of your extension to the relevant documentation file before submitting a pull request.

---

For contribution guidelines, see [contributing.md](./contributing.md).
