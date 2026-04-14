# Integration Guides

Shuttle Resource is a [Concourse CI](https://concourse-ci.org/) custom resource type. This document covers how to integrate it into a Concourse pipeline, configure the S3 backend, and work with S3-compatible storage systems.

---

## Prerequisites

- A running Concourse CI installation (v5.0 or later recommended)
- An S3 bucket (AWS S3 or a compatible alternative) accessible from your Concourse workers
- IAM credentials with at minimum `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, and `s3:ListBucket` permissions on the target bucket

---

## Registering the Resource Type

Add the following block to your pipeline YAML to register Shuttle Resource as a custom resource type:

```yaml
resource_types:
  - name: shuttle
    type: registry-image
    source:
      repository: tristanpoland/shuttle-resource
      tag: latest
```

Replace `latest` with a specific image digest or tag for reproducible pipelines.

---

## Defining a Resource

Declare a resource using the registered type. The `source` block configures the S3 backend:

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

### Source Configuration Reference

| Field               | Type    | Required | Description |
|---------------------|---------|----------|-------------|
| `bucket`            | string  | Yes      | Name of the S3 bucket |
| `access_key_id`     | string  | Yes      | AWS access key ID (use Concourse vars) |
| `secret_access_key` | string  | Yes      | AWS secret access key (use Concourse vars) |
| `region`            | string  | Yes      | AWS region (e.g., `us-east-1`) |
| `path`              | string  | Yes      | Key prefix within the bucket |
| `endpoint`          | string  | No       | Custom S3-compatible endpoint URL |
| `tls_skip_verify`   | boolean | No       | Disable TLS certificate verification (not recommended for production) |
| `seed_version`      | boolean | No       | Inject version `1` to bootstrap a fresh pipeline |

---

## Using S3-Compatible Storage

Shuttle Resource supports any S3-compatible object store by setting the `endpoint` field:

### MinIO

```yaml
source:
  bucket: shuttle-bucket
  access_key_id: minioadmin
  secret_access_key: minioadmin
  region: us-east-1
  endpoint: http://minio.internal:9000
```

### Ceph RADOS Gateway

```yaml
source:
  bucket: shuttle-bucket
  access_key_id: ((ceph_key_id))
  secret_access_key: ((ceph_secret))
  region: default
  endpoint: https://radosgw.internal
```

When using custom endpoints, ensure the endpoint is reachable from Concourse workers. If the endpoint uses a self-signed certificate, you may set `tls_skip_verify: true`, though this is not recommended in production environments.

---

## Concourse Version Compatibility

Shuttle Resource has been tested with Concourse v5.x and later. The resource uses the standard Concourse resource protocol (JSON over stdin/stdout), so it is compatible with any version that supports custom resource types via `registry-image`.

---

## Docker Image

The resource is distributed as a Docker image built from the included `Dockerfile`. The build uses a multi-stage build:

1. **Build stage** (`golang:1.15`): Compiles the `check`, `in`, and `out` binaries.
2. **Runtime stage** (`ubuntu:bionic`): Copies the binaries to `/opt/resource/`.

To build and push your own image:

```sh
docker build -t your-registry/shuttle-resource:latest .
docker push your-registry/shuttle-resource:latest
```

Then update the `repository` field in your `resource_types` block accordingly.

---

## Future Integrations

Support for additional storage backends (such as Redis, Consul, or cloud-native services) is under consideration. See [extending.md](./extending.md) for guidance on contributing a new backend.
