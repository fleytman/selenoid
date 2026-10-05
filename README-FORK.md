# Selenoid fork

Fork of [aerokube/selenoid](https://github.com/aerokube/selenoid) (archived in December 2024, Apache-2.0).
Meant for a macOS device farm where Selenoid runs in driver mode (`-disable-docker`) and starts Appium
as a local process for every session. Upstream `README.md` and `LICENSE` are kept unchanged.

## The change

One line in `service/driver.go` (`StartWithCancel`):

```diff
-	hp := session.HostPort{}
+	hp := session.HostPort{Devtools: u.Host}
```

Selenoid reverse-proxies `/devtools/<session-id>/<path>` to `http://<HostPort.Devtools>/<path>`
(`reverseProxy` in `selenoid.go`, route in `main.go`). Docker sessions fill `HostPort.Devtools` with the
container port, but process drivers returned an empty `HostPort`, so every `/devtools/...` request of an Appium
session ended with `502 Bad Gateway`. With the patch the path is forwarded to the root of the Appium process
(`127.0.0.1:<port>`), so Appium websocket endpoints become reachable through Selenoid and GGR, for example:

```
ws://<ggr-or-selenoid>/devtools/<session-id>/ws/session/<appium-session-id>/appium/device/syslog
ws://<ggr-or-selenoid>/devtools/<session-id>/bidi/<appium-session-id>
```

Nothing else changes: regular `/wd/hub/session/...` traffic and docker mode behave as upstream.
Note that `/devtools/<session-id>/` exposes the whole HTTP/websocket API of that Appium process
(including `/status` and `/sessions`), not only websocket endpoints.

## Branches and tags

| branch | based on | Go (as upstream CI) | tags |
|---|---|---|---|
| `fork/1.10.9` | tag `1.10.9` | 1.19.3 | `1.10.9-fork.<n>` |

`selenoid -version` and `/ping` report the tag, e.g. `Git Revision: 1.10.9-fork.1`.

Upstream workflows are removed: `build.yml` and `release.yml` push to Docker Hub and gh-pages with aerokube
secrets (`release.yml` would also fire on our releases), `test.yml` is replaced by the `pull_request` trigger of
`release-fork.yml`. `ci/*.sh` are kept as upstream
but not used by CI.

## Build locally

Same flags as upstream `ci/build.sh` (`CGO_ENABLED=0`, `-X main.gitRevision`, `-X main.buildStamp`, `-s -w`):

```bash
VERSION=$(git describe --tags)
STAMP=$(LC_ALL=C date -u '+%Y-%m-%d_%I:%M:%S%p')
CGO_ENABLED=0 GOOS=darwin GOARCH=arm64 GOTOOLCHAIN=go1.19.3 go build \
  -ldflags "-X main.buildStamp=${STAMP} -X main.gitRevision=${VERSION} -s -w" \
  -o dist/selenoid_darwin_arm64 .
./dist/selenoid_darwin_arm64 -version
```

Tests: `go test -tags 's3 metadata' ./...`.

## Release

1. Commit on top of the branch.
2. Tag and push: `git tag 1.10.9-fork.2 && git push origin fork/1.10.9 1.10.9-fork.2`.
3. `.github/workflows/release-fork.yml` checks that the tag matches `git describe` and the upstream base, runs
   tests, builds `selenoid_darwin_arm64` and `selenoid_linux_amd64` with `.sha256` files and creates a GitHub
   Release with them.

Download URL for the farm:
`https://github.com/<org>/<repo>/releases/download/<tag>/selenoid_darwin_arm64`.
