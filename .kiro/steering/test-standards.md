# Test Standards

## Unit Tests

### Structure

Table-driven tests with closure-based subtests:
```go
func TestMethodName(t *testing.T) {
    testCases := []struct {
        name     string
        testFunc func(t *testing.T)
    }{
        {
            name: "Success: descriptive scenario",
            testFunc: func(t *testing.T) {
                // setup, act, assert
            },
        },
        {
            name: "Fail: missing required field",
            testFunc: func(t *testing.T) { ... },
        },
    }
    for _, tc := range testCases {
        t.Run(tc.name, tc.testFunc)
    }
}
```

### Mocking

Use `gomock` with generated mocks from `pkg/driver/mocks/` and `pkg/cloud/mocks/`:
```go
mockCtl := gomock.NewController(t)
mockCloud := mocks.NewMockCloud(mockCtl)
mockCloud.EXPECT().CreateAccessPoint(gomock.Eq(ctx), gomock.Any(), gomock.Any(), gomock.Eq(util.FileSystemTypeEFS)).Return(accessPoint, nil)
// ... test logic ...
mockCtl.Finish()
```

Use `FakeCloudProvider` (in `pkg/cloud/fakes.go`) for sanity tests that need a working in-memory cloud backend without mock expectations.

Regenerate mocks with `./hack/update-gomock` — never edit mock files by hand.

### Assertions

Use `t.Fatalf` for failures that should stop the test. Use direct comparisons:
```go
if res.Volume.VolumeId != expectedId {
    t.Fatalf("Volume Id mismatched. Expected: %v, Actual: %v", expectedId, res.Volume.VolumeId)
}
```

For gRPC error checks:
```go
if err == nil {
    t.Fatal("Expected error, got nil")
}
st, ok := status.FromError(err)
if !ok || st.Code() != codes.InvalidArgument {
    t.Fatalf("Expected InvalidArgument, got: %v", err)
}
```

### Test Setup Pattern

Each test case creates its own `Driver` struct with mocked dependencies:
```go
driver := &Driver{
    endpoint:     endpoint,
    cloud:        mockCloud,
    gidAllocator: NewGidAllocator(),
    lockManager:  NewLockManagerMap(),
}
```

## E2E Tests

### Framework

Uses Ginkgo v2 + Gomega with the Kubernetes test framework (`k8s.io/kubernetes/test/e2e/framework`).

### Running

```bash
# Local cluster (requires existing EFS filesystem + kubeconfig)
go test -v -timeout 0 ./test/e2e/... \
  -ginkgo.focus="\[efs-csi\]" -ginkgo.skip="\[Disruptive\]" \
  --file-system-id=$FS_ID --create-file-system=false --deploy-driver=true --region=$REGION

# Release pipeline (CodePipeline in 745939127895)
aws codepipeline start-pipeline-execution \
  --name csi-driver-dev-e2e-test \
  --variables name=IMAGE_TAG,value=my-tag name=DRIVER_BRANCH,value=my-branch \
  --region us-east-1
```

### Key Flags

- `--file-system-id` — existing EFS filesystem ID
- `--create-file-system` — provision a new filesystem (requires `--cluster-name`, `--region`)
- `--deploy-driver` — install the driver via Helm before tests
- `--cross-account-secret-name` — K8s secret for cross-account tests
- `--s3files-file-system-id` — existing S3 Files filesystem for S3 Files tests

### Test Naming

E2E specs use `[efs-csi]` tag. Disruptive tests use `[Disruptive]` tag. Serial tests use `[Serial]`.
