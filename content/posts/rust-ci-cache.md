+++
title = "Rust CI Cache"
date = "2023-01-09T16:28:35+02:00"
author = "Bohdan Ivashko"
tags = ["rust", "ci", "cache"]
keywords = ["rust", "ci", "cache", "build", "ci/cd", "github"]
description = "Deeper dive into different options for caching rust builds on CI and their problems"
+++

Even though rust generally has better build times than C++ it still compares poorly to some other languages and CI workflows can take quite some time to run from scratch (i.e. one of the projects I've worked on took 40 minutes to run its' suite on github actions default workers, even though the project is far from being huge). Long workflows can reduce iteration speed, especially as teams grow, so generally we want to speed up the CI, and what's a better way to do that than to use a good old cache? Well, as it turns out, caching build dependencies and artifacts in rust may be not as straightforward as it seems at the first glance.

## General approach

If you were to take a look at the [cargo book](https://doc.rust-lang.org/cargo/index.html) you'd notice that it already covers the basics on how to cache downloaded [packages](https://doc.rust-lang.org/cargo/guide/cargo-home.html) and [build artifacts](https://doc.rust-lang.org/cargo/guide/build-cache.html). But if it was as simple as that this article wouldn't exist in the first place, so let's dig a bit deeper.

I'll be using a small personal project inspired by one of the services I had to do for work. Originally it was written in a different language but to represent it in rust I've decided to use axum as a web server, sled/rocksdb for storage, tracing for logging, and clap for cmdline args. It's a very simple service but it has enought dependencies to demonstrate the impact of different caching approaches. You can find the repo [here](https://github.com/arriven/rust-ci-playground).

The repo also includes a github actions workflow that has a couple of different cache configurations that I'll be comparing further.

## Caching downloaded packages

According to this [chapter](https://doc.rust-lang.org/cargo/guide/cargo-home.html) of the cargo book all you need to do in order to cache downloaded packages is to simply cache `$CARGO_HOME` directory, but to be more efficient you can cache only specific subdirectories in it (`registry/cache/`, `registry/index/`, and `git/db/`. it also specifies that you could cache `bin/` subdirectory but that one is used for installed tools and you would often prefer to use other means to get them faster, like using an action that downloads pre-built binaries or using a build image with all the tools already pre-installed (the latter has questionable results on [github](https://github.com/actions/cache/issues/924))). In order to test this I included a `home` cache action (yes, I'm not very creative when it comes to naming) into the repo that uses a following cache configuration:

```yaml
- uses: actions/cache@v3
  with:
    path: |
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/
      ~/.cargo/git/db/
    key: ${{ runner.os }}-home-${{ hashFiles('**/Cargo.lock') }}
```

**Note**: if you're wondering why the key uses hash of all the `Cargo.lock` files in the workspace - it's because caches in github actions are [immutable](https://github.com/actions/toolkit/issues/505) (yes, even if you use a [fork](https://github.com/martijnhols/actions-cache) that allows you to push them at will) so in order to have a newer entry you need to have a different key.

When running it, the first run that doesn't have any cache looks like this:

```bash
$ cargo test --locked --no-run
    Updating crates.io index
 Downloading crates ...
  Downloaded lazy_static v1.4.0
  Downloaded clap_derive v4.0.21
  # ...
  # a bunch of other crates being downloaded
  # ...
  Downloaded pin-project-lite v0.2.9
   Compiling proc-macro2 v1.0.49
   Compiling unicode-ident v1.0.6
   # ...
   # a bunch of other crates being compiled
   # ...
   Compiling tracing-subscriber v0.3.16
   Compiling linkmapper v0.1.0 (/home/runner/work/rust-ci-playground/rust-ci-playground)
    Finished test [unoptimized + debuginfo] target(s) in 1m 09s
  Executable unittests src/main.rs (target/debug/deps/linkmapper-da17f15a9210ca29)
```

When running it for the second time the logs look slightly different:

```bash
Run cargo test --locked --no-run
   Compiling proc-macro2 v1.0.49
   Compiling unicode-ident v1.0.6
   # ...
   # a bunch of other crates being compiled
   # ...
   Compiling tracing-subscriber v0.3.16
   Compiling linkmapper v0.1.0 (/home/runner/work/rust-ci-playground/rust-ci-playground)
    Finished test [unoptimized + debuginfo] target(s) in 52.84s
  Executable unittests src/main.rs (target/debug/deps/linkmapper-da17f15a9210ca29)
```

As you can notice, we no longer have to update crates.io index and it doesn't need to re-download the crates. This shaves about 30-45s off of the overall build time, which is nice. The cache ends up taking 86mb of space after being compressed by github (100mb for the `rocksdb` version), so it probably won't result in any cache eviction [issues](https://docs.github.com/en/actions/using-workflows/caching-dependencies-to-speed-up-workflows#force-deleting-cache-entries).

## Caching build artifacts

Now let's take a look at another [chapter](https://doc.rust-lang.org/cargo/guide/build-cache.html) of the cargo book. According to it we can cache the build artifacts by caching the `target` directory of our workspace or by using some third-party tools like [sccache](https://github.com/mozilla/sccache) (**Note**: there are also other ways to achieve this if you use build systems other than cargo but I won't cover that here). All of the approaches benefit from disabling incremental build since it only slows down compilation and doesn't work well with caching.

### **`target`-based cache**

I'll be using the following cache configuration to test it out (called `home_and_target` in the workflow file):

```yaml
- uses: actions/cache@v3
  with:
    path: |
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/
      ~/.cargo/git/db/
      target/
    key: ${{ runner.os }}-home_and_target-${{ hashFiles('**/Cargo.lock') }}
```

Again, the first run didn't show anything unexpected, without the cache being populated we still have to download and compile everything:

```bash
$ cargo test --locked --no-run
    Updating crates.io index
 Downloading crates ...
  Downloaded lazy_static v1.4.0
  Downloaded clap_derive v4.0.21
  # ...
  # a bunch of other crates being downloaded
  # ...
  Downloaded pin-project-lite v0.2.9
   Compiling proc-macro2 v1.0.49
   Compiling unicode-ident v1.0.6
   # ...
   # a bunch of other crates being compiled
   # ...
   Compiling tracing-subscriber v0.3.16
   Compiling linkmapper v0.1.0 (/home/runner/work/rust-ci-playground/rust-ci-playground)
    Finished test [unoptimized + debuginfo] target(s) in 1m 09s
  Executable unittests src/main.rs (target/debug/deps/linkmapper-da17f15a9210ca29)
```

On the second run the situation is a lot better, though:

```bash
$ cargo test --locked --no-run
   Compiling linkmapper v0.1.0 (/home/runner/work/rust-ci-playground/rust-ci-playground)
    Finished test [unoptimized + debuginfo] target(s) in 3.73s
  Executable unittests src/main.rs (target/debug/deps/linkmapper-da17f15a9210ca29)
```

As you can notice, now we didn't have to neither download nor compile any of the crates. But wait, our own code didn't change between the runs too, shouldn't it be cached? Why do we see `Compiling linkmapper v0.1.0 ...` in the logs? Well, as it turns out, cargo uses mtime to detect whether the source code has been changed and since CI runners clone the repo from scratch for each job it results in updated mtime (dependencies are unpacked with tar which saves mtime and aren't affected by this). There's an [open issue](https://github.com/rust-lang/cargo/issues/6529) to address that but for now if your project has a lot of custom code compared to dependencies you can restore the mtime manually to the time of the commit using [this](https://github.com/chetan/git-restore-mtime-action) github action or its' [underlying script](https://github.com/chetan/git-restore-mtime-action/blob/master/src/git-restore-mtime-bare) if you use some other CI systems (thankfully the script doesn't have a lot of dependencies). My updated cache configuration looks like this:

```yaml
- uses: actions/cache@v3
  with:
    path: |
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/
      ~/.cargo/git/db/
      target/
    key: ${{ runner.os }}-home_and_target-${{ hashFiles('**/Cargo.lock') }}
- uses: chetan/git-restore-mtime-action@v1
  if: ${{ inputs.restore_mtime }}
```

Running it again finally gives us the expected result, and the overall build becomes almost instant for the perfect cache hit:

```bash
$ cargo test --locked --no-run
    Finished test [unoptimized + debuginfo] target(s) in 0.62s
  Executable unittests src/main.rs (target/debug/deps/linkmapper-da17f15a9210ca29)
```

I've never seen this approach being used even though its' only downside I know of is that it completely replicates the way `cargo` caches dependencies locally: if you ever had cases where `cargo` didn't pick up changes to the code introduced by `git pull` or similar - there's a non-zero chance you can encounter that on CI. Manually cleaning caches on github is fairly easy (unless you have lots of them) and you can use PR labels to conditionally disable cache usage so my guess is that this approach isn't widely adopted because most people just don't know about it (this is kind of confirmed by people who were reviewing the draft of this article).

**Except if we use `rocksdb` instead of `sled`**

For some reason not including `$CARGO_HOME/registry/src` to the cache leads to full recompilation of `librocksdb-sys` and everything that depends on it (~I suspect `bindgen` as the culprit but this needs further testing~ Edit: it turns out there's an [issue](https://github.com/rust-lang/cargo/issues/11083) and it's caused by `rocksdb` using `rerun-if-changed` on a folder in `build.rs`):

```bash
$ cargo test --locked --no-run
   Compiling librocksdb-sys v0.8.0+7.4.4
   Compiling rocksdb v0.19.0
   Compiling linkmapper v0.1.0 (/home/runner/work/rust-ci-playground/rust-ci-playground)
    Finished test [unoptimized + debuginfo] target(s) in 9m 48s
  Executable unittests src/main.rs (target/debug/deps/linkmapper-6393465c78bf12f8)
```

Yes, it takes more than 9 minutes to compile `rocksdb` on the github runner. This is definitely not something we want if we do need to use rocksdb. Fortunatelly, including the whole `$CARGO_HOME` into the cache solves this issue (`full` cache configuration in the repo):

```yaml
- uses: actions/cache@v3
  with:
    path: |
      ~/.cargo/
      target/
    key: ${{ runner.os }}-full-${{ hashFiles('**/Cargo.lock') }}
- uses: chetan/git-restore-mtime-action@v1
  if: ${{ inputs.restore_mtime }}
```

**Note:** based on my [tests](https://github.com/arriven/rust-ci-playground/actions/runs/3782411644/jobs/6430124523) just adding `$CARGO_HOME/registry/src` to the `home_and_target` cache is enought to fix this, but since `$CARGO_HOME` content is unstable it's safer to just cache the full thing, it's not that much overhead compared to the `target` folder size.

**Note 2:** similar issue occurs if you have a git-based dependency and don't include `$CARGO_HOME/git/checkouts` into the cache, except in that case it also forces cargo to re-download any git submodule that package depends on

```bash
$ cargo test --locked --no-run
    Finished test [unoptimized + debuginfo] target(s) in 1.86s
  Executable unittests src/main.rs (target/debug/deps/linkmapper-6393465c78bf12f8)
```

The resulting cache size is 250 mb (830 mb for `rocksdb` version).

`target`-based cache can end up being a bit bloated and I've definitely seen a project where the time saved on build was negated by the time it took to sync the cache (to be fair, the cache was reaching 5GB at some point in compressed state and the project was using self-hosted gitlab runners which are more powerful than default github runners but have worse caching techniques). You also probably want to use something like [cargo sweep](https://github.com/holmgr/cargo-sweep) to avoid infinite growth of the cache with old build artifacts unless you clean it up regularly or use more elaborate caching schemes (like building the cache from scratch on push to main and only restoring it in PRs).

If you go down this road and only care about caching dependencies (and don't have dependencies like `rocksdb`) you can also use [Swatinem/rust-cache](https://github.com/Swatinem/rust-cache) which uses the same caching scheme as `home_and_target` but builds a more sensitive cache key and cleans up the cached directories from old build artifacts and non-deps objects.

**Note**: it's possible to only cache the `target` dir and avoid caching `$CARGO_HOME` and it even will result in almost no recompilation (`rocksdb` is one of the exceptions again) but it's a rather weird caching setup, especially considering the size difference between cargo home and build artifacts for larger projects. Still, here's a build log for it to confirm that it works as expected if you're interested:

```bash
$ cargo build
    Updating crates.io index
  Downloaded fnv v1.0.7
  Downloaded futures-channel v0.3.25
  # ...
  # a bunch of other crates being downloaded
  # ...
  Downloaded linux-raw-sys v0.1.4
  Downloaded 96 crates (6.6 MB) in 1.18s
    Finished dev [unoptimized + debuginfo] target(s) in 39.47s
```

### **Using `sccache`**

Mozilla's [sccache](https://github.com/mozilla/sccache) uses a different approach for caching build artifacts - it acts as a distributed compilation cache, that can be backed by a local or cloud storage. It still leads to all the packages being recompiled (from the cargo perspective), but that compilation takes noticeably less time. It also doesn't affect calls to linker. If you end up pointing it to a cloud-based cache you could even set it up on your dev machines to speed up local compilation time.

Although I might be unlucky, `sccache` does seem to slow down the initial compilation time a bit (5-10%). Other than that there are no suprises here, it results in worse compilation time on a 'perfect' cache hit than the `target`-base approach (about 5x speedup from the raw compilation time but it would be hard to compete with skipping the compilation completely), and the cache ends up taking only 170mb (490 mb for `rocksdb` version), which might end up being crucial for some projects (i.e. for the project where the `target`-based cache ended up taking 5GBs of space `sccache` ended up taking less than 1GB). Since `sccache` requires a bit more setup than other methods I'll include the full action here:

```yaml
sccache:
  runs-on: ubuntu-latest
  if: ${{ inputs.sccache }}
  env:
    RUSTC_WRAPPER: sccache
    SCCACHE_CACHE_SIZE: "1G"
  steps:
    - uses: actions/checkout@v2
    - uses: actions-rs/toolchain@v1
      with:
        toolchain: stable
        override: true
    - uses: taiki-e/install-action@cargo-binstall
    - run: cargo binstall --no-confirm --no-symlinks sccache
    - uses: actions/cache@v3
      with:
        path: |
          ~/.cargo/registry/index/
          ~/.cargo/registry/cache/
          ~/.cargo/git/db/
          ~/.cache/
        key: ${{ runner.os }}-sccache-${{ hashFiles('**/Cargo.lock') }}
    - uses: chetan/git-restore-mtime-action@v1
      if: ${{ inputs.restore_mtime }}
    - run: cargo test --locked --no-run
    - run: cargo test --locked --no-fail-fast
    - run: cargo clippy --locked --workspace --tests --no-deps -- -D warnings
```

## Results

In the end, I'd like to bring you a comparison table of all the approaches covered here. Please note that Github Actions public runners can fluctuate in performance so I did notice some runs take up to 25% more or less time to complete without any changes to the code or action but it should still be enough to see general tendencies (it does lead to some weird results like when enabling mtime restore for `sccache` action leads to 20-40s slowdown, even though mtime doesn't affect the work done there in the slightest)

### `cache profile` overview

I already tried to reference each profile in the article but it probably won't hurt to also have these definitions next to where they are used so you don't have to look for them each time

- `home` - only caching recommended folders from `$CARGO_HOME` without any caching of build artifacts (eliminates `crates.io` sync)
- `home_and_target` - caching recommended folders from `$CARGO_HOME` + whole `target` folder
- `full` - caching whole `$CARGO_HOME` + whole `target` folders
- `swatinem` - using `Swatinem/rust-cache@v2` as is (it uses the same set of folders as `home_and_target` but does additional cleanup to only cache dependencies)
- `sccache` - using `mozilla/sccache` and caching recommended folders from `$CARGO_HOME` (`$CARGO_HOME/git/db`, `$CARGO_HOME/registry/index`, `$CARGO_HOME/registry/cache`) + sccache cache folder (`$SCCACHE_DIR`) without caching build artifacts themselves

### Using `sled` for storage (happy path)

| cache profile   | cache size | toolchain + job setup time | cache sync time | first run (no cache) | second run (cache, no mtime restore) | third run (cache, mtime restore) |
| --------------- | ---------- | -------------------------- | --------------- | -------------------- | ------------------------------------ | -------------------------------- |
| home            | 86 mb      | 15s                        | 2s              | 1m 50s               | 1m 27s                               | 1m 27s                           |
| home_and_target | 250 mb     | 15s                        | 8s              | 2m 8s                | 31s                                  | 29s                              |
| full            | 290 mb     | 15s                        | 13s             | 1m 56s               | 38s                                  | 31s                              |
| swatinem        | 250 mb     | 15s                        | 9s              | 1m 49s               | 34s                                  | 27s                              |
| sccache         | 170 mb     | 25s                        | 5s              | 2m 5s                | 1m 1s                                | 1m 18s                           |

### Using `rocksdb` for storage (unhappy path)

| cache profile   | cache size | toolchain + job setup time | cache sync time | first run (no cache) | second run (cache, no mtime restore) | third run (cache, mtime restore) |
| --------------- | ---------- | -------------------------- | --------------- | -------------------- | ------------------------------------ | -------------------------------- |
| home            | 100 mb     | 15s                        | 2s              | 8m 53s               | 8m 26s                               | 8m 38s                           |
| home_and_target | 780 mb     | 15s                        | 25s             | 10m 11s              | 10m 40s                              | 8m 28s                           |
| full            | 830 mb     | 15s                        | 26s             | 9m 38s               | 56s                                  | 47s                              |
| swatinem        | 730 mb     | 15s                        | 23s             | 9m 0s                | 8m 26s                               | 10m 16s                          |
| sccache         | 490 mb     | 25s                        | 10s             | 13m 58s              | 2m 10s                               | 2m 50s                           |

As you can notice, until [this](https://github.com/rust-lang/cargo/issues/11083) is resolved, the only ways to save time caching packages like `rocksdb` are caching the whole `$CARGO_HOME` folder or using a third-party tool like `sccache`

**Shouts out to [Predrag Gruevski](https://hachyderm.io/@predrag), [Anthony Bondarenko](https://github.com/yneth), and a few folks who preferred to remain unnamed, for feedback on drafts of this post. Any mistakes are mine alone.**
