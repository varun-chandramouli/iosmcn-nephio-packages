# iosmcncore-flux

## Description
kpt package for deploying iosmcncore helm charts via flux

## Usage

### Fetch the package
`kpt pkg get REPO_URI[.git]/PKG_PATH[@VERSION] iosmcncore-flux`
Details: https://kpt.dev/reference/cli/pkg/get/

### View package content
`kpt pkg tree iosmcncore-flux`
Details: https://kpt.dev/reference/cli/pkg/tree/

### Apply the package
```
kpt live init iosmcncore-flux
kpt live apply iosmcncore-flux --reconcile-timeout=2m --output=table
```
Details: https://kpt.dev/reference/cli/live/
