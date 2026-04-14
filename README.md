# Shuttle Resource

Shuttle Resource is a [Concourse CI](https://concourse-ci.org/) custom resource type that enables coordinated communication between pipeline jobs. It provides a durable, asynchronous signaling mechanism: one job initiates a shuttle and another waits for a response, allowing pipelines to synchronize across job and pipeline boundaries without blocking workers indefinitely.

---

## Table of Contents

- [Introduction](#introduction)
- [Features](#features)
- [Getting Started](#getting-started)
- [Architecture Overview](#architecture-overview)
- [Usage Examples](#usage-examples)
- [Documentation](#documentation)
- [Contributing](#contributing)
- [License](#license)

---

## Introduction

Modern CI/CD pipelines often need to coordinate work across multiple jobs, teams, or even separate Concourse instances. Concourse's built-in primitives handle linear and fan-in/fan-out workflows well, but they do not natively support waiting for a response from an external or asynchronous process.

Shuttle Resource fills this gap. A "caller" job creates a shuttle — a lightweight record stored in S3 — and a "waiting" job polls until a "responding" job marks that shuttle as complete. The result (success or failure) is propagated back to the waiting job, which can then act accordingly.

This pattern is useful for:

- Triggering jobs in a separate pipeline and waiting for their outcome
- Coordinating between Concourse teams that share an S3 backend
- Integrating with external processes that report results asynchronously
- Modeling approval gates or manual intervention steps

---

## Features

- **Asynchronous coordination** between Concourse jobs without tying up a worker
- **Failure propagation** so waiting jobs know whether the remote job succeeded or failed
- **S3-compatible backend** — works with AWS S3, MinIO, Ceph, and any S3-compatible store
- **Flexible operation modes** — wait, return, skip, and a default read-only get
- **Bootstrap support** via `seed_version` to trigger pipelines on their first run
- **Small, auditable codebase** with a clean separation of concerns

---

## Getting Started

### Prerequisites

- A Concourse CI installation (v5.0 or later recommended)
- An S3 bucket accessible from your Concourse workers
- IAM credentials with `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, and `s3:ListBucket` permissions

### Register the Resource Type

Add the following to your pipeline YAML:

```yaml
resource_types:
  - name: shuttle
    type: registry-image
    source:
      repository: tristanpoland/shuttle-resource
      tag: latest
```

### Define a Resource

```yaml
resources:
  - name: my-shuttle
    type: shuttle
    source:
      bucket: my-concourse-bucket
      access_key_id: ((aws_access_key_id))
      secret_access_key: ((aws_secret_access_key))
      region: us-east-1
      path: shuttles/my-shuttle
```

For full source configuration options and S3-compatible backend setup, see [docs/integrations.md](./docs/integrations.md).

### Build the Image Yourself

```sh
git clone https://github.com/tristanpoland/shuttle-resource.git
cd shuttle-resource
docker build -t your-registry/shuttle-resource:latest .
docker push your-registry/shuttle-resource:latest
```

---

## Architecture Overview

Shuttle Resource stores coordination state as JSON objects in S3. Each shuttle corresponds to a single S3 object keyed by a Unix nanosecond timestamp. The object records the identity of the calling job and whether the shuttle has been responded to.

The three Concourse resource executables each play a distinct role:

| Executable | Role |
|------------|------|
| `check`    | Lists available versions (S3 object keys) newer than the current version |
| `in`       | Reads, waits on, or responds to a shuttle depending on parameters |
| `out`      | Creates a new shuttle version stamped with the calling job's identity |

The `in` executable supports four modes controlled by parameters: **get** (default), **wait**, **return**, and **skip**. A typical handshake involves a caller job running `out`, a remote job running `in` with `return: true`, and a waiting job running `in` with `wait: true`.

For a detailed explanation including a sequence diagram and component reference, see [docs/architecture.md](./docs/architecture.md).

---

## Usage Examples

### Minimal: Trigger and Wait

```yaml
jobs:
  - name: caller
    plan:
      - put: my-shuttle

  - name: remote-worker
    plan:
      - get: my-shuttle
        trigger: true
      - task: do-work
        # ...
      - put: my-shuttle
        params:
          return: true

  - name: waiter
    plan:
      - get: my-shuttle
        passed: [caller]
      - get: my-shuttle
        params:
          wait: true
```

### Propagate Failure

```yaml
      - put: my-shuttle
        params:
          return: true
          fail: true
```

### Skip Coordination

```yaml
      - get: my-shuttle
        params:
          skip: true
```

For complete annotated examples including failure propagation, timeouts, and bootstrapping, see [docs/examples.md](./docs/examples.md).

---

## Documentation

| Document | Description |
|----------|-------------|
| [docs/architecture.md](./docs/architecture.md) | Core concepts, sequence diagram, and component reference |
| [docs/integrations.md](./docs/integrations.md) | Concourse setup, S3-compatible backends, and Docker image build |
| [docs/examples.md](./docs/examples.md) | Complete pipeline YAML examples with explanations |
| [docs/extending.md](./docs/extending.md) | Guide for adding new storage backends and operation modes |
| [docs/faq.md](./docs/faq.md) | Frequently asked questions and troubleshooting guidance |
| [docs/contributing.md](./docs/contributing.md) | Contribution process, code style, and review expectations |

---

## Contributing

Contributions are welcome. Please open an issue before submitting large changes to discuss the approach. For smaller fixes, a pull request is sufficient.

See [docs/contributing.md](./docs/contributing.md) for full guidelines including code style, commit message format, and the review process.

---

## License

Distributed under the MIT License. See [LICENSE](./LICENSE) for details.
