# Build & Test

```bash
make                                          # Build binary (bin/aws-efs-csi-driver)
make test                                     # Unit tests (go test -v -race ./pkg/...)
make verify                                   # Static analysis (gofmt, govet, golint)
go test -v -race ./pkg/driver/ -run TestName  # Single unit test
go test -v -race ./pkg/cloud/ -run TestName   # Single cloud test
```

# E2E Tests (release pipeline)

E2E tests run via CodePipeline in account 745939127895 (us-east-1). Push your branch to CodeCommit and trigger:

```bash
git remote add codecommit https://git-codecommit.us-east-1.amazonaws.com/v1/repos/aws-efs-csi-driver
git push codecommit my-branch:my-branch

# Build from branch + run all E2E tests
aws codepipeline start-pipeline-execution \
  --name csi-driver-dev-e2e-test \
  --variables name=IMAGE_TAG,value=my-tag name=DRIVER_BRANCH,value=my-branch \
  --region us-east-1

# Test existing image (skip build)
aws codepipeline start-pipeline-execution \
  --name csi-driver-dev-e2e-test \
  --variables name=IMAGE_TAG,value=my-tag \
  --region us-east-1

# Single test scenario
aws codebuild start-build \
  --project-name csi-driver-dev-e2e-test-amd64 \
  --environment-variables-override name=VERSION,value=my-tag,type=PLAINTEXT name=RELEASE_BRANCH,value=dev,type=PLAINTEXT \
  --region us-east-1
```

E2E test projects: `csi-driver-dev-e2e-test-{amd64,arm64,fips,eu,eu-fips,auto-mode,cross-account,cross-account-az,cross-account-ca-true}`

# E2E Tests (local cluster)

```bash
export KUBECONFIG=$HOME/.kube/config
go test -v -timeout 0 ./test/e2e/... -ginkgo.focus="\[efs-csi\]" -ginkgo.skip="\[Disruptive\]" \
  --file-system-id=$FS_ID --create-file-system=false --deploy-driver=true --region=$REGION
```

# Standards

Follow .kiro/steering/coding-standards.md — error handling, logging, naming conventions
Follow .kiro/steering/test-standards.md — mocking style, test patterns
Follow .kiro/steering/architecture.md — component overview, data flow
