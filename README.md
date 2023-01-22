## Compare package managers

Official benchs from [pnpm](https://pnpm.io/benchmarks) and [yarn 3+](https://yarnpkg.com/benchmarks) aren't
conclusive. Let's test them based on [nextjs-monorepo-example](https://github.com/belgattitude/nextjs-monorepo-example)
for fun with CI first approach. 

Potential for co2 emissions reductions at install, build and runtime (♻️🌳❤️) ?

### TLDR;

#### 📥 Install speed 

With cache PNPM 7.22.1 and Yarn 4.0.0-rc.36 (linker: node_modules, supportedArchitecture: current, compressionLevel: 0) seems equally fast on the CI.

See the action in [.github/workflows/ci-install-benchmark.yml](https://github.com/belgattitude/compare-package-managers/blob/main/.github/workflows/ci-install-benchmark.yml)
and the [history log](https://github.com/belgattitude/compare-package-managers/actions/workflows/ci-install-benchmark.yml). 

That said if you're deploying on vercel, hacks are needed to preserve the cache on lock changes. 

#### ⏩ Nextjs build speed and lambda size

Build the nextjs-app [standalone mode](https://nextjs.org/docs/advanced-features/output-file-tracing#automatically-copying-traced-files): **Yarn / Pnpm are euqivalent (+/- 1min)**. Curious ? See the action in
[.github/workflows/ci-build-benchmark.yml](https://github.com/belgattitude/compare-package-managers/blob/main/.github/workflows/ci-build-benchmark.yml) and
the [history log](https://github.com/belgattitude/compare-package-managers/actions/workflows/ci-build-benchmark.yml)

That said I couldn't reliably make prisma works with pnpm (without `auto-install-peers=true` wich increase the deps a lot), 
neither nextjs with standalone mode.

See the section "Debug size" in [.github/workflows/ci-build-benchmark.yml](https://github.com/belgattitude/compare-package-managers/blob/main/.github/workflows/ci-build-benchmark.yml) and
the [history log](https://github.com/belgattitude/compare-package-managers/actions/workflows/ci-build-benchmark.yml)

#### 🔢 Install size (in monorepo)

If you're not using nextjs (with standalone mode) things are different. Yarn does a better job at deduping and
also avoid installing native binaries (ie: glibc, musl) thx to [supportedArchitectures](https://yarnpkg.com/configuration/yarnrc#supportedArchitectures).
See also: https://github.com/pnpm/pnpm/issues/5504.

#### 🦺 Safe to use ?

They differ in the hoisting structure and the peer-dependency strategy. Expect different kind of edge cases. In my 
experience pnpm is sometimes harder to work with (ie prisma)

### Technicalities

- [Yarn 4.0.0-rc.36](https://yarnpkg.com/) - "Safe, stable, reproducible projects".
- [Pnpm 7.25.1](https://pnpm.io/) - "Fast, disk space efficient package manager".

Yarn support 3 module resolution algorithms (often called hoisting): node_modules, pnp and pnpm (alpha). Only the
`nodeLinker: node-modules` have been included in this test to prevent any compatibility issues. 
Keep in mind that the pnp linker will be the fastest in install and will even bring speedups at runtime.

Pnpm is a fast moving package manager recently endorsed by vercel that plays well in monorepos. 

### Aspects

- Reproducible ? 
  - Yarn lightweight js binary is committed within the repo (<3mb). That avoid bugs due to
    difference in versions. PNPM is too big to be stored in the repo, you need to install it.
    The experimental [corepack](https://nodejs.org/api/corepack.html)
    will bring solution to this. Both package managers already support corepack, but there's some time before a stable ecosystem shift.
  - Yarn is very picky about strictness, checksums... it's quite difficult to abuse. Something that is generally preferred in the enterprise world. 

- Fast ? 
  - On CI what makes the biggest different is to set up a cache. Simple and easy. 
    See [pnpm cache gist](https://gist.github.com/belgattitude/838b2eba30c324f1f0033a797bab2e31) and [yarn cache gist](https://gist.github.com/belgattitude/042f9caf10d029badbde6cf9d43e400a),
    that gives a >2x boost.    
  - It's super difficult to compare this. I would say in simple scenarios or whenever you deal with many versions of the same dep: PNPM rules. In monorepos with a couple of 'native' deps (ie: prisma, swc, esbuild...) Yarn should be faster.    
    
- Space efficient ? 
  - PNPM helps when many versions of the same library co-exists (ie: lodash...). It only stores
    the files that differs (creates a symlink instead). It might help in those situations (a symlink is 4kb so
    it gives best results if non-changed files are more than 4kb). So useful is some specific scenarios.
  - PNPM handles peer-deps differently. In some aspects it's more solid than yarn with node_modules
    but it does not dedupe them (leading to more versions installed). 
  - Yarn offers `supportedArchitectures`. When configured to current, it helps to not download native binaries for
    other platforms (ie: amd64 + musl for esbuild, swc...)

### Scenarios

The following have been tested on CI. Look for the results in the action history:

See results in [actions](https://github.com/belgattitude/compare-package-managers/actions)


### CI: With cache

> Example from a recent run

| CI Scenario             | Install | CI fetch cache | CI persist cache |  Setup | 
|-------------------------|--------:|---------------:|-----------------:|-------:|
| yarn4 mixed-compression |    ±40s |            ±7s |          *(±6s)* |     0s |
| yarn4 no compression    |    ±30s |            ±4s |          *(±9s)* |     0s |
| pnpm7                   |    ±19s |            ±9s |         *(±16s)* |     1s |


The CI fetch cache (time taken for the github action to load and extract the cache archive) 
differences happens can be explained by:

- PNPM dedupe differently that explains the difference in sizes (see below).
- YARN with supportedArchitectures: current does not download extra binaries (ie: linux+musl)
- When yarn use mixed-compression the internal github zstd don't compress the yarn zip archives.   

But in my experience the fetch cache is varying a lot between runs.

Thus: pnpm (19+9+1) = 29s vs yarn no-comp (14+4+0) = 18s.

https://github.com/belgattitude/compare-package-managers/actions/runs/3976413886/jobs/6816885534

![img.png](img.png)


Important to mention though

- on yarn.lock changes only yarn is able to start with the cache (download differences only and persist).
- ci persist cache only happens on yarn.lock changes, for example

<img src="https://user-images.githubusercontent.com/259798/199542234-f828450c-e8e4-4e61-b391-cc022adaa3eb.png" />

### CI: Without cache

> Data: https://github.com/belgattitude/compare-package-managers/actions/runs/3346115309/jobs/5542514593

| CI Scenario              | Install | Setup | 
|--------------------------|--------:|------:|
| yarn4 mixed-compression  |   ±140s |    0s |
| yarn4 no compression     |    ±54s |    0s |
| pnpm7                    |    ±50s |    1s | 

Disabling compression in yarn through [compressionLevel: 0](https://yarnpkg.com/configuration/yarnrc#compressionLevel) makes it twice faster. Makes sense as
the js zip compression brings an overhead on single-core cpu's. Pnpm results are very close, but as it does 
not dedupe as *well* (?) it actually downloads more packages (multiple typescript versions...). Difficult
comparison. Confirmed by gdu and action cache:

![image](https://user-images.githubusercontent.com/259798/213884689-3e550c69-0ca2-4551-b383-c99ebcbbbf8e.png)

### CI: action/cache

> First run data: https://github.com/belgattitude/compare-package-managers/actions/runs/3346115309/jobs/5542514593

**Warning** the fetch/persist times varies a lot between runs.

| CI Scenario              | Fetch cache | Persist cache |  Size | 
|--------------------------|------------:|--------------:|------:|
| yarn4 mixed-compression  |         ±5s |           ±6s | 220MB |
| yarn4 no compression     |         ±8s |           ±9s | 180MB |
| pnpm7                    |        ±14s |          ±16s | 273MB |

When already compressed the yarn cache is stored faster in the github cache. Make sense as action/cache won't 
try to compress something already compressed. On the pnpm side saving the pnpm-store is much slower, this is due
to different deduplication but also to the fact that it has to compress more files (+symlinks...). Note also
that regarding cache yarn has an advantage: it does not need to be recreated for different os/architectures. 

<details>
  <summary>Give me a screenshot of cache retrieval</summary>
  <img src="https://user-images.githubusercontent.com/259798/199530263-c443171b-0d47-4937-ab4b-a0382d4200f2.png" /> 
</details>

<details>
  <summary>Give me a screenshot of cache restoration</summary>
  <img src="https://user-images.githubusercontent.com/259798/199531335-34584af8-366e-477d-bc50-8016c734ad48.png" /> 
</details>


## Want to test locally ?

Use [hyperfine](https://github.com/sharkdp/hyperfine) and [taskset](https://man7.org/linux/man-pages/man1/taskset.1.html) 
to mimic single-core speed.

| Command | Mean [s] | Min [s] | Max [s] | Relative |
|:---|---:|---:|---:|---:|
| `taskset -c 0 npm run install:yarn-mixed-comp:cache` | 48.852 ± 1.146 | 47.977 | 50.347 | 1.56 ± 1.24 |
| `taskset -c 0 npm run install:yarn-no-comp:cache` | 38.916 ± 0.170 | 38.707 | 39.075 | 1.24 ± 0.99 |
| `taskset -c 0 npm run install:pnpm:cache` | 31.368 ± 24.988 | 20.147 | 76.067 | 1.00 |


```bash
hyperfine --runs=3 --export-markdown "docs/bench-yarn-vs-pnpm-single-core.md" \
--prepare "npm run install:yarn-mixed-comp; npx -y rimraf@3.0.1 '**/node_modules'" \
"taskset -c 0 npm run install:yarn-mixed-comp:cache" \
--prepare "npm run install:yarn-no-comp; npx -y rimraf@3.0.1 '**/node_modules'" \
"taskset -c 0 npm run install:yarn-no-comp:cache" \
--prepare "pnpm i  npx -y rimraf@3.0.1 '**/node_modules'" \
"taskset -c 0 npm run install:pnpm:cache" 
```

## Changelog

## Sponsors :heart:

If you are enjoying some this guide in your company, I'd really appreciate a [sponsorship](https://github.com/sponsors/belgattitude), a [coffee](https://ko-fi.com/belgattitude) or a dropped star.
That gives me some more time to improve it to the next level.

