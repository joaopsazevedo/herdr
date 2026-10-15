# Building this fork

This fork is upstream [herdr](https://github.com/herdrdev/herdr) plus local
patches. Its `master` is always upstream `master` with these patch commits on
top (rebased, no merge commits):

- **fix: inherit workspace launch env in new panes.** With upstream herdr,
  `herdr workspace create --env MY_VAR=foo` sets `MY_VAR` only on the first
  pane. This patch stores the env on the workspace (and in the saved session),
  so split panes, new tabs and restored panes get it too.

These steps build it the same way on any Linux or macOS machine.

## 1. Get the code (once per machine)

```bash
git clone https://github.com/joaopsazevedo/herdr.git
cd herdr
git remote add upstream https://github.com/herdrdev/herdr.git
git config user.email joaopsazevedo@outlook.com   # this repo only
git config push.followTags false   # never push upstream's tags to the fork
```

Keep `push.followTags` off: with it on, a push also sends upstream's release
tags, and a tag push can start the fork's copy of upstream's Release workflow.

## 2. Install the toolchain (once per machine)

| Tool | Version | Needed for |
|---|---|---|
| Rust (rustup) | pinned in `rust-toolchain.toml` | build |
| Zig | **exactly** the pin in `vendor/libghostty-vt/build.zig.zon` | build |
| just | any recent | build, tests |
| cargo-nextest | any recent | tests |
| bun | 1.3.14 (the CI pin) | tests |
| python3 | 3.11 or newer | tests |
| node | 22 LTS | some docs tests |

**Rust:** install [rustup](https://rustup.rs). On the first `cargo` command in
the repo, rustup installs the pinned toolchain with clippy and rustfmt.

**Zig:** the version must match the pin exactly. Zig 0.15.2 and 0.16.0 each
reject a tree pinned to the other. Run this from the repo root. It reads the
pin, downloads that version from ziglang.org, and checks the sha256:

```bash
ZIG_VERSION=$(sed -n 's/.*minimum_zig_version = "\([^"]*\)".*/\1/p' vendor/libghostty-vt/build.zig.zon)
case "$(uname -s)-$(uname -m)" in
  Linux-x86_64)  ZIG_TARGET=x86_64-linux ;;
  Linux-aarch64) ZIG_TARGET=aarch64-linux ;;
  Darwin-arm64)  ZIG_TARGET=aarch64-macos ;;
  Darwin-x86_64) ZIG_TARGET=x86_64-macos ;;
esac
read -r ZIG_URL ZIG_SHA < <(curl -fsSL https://ziglang.org/download/index.json |
  python3 -c 'import json,sys; e=json.load(sys.stdin)[sys.argv[1]][sys.argv[2]]; print(e["tarball"], e["shasum"])' \
  "$ZIG_VERSION" "$ZIG_TARGET")
mkdir -p ~/.local/toolchains/zig-$ZIG_VERSION &&
  curl -fsSLo /tmp/zig.tar.xz "$ZIG_URL" &&
  [ "$( (sha256sum /tmp/zig.tar.xz 2>/dev/null || shasum -a 256 /tmp/zig.tar.xz) | cut -d' ' -f1)" = "$ZIG_SHA" ] &&
  tar -xJf /tmp/zig.tar.xz -C ~/.local/toolchains/zig-$ZIG_VERSION --strip-components=1 &&
  rm /tmp/zig.tar.xz &&
  ~/.local/toolchains/zig-$ZIG_VERSION/zig version
```

The build finds Zig through the `ZIG` environment variable, or `zig` on
`PATH`:

```bash
export ZIG=~/.local/toolchains/zig-0.16.0/zig   # use the version you installed
```

**just and cargo-nextest** (prebuilt binaries, into `~/.cargo/bin`):

```bash
curl --proto '=https' --tlsv1.2 -fsSL https://just.systems/install.sh | bash -s -- --to ~/.cargo/bin
curl -fsSL https://get.nexte.st/latest/linux | tar -xzf - -C ~/.cargo/bin   # macOS: .../latest/mac
```

**bun:** `curl -fsSL https://bun.sh/install | bash -s "bun-v1.3.14"`, then add
`~/.bun/bin` to `PATH`.

**node:** any Node 22 install, for example from nodejs.org or your package
manager.

## 3. Build and install

```bash
export ZIG=~/.local/toolchains/zig-<version>/zig
just build                         # cargo build --release --locked
target/release/herdr --version
```

The binary is `target/release/herdr`. To install it:

```bash
cp -a target/release/herdr ~/bin/herdr
herdr server stop                  # the running server keeps the old binary until it stops
```

## 4. Update to the latest upstream

On the machine where you do the update:

```bash
git checkout master
git fetch upstream
git rebase upstream/master
grep minimum_zig_version vendor/libghostty-vt/build.zig.zon   # did the Zig pin change?
just build
git push --force-with-lease origin master
```

The rebase rewrites `master`, so the push needs `--force-with-lease`. If the
rebase stops on a conflict, fix it, run the patch tests (below), then
`git add` the file and `git rebase --continue`.

To add a new patch, commit it on `master` and push. It then gets rebased with
the others.

On your other machines, `git pull` fails after such an update, because the old
and new `master` have different histories. Take the new one instead (this
throws away local changes on `master`):

```bash
git fetch origin
git reset --hard origin/master
```

## 5. Test

Only the tests of the local patch (fast):

```bash
cargo nextest run --locked -E 'test(workspace::tests::) or test(persist::snapshot::tests::)'
```

The full Linux/macOS suite (what upstream CI runs, without the Windows lint):

```bash
just ci
just docs-contract-test
```

`just check` runs all of the above plus a lint of the Windows target. That
lint needs the Windows SDK, downloaded once per machine:

```bash
cargo install xwin --locked
just setup-windows-cross           # asks you to accept Microsoft's SDK license
just check
```

To try a debug build from inside a running herdr session, clear the inherited
socket variables, or the new binary talks to your live server:

```bash
env -u HERDR_SOCKET_PATH -u HERDR_CLIENT_SOCKET_PATH cargo run -- <command>
```

## Machine notes

- **Many CPUs but a low process limit** (for example a container that sees 256
  CPUs with a cgroup `pids.max` of 2048): the test suite starts many threads per
  visible CPU and runs out. Symptoms: "socket did not appear" timeouts, or
  "failed to spawn thread: Resource temporarily unavailable". Run the tests
  under `taskset -c 0-15`, for example `taskset -c 0-15 just ci`. Compare
  `cat /sys/fs/cgroup/pids.max` with `nproc` to check.
- **Containers that reset the home directory:** tools in `~/.rustup`,
  `~/.cargo/bin`, `~/.bun` and `~/.local/toolchains` can disappear. Check them
  before a build. rustup reinstalls the Rust toolchain by itself; Zig and just
  must be downloaded again.
- **Memory:** linking the test binary needs more than 2 GiB of RAM.
- **aarch64 Linux:** with Zig 0.15.2, the SIMD C++ code in libghostty-vt failed
  to compile. `LIBGHOSTTY_VT_SIMD=false` worked around it. Not checked with Zig
  0.16.0; try the default first.

## Optional: a Podman build image

For a build environment that survives machine resets. Set `ZIG_VERSION` and
`ZIG_SHA256` from the pin and from https://ziglang.org/download/index.json.

```dockerfile
FROM docker.io/library/rust:1.96.1
ARG ZIG_VERSION=0.16.0
ARG ZIG_SHA256=70e49664a74374b48b51e6f3fdfbf437f6395d42509050588bd49abe52ba3d00

RUN apt-get update \
 && apt-get install -y --no-install-recommends python3 xz-utils unzip nodejs \
 && rm -rf /var/lib/apt/lists/*
RUN rustup component add clippy rustfmt
RUN curl -fsSLo /tmp/zig.tar.xz https://ziglang.org/download/${ZIG_VERSION}/zig-x86_64-linux-${ZIG_VERSION}.tar.xz \
 && echo "${ZIG_SHA256}  /tmp/zig.tar.xz" | sha256sum -c - \
 && mkdir -p /opt/zig && tar -xJf /tmp/zig.tar.xz -C /opt/zig --strip-components=1 \
 && ln -s /opt/zig/zig /usr/local/bin/zig && rm /tmp/zig.tar.xz
RUN curl -fsSL https://bun.sh/install | bash -s "bun-v1.3.14" \
 && ln -s /root/.bun/bin/bun /usr/local/bin/bun
RUN curl --proto '=https' --tlsv1.2 -fsSL https://just.systems/install.sh | bash -s -- --to /usr/local/bin \
 && curl -fsSL https://get.nexte.st/latest/linux | tar -xzf - -C /usr/local/bin
```

Save it outside the repo as `Containerfile`, then build and run it with
rootless Podman:

```bash
podman build -t herdr-buildenv -f Containerfile .
podman run --rm -it -v "$PWD":/work -w /work \
  -v herdr-cargo:/usr/local/cargo/registry herdr-buildenv just build
```

The binary is in `target/release/herdr`, as with a normal build. Add
`--security-opt label=disable` on SELinux hosts if the mount is refused.
