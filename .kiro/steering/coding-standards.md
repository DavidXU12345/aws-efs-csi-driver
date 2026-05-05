# Coding Standards

## Error Handling

Use gRPC status errors for CSI method returns:
```go
return nil, status.Errorf(codes.InvalidArgument, "Volume ID not provided")
```

Use sentinel errors in the `cloud` package (`ErrNotFound`, `ErrAlreadyExists`, `ErrAccessDenied`). Check with `errors.Is()`.

Never swallow errors silently. Wrap with context using `fmt.Errorf("operation failed: %w", err)`.

## Logging

Use `k8s.io/klog/v2`. Levels:
- `klog.V(4).Infof` — CSI method entry with sanitized request args
- `klog.V(5).Infof` — detailed operation steps (mkdir, mount, unmount)
- `klog.Infof` — significant business events
- `klog.Warningf` — recoverable issues (fallback paths, deprecated options)
- `klog.Errorf` — failures that will be returned to the caller

Never log credentials, IAM role ARNs with session tokens, or full request/response bodies. Use `util.SanitizeRequest()` for CSI requests.

## Naming

- Interfaces in `cloud` package: `Cloud`, `Efs`, `S3Files`, `MetadataService`
- Struct fields: camelCase, exported where needed for JSON serialization
- Constants: CamelCase (e.g., `AccessPointPerFsLimit`, `DefaultGidMin`)
- Test functions: `TestMethodName` with table-driven subtests

## Dependencies

- AWS SDK: `aws-sdk-go-v2` (NOT v1)
- CSI spec: `github.com/container-storage-interface/spec`
- Mocking: `github.com/golang/mock/gomock`
- K8s client: `k8s.io/client-go`
- Vendored: all deps in `vendor/`, use `go mod vendor` after changes

## Patterns to Follow

- Table-driven tests with `testCases []struct{ name string; testFunc func(t *testing.T) }`
- `gomock` for interface mocking, `FakeCloudProvider` for integration-style tests
- CSI methods validate all required fields upfront, return `codes.InvalidArgument` for missing fields
- Volume context properties parsed from `map[string]string` — validate types explicitly

## Patterns to Avoid

- Do NOT use `log` or `fmt.Println` for logging — always `klog`
- Do NOT add new dependencies without vendoring (`go mod vendor`)
- Do NOT use `aws-sdk-go` v1 — the project has fully migrated to v2
- Do NOT modify generated mock files in `pkg/driver/mocks/` or `pkg/cloud/mocks/` by hand — regenerate with `./hack/update-gomock`
