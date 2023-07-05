`xk6` - Custom k6 Builder
===============================

This command line tool and associated Go package makes it easy to make custom builds of [k6](https://github.com/grafana/k6).

It is used heavily by k6 extension developers as well as anyone who wishes to make custom `k6` binaries (with or without extensions).


## Docker

The easiest way to use xk6 is via our [Docker image](https://hub.docker.com/r/grafana/xk6/). This avoids having to setup a local Go environment, and install xk6 manually.

For example, to build a k6 v0.43.1 binary on Linux with the [xk6-kafka](https://github.com/mostafa/xk6-kafka) and [xk6-output-influxdb](https://github.com/grafana/xk6-output-influxdb) extensions, you would run:

```bash
docker run --rm -it -u "$(id -u):$(id -g)" -v "${PWD}:/xk6" grafana/xk6 build v0.43.1 \
  --with github.com/mostafa/xk6-kafka@v0.17.0 \
  --with github.com/grafana/xk6-output-influxdb@v0.3.0
```

This would create a `k6` binary in the current working directory.

Note the use of the `-u` (user) option to specify the user and group IDs of the account on the host machine. This is important for the `k6` file to have the same file permissions as the host user.

The `-v` (volume) option is also required to mount the current working directory inside the container, so that the `k6` binary can be written to it.

Note that if you're using SELinux, you might need to add `:z` to the `--volume` option to avoid permission errors. E.g. `-v "${PWD}:/xk6:z"`.

If you prefer to setup Go and use xk6 without Docker, see the "Local Installation" section below.


### macOS

On macOS you will need to set the `GOOS=darwin` environment variable to build a macOS binary.

You can do this with the `--env` or `-e` argument to `docker run`:
```bash
docker run --rm -it -e GOOS=darwin -u "$(id -u):$(id -g)" -v "${PWD}:/xk6" \
  grafana/xk6 build v0.43.1 \
  --with github.com/mostafa/xk6-kafka@v0.17.0 \
  --with github.com/grafana/xk6-output-influxdb@v0.3.0
```


### Windows

On Windows you can either build a native Windows binary, or, if you're using WSL2, a Linux binary you can use in WSL2.

For the native Windows binary if you're using PowerShell:
```powershell
docker run --rm -it -e GOOS=windows -u "$(id -u):$(id -g)" -v "${PWD}:/xk6" `
  grafana/xk6 build v0.43.1 --output k6.exe `
  --with github.com/mostafa/xk6-kafka@v0.17.0 `
  --with github.com/grafana/xk6-output-influxdb@v0.3.0
```

For the native Windows binary if you're using cmd.exe:
```batch
docker run --rm -it -e GOOS=windows -v "%cd%:/xk6" ^
  grafana/xk6 build v0.43.1 --output k6.exe ^
  --with github.com/mostafa/xk6-kafka@v0.17.0 ^
  --with github.com/grafana/xk6-output-influxdb@v0.3.0
```

For the Linux binary on WSL2, you can use the same command as for Linux.


## Local Installation

### Requirements

- [Go installed](https://golang.org/doc/install). At least version 1.17 is needed.

### Install xk6

```bash
$ go install go.k6.io/xk6/cmd/xk6@latest
```

This will install the `xk6` binary in your `$GOPATH/bin` directory.

If you're getting a `command not found` error when trying to run `xk6`, make sure that you precisely follow the [Go installation instructions](https://go.dev/doc/install) for your platform.
Specifically, ensure that the `$GOPATH/bin` directory is part of your `$PATH`. For example, you might want to add this to your shell's initialization file: `export PATH=$(go env GOPATH)/bin:$PATH`. See [this article](https://go.dev/doc/gopath_code#GOPATH) for more information.


## Command usage

The `xk6` command has two primary uses:

1. Compile custom `k6` binaries
2. A replacement for `go run` while developing k6 extensions

The `xk6` command will use the latest version of k6 by default. You can customize this for all invocations by setting the `K6_VERSION` environment variable.

As usual with `go` command, the `xk6` command will pass the `GOOS`, `GOARCH`, and `GOARM` environment variables through for cross-compilation.


### Custom builds

Syntax:

```
$ xk6 build [<k6_version>]
    [--output <file>]
    [--with <module[@version][=replacement]>...]
    [--replace <module=replacement>...]
```

- `<k6_version>` is the core k6 version to build; defaults to `K6_VERSION` env variable or whatever is the latest version needed by all extensions.
- `--output` changes the output file.
- `--with` can be used multiple times to add extensions by specifying the Go module name and optionally its version, similar to `go get`. Module name is required, but specific version and/or local replacement are optional. For an up-to-date list of k6 extensions, head to our [extensions page](https://k6.io/docs/extensions/).
- `--replace` can be used multiple times to add replacements by specifying the Go module name and the replacement module, similar to `go mod edit -replace=`. Version of the replacement can be specified with the `@version` suffix in the replacement path.

Examples:

```bash
$ xk6 build \
    --with github.com/grafana/xk6-browser

$ xk6 build v0.35.0 \
    --with github.com/grafana/xk6-browser@v0.1.1

$ xk6 build \
    --with github.com/grafana/xk6-browser=../../my-fork

$ xk6 build \
    --with github.com/grafana/xk6-browser=.

$ xk6 build \
    --with github.com/grafana/xk6-browser@v0.1.1=../../my-fork

# Build using a k6 fork repository. Note that a version is required if
# XK6_K6_REPO is a URI.
$ XK6_K6_REPO=github.com/example/k6 xk6 build master \
    --with github.com/grafana/xk6-browser

# Build using a k6 fork repository from a local path. The version must be omitted
# and the path must be absolute.
$ XK6_K6_REPO="$PWD/../../k6" xk6 build \
    --with github.com/grafana/xk6-browser
```

### For extension development

If you run `xk6` from within the folder of the k6 extension you're working on _without the `build` subcommand_, it will build k6 with your current module and run it, as if you manually plugged it in and invoked `go run`.

The binary will be built and run from the current directory, then cleaned up.

The current working directory must be inside an initialized Go module.

Also note that because of the way xk6 works, vendored dependencies (the vendor directory created by `go mod vendor`) will not be taken into account when building a binary, and you don't need to commit them to the extension repository.

Syntax:

```
$ xk6 <args...>
```
- `<args...>` are passed through to the `k6` command.

For example:

```bash
$ xk6 version
$ xk6 run -u 10 -d 10s test.js
```

The race detector can be enabled by setting the env variable `XK6_RACE_DETECTOR=1` or through the `XK6_BUILD_FLAGS` env variable.


## Library usage

```go
builder := xk6.Builder{
	K6Version: "v0.35.0",
	Extensions: []xk6.Dependency{
		{
			PackagePath: "github.com/grafana/xk6-browser",
			Version:     "v0.1.1",
		},
	},
}
err := builder.Build(context.Background(), "./k6")
```

Versions can be anything compatible with `go get`.


## Environment variables

Because the subcommands and flags are constrained to benefit rapid extension prototyping, xk6 does read some environment variables to take cues for its behavior and/or configuration when there is no room for flags.

- `K6_VERSION` sets the version of k6 to build.
- `XK6_BUILD_FLAGS` sets any go build flags if needed. Defaults to '-ldflags=-w -s'.
- `XK6_RACE_DETECTOR=1` enables the Go race detector in the build.
- `XK6_SKIP_CLEANUP=1` causes xk6 to leave build artifacts on disk after exiting.
- `XK6_K6_REPO` optionally sets the path to the main k6 repository. This is useful when building with k6 forks.


---

> This project originally forked from the [xcaddy](https://github.com/caddyserver/xcaddy) project. **Thank you!**
