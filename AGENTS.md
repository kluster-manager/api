# AGENTS.md

This file provides guidance to coding agents (e.g. Claude Code, claude.ai/code) when working with code in this repository.

## Repository purpose

Go module `open-cluster-management.io/api` — the **canonical** source of API types, generated Kubernetes clients, and CRDs for [Open Cluster Management (OCM)](https://open-cluster-management.io/). External controllers/projects import this lightweight library to talk to OCM without pulling in operator implementation dependencies. Library only; no binaries.

The fork is mirrored to `kluster-management/api`; **upstream is `open-cluster-management-io/api`** and this repo tracks it.

## Architecture

Four top-level API groups, each in its own directory with one folder per version. Hand-written `*_types.go`, generated `zz_generated.*.go`, generated CRD YAMLs:

- `cluster/` — `cluster.open-cluster-management.io` (managed cluster set, placements, placement decisions, etc.). Versions: `v1`, `v1alpha1`, `v1beta1`, `v1beta2`.
- `work/` — `work.open-cluster-management.io` (`ManifestWork`). Versions: `v1`, `v1alpha1`.
- `addon/` — `addon.open-cluster-management.io` (`ClusterManagementAddOn`, `ManagedClusterAddOn`). Versions: `v1alpha1`, `v1beta1`.
- `operator/v1/` — `operator.open-cluster-management.io` (cluster operator install types).

Supporting top-level dirs:

- `client/` — generated typed clientsets, listers, and informers for every API group/version above. Do not hand-edit.
- `utils/` — small public helpers (feature gates, conversion utilities, etc.).
- `feature/feature.go` — feature gate definitions used across OCM components.
- `hack/` — codegen scripts wrapped by `Makefile`'s `update-*` targets.
- `dependencymagnet/` — Go-only side-effect imports so `go mod` keeps codegen tools vendored.
- `test/` — apidiff / round-trip tests.
- `Dockerfile.build` — pinned Go toolchain image used by the `*-with-container` Make targets, so codegen is reproducible regardless of the host Go version.
- `vendor/` — vendored Go deps **and** the OpenShift `build-machinery-go` Makefile fragments used by `Makefile`. Required.

## Common commands

Codegen is the primary workflow here. The Makefile pulls in OpenShift's `build-machinery-go`.

- `make build` (alias `make all`) — Go build (mostly a sanity check; library has no main).
- `make update` — full regen chain: `update-scripts update-codegen-crds`. Run after changing any `*_types.go`.
- `make update-scripts` — re-run hack scripts (deepcopy, openapi, clientset, listers, informers).
- `make update-codegen-crds` — re-render CRD YAMLs via `controller-gen`.
- `make verify` — `check-env verify-scripts verify-codegen-crds verify-gocilint`. Use locally before opening a PR: it confirms `make update` left the tree clean *and* lints.
- `make verify-scripts` / `make verify-codegen-crds` / `make verify-gocilint` — individual verify steps.
- `make ensure-controller-gen` — install the pinned `controller-gen` into `bin/`.
- `make update-with-container` — same as `make update`, but inside `Dockerfile.build`. Use this if your host Go version doesn't match the pinned toolchain.
- `make build-runtime-image` — build `Dockerfile.build`.
- `make check-env` — sanity check that `GOPATH` and friends are set correctly.

There are no Go tests run by default; project relies on `verify` plus downstream test coverage.

## Conventions

- Module path is `open-cluster-management.io/api` (**upstream**); imports must use that, not the GitHub URL.
- **Upstream-tracking** fork (mirrored on GitHub as `kluster-management/api`). Prefer rebasing onto upstream over diverging. Isolate AppsCode-only patches so they replay cleanly.
- Do not hand-edit any `zz_generated.*.go`, files under `client/`, or generated CRD YAMLs. Change `<group>/<version>/*_types.go` and re-run `make update`.
- Vendor directory is checked in **and load-bearing** — Makefile includes `.mk` files from `vendor/github.com/openshift/build-machinery-go/make/`. Don't break that with a careless `go mod vendor`.
- License: Apache-2.0 (`LICENSE`); per-file headers note "Copyright Contributors to the Open Cluster Management project".
- Sign off commits (`git commit -s`) — `DCO` is checked in.
- When adding a new API group: drop a new top-level directory with `<version>/types_*.go`, add `register.go` and `groupversion_info.go`, then register it in `hack/`'s codegen scripts so `make update` regenerates clientsets/CRDs.
- Bumping the pinned Go toolchain: change `Dockerfile.build` and `go.mod` together so `make update-with-container` stays reproducible.
