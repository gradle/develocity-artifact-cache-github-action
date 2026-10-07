# Develocity Artifact Cache GitHub Action

Cache Gradle, Maven, npm, pip, and Sonar dependencies and build outputs in GitHub
Actions with the [Develocity Artifact Cache](https://docs.develocity.ai/artifact-cache/),
in a single workflow step. The action restores the cache before your build and stores
it afterward automatically.

## Why use this action?

The action integrates the Develocity Artifact Cache the way you already integrate other
tooling in GitHub Actions: a single step you add to a job, in the same shape as
`setup-gradle`, `setup-node`, or `setup-java`. It manages the whole cache lifecycle for
you:

- **One step, both phases.** The action restores in its main step and stores in an
  automatic post step. You do not write or order a separate store step, and cache
  activity never fails the build.
- **Every cache type, autodetected.** It caches Gradle, Maven, npm, pip, and Sonar
  content automatically, with no per-type configuration.
- **Cache images that work out of the box.** Images are generated per job and isolated by
  runner operating system and architecture, with smart fallbacks to the base branch and
  the repository default branch, so new branches restore instead of starting cold.
- **A verified, tested CLI.** The action downloads a checksum-verified CLI JAR and defaults
  to the version pinned and tested for this release, so you do not track CLI versions or
  download URLs yourself.
- **Visible results.** A run's cache activity is summarized as a table in the job summary
  (whenever the action reaches the cache), and Gradle and Maven builds surface the same
  metrics in their Build Scan.

Already driving the CLI by hand? See
[Migrating from a custom setup](#migrating-from-a-custom-artifact-cache-cli-setup).

## Quick start

Add the action to a job **once**, before the steps whose dependencies you want cached.
It runs in two automatic phases:

- its **main** step, where you place it in the job, **restores** the cache, and
- a **post** step that GitHub runs at the **end of the job** **stores** the cache.

You do **not** add a separate step to store. The post step is registered automatically,
the same model as [`actions/cache`](https://github.com/actions/cache) and
[`gradle/actions`](https://github.com/gradle/actions).

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7

      - uses: actions/setup-java@v5 # provides the JDK your build runs on
        with:
          distribution: temurin
          java-version: '21'

      - uses: gradle/develocity-artifact-cache-github-action@v1
        with:
          develocity-url: https://develocity.example.com
          develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}

      - run: ./gradlew build # uses the restored cache; its output is stored after the job
```

Place the action **before** your build so the restore happens first. The store then runs
only when the **job succeeds** (it is skipped if a job step failed) and **never fails the
build**. To make the cache **read-only** (restore, but never store), set
`cache-read-only: true`.

### Using with `setup-gradle`

When your workflow also uses [`gradle/actions/setup-gradle`](https://github.com/gradle/actions),
two things matter:

1. **Apply `setup-gradle` before this action.** `setup-gradle` establishes the Gradle User
   Home for the job — it creates the directory, writes its init scripts, and exports
   `GRADLE_USER_HOME`. Running it first lets this action's CLI auto-detect that same Gradle
   User Home, so the cache is **restored into, and stored from, the location your Gradle
   build actually uses**. If this action ran first, it would resolve the default Gradle User
   Home, which may not match the one `setup-gradle` goes on to configure — notably on
   Windows, where `setup-gradle` can relocate it.
2. **Disable `setup-gradle`'s own Gradle User Home caching.** This action already caches the
   Gradle dependencies that `setup-gradle`'s cache would otherwise store; disabling it defers
   to this action and avoids both mechanisms caching the same content. On `gradle/actions`
   6.4.0 or newer, set `cache-provider: external`, which keeps the job summary accurate by
   reporting that caching is handled externally rather than turned off; on older versions, set
   `cache-disabled: true`.

```yaml
      - uses: gradle/actions/setup-gradle@v6
        with:
          cache-provider: external # defer dependency caching to the Artifact Cache action

      - uses: gradle/develocity-artifact-cache-github-action@v1
        with:
          develocity-url: https://develocity.example.com
          develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}

      - run: ./gradlew build
```

## Inputs

| Input | Required | Default | Description |
| --- | --- | --- | --- |
| `develocity-url` | one of¹ | (none) | Develocity server URL, passed to the CLI as `--dv-server`. |
| `develocity-edge-url` | one of¹ | (none) | Develocity Edge URL, passed to the CLI as `--dv-edge` to target an Edge directly instead of a server. Whichever endpoint is set also identifies the Develocity host for the access-key lookup and short-lived-token exchange. |
| `develocity-auth` | no | (none) | `workload-identity` always authenticates with a GitHub OIDC token, ignoring `develocity-access-key`. Unset or `access-key` uses `develocity-access-key` when it is set and falls back to workload identity otherwise. Any other value fails the step (see [Authentication](#authentication)). |
| `develocity-access-key` | no | (none) | Access key for the chosen endpoint host (`develocity-url` or `develocity-edge-url`) in `host=key` form (the only key source). The action exchanges it for a short-lived token used for its own cache restore/store, and, when the AC CLI is downloaded via the edge route, as the bearer credential for that download — it is not exported to other steps. When unset or blank, the action falls back to [workload identity](#workload-identity); if that fails too, it warns and skips. Ignored when `develocity-auth` is `workload-identity` (see [Authentication](#authentication)). |
| `develocity-oidc-audience` | no | origin of the chosen endpoint | Audience of the GitHub OIDC token the action mints; must match the workload identity rule. Ignored when the action authenticates with `develocity-access-key` (see [Workload identity](#workload-identity)). |
| `develocity-token-expiry` | no | `2` | Lifetime, in hours, of the short-lived token obtained from the access key or OIDC token. Raise it only if a build could run long enough for the token to expire before the post-step store (see [Authentication](#authentication)). |
| `image-names` | no | (none) | Ordered image names, one per line: the first is the primary (stored under, tried first on restore), the rest are restore-only fallbacks. Replaces the auto-generated name (see [Cache image names](#cache-image-names)). |
| `cache-read-only` | no | `false` | When `true`, restore only: the post step skips the store. |
| `cli-version` | no | pinned general-availability version | Artifact Cache CLI version to download. Defaults to the tested version the action pins; usually leave unset (see [Versioning](#versioning)). |
| `cli-repository` | no | public URL, or the edge route when an edge-capable Develocity node is detected | JAR download source: the directory holding the JARs (see [Downloading the Artifact Cache CLI](#downloading-the-artifact-cache-cli) for the layout). Set to an internal mirror for air-gapped runners; an explicit value always wins over the edge route (see [Downloading the Artifact Cache CLI](#downloading-the-artifact-cache-cli)). Must be an `https` URL unless `cli-repository-allow-insecure` is set. |
| `cli-repository-header` | no | (none) | A single `Name: Value` HTTP header for an authenticated mirror. Supply via a secret. Not sent on the edge route, which authenticates with its own bearer token instead — but still validated, so a malformed value still fails the run. |
| `develocity-url-allow-insecure` | no | `false` | Allow the endpoint (`develocity-url` or `develocity-edge-url`) to be a plain-`http` URL. The default warns and skips before any credential is sent, since over `http` the access key or OIDC token travels in cleartext. Matches the Jenkins plugin's `develocityUrlAllowInsecure` (see [Security](#security)). |
| `allow-untrusted` | no | `false` | Accept the endpoint's TLS certificate without verifying it, for the token exchange, the CLI download from its Edge node, and the CLI itself (`--allow-untrusted-server`). A `cli-repository` mirror and redirects away from the endpoint are still verified. Matches the Jenkins plugin's `allowUntrusted` (see [Security](#security)). |
| `cli-repository-allow-insecure` | no | `false` | Allow the CLI download over plain `http`: from a `cli-repository` mirror, or from the Edge node at an `http` endpoint (which also needs `develocity-url-allow-insecure`). The default rejects `http`; enable only for trusted internal CI without TLS (over `http` the download and any `cli-repository-header` credential, or on the edge route the bearer token, are sent in cleartext). |
| `cli-sha256` | no | (built in) | Expected JAR SHA-256; needed only to verify a version the action does not already ship a checksum for. |
| `additional-cli-args` | no | (none) | Extra CLI arguments (one per line) appended to restore and store. The action-managed flags (`--dv-server`, `--image-name`, `--cache-metrics-file`, `--dv-edge`, and `--allow-untrusted-server`, which `allow-untrusted` sets) are rejected. |

¹ Provide exactly one of `develocity-url` or `develocity-edge-url` — they are mutually exclusive.

## Authentication

The action authenticates to the endpoint it targets (whichever of `develocity-url` or
`develocity-edge-url` you set) with one of two credentials: an access key, or a GitHub OIDC
token the action mints itself ([workload identity](#workload-identity)). Which one it uses
depends on `develocity-auth` and on whether `develocity-access-key` is set:

| `develocity-auth` | `develocity-access-key` | Credential |
| --- | --- | --- |
| unset or `access-key` | set | the access key |
| unset or `access-key` | unset or blank | a GitHub OIDC token (the fallback) |
| `workload-identity` | either (ignored) | a GitHub OIDC token |

Any other `develocity-auth` value **fails the step**. An access key needs only the secret it
is stored in. A GitHub OIDC token needs Develocity 2026.3 or later, a workload identity rule,
`id-token: write` on the job, and `develocity-edge-url`.

Most workflows leave `develocity-auth` unset. Set it to `workload-identity` when a key may
still reach the action, for example from a shared workflow that passes one to every caller,
and the job should authenticate with workload identity anyway.

The action logs which credential it used. Outside `develocity-auth: workload-identity`, a key
that is set is always used, never replaced by an OIDC token, even when it has no entry for the
endpoint host. If neither credential is available, the action **warns and skips all cache
activity for the job** rather than failing the build. That includes pull requests from forks,
which GitHub gives neither secrets nor OIDC tokens.

### Access key

Supply the key through the **`develocity-access-key` input**, the **only**
key source: the action does **not** read the `DEVELOCITY_ACCESS_KEY` environment variable or
`keys.properties`, so it never authenticates with an ambient key you did not explicitly pass
to it.

```yaml
      - uses: gradle/develocity-artifact-cache-github-action@v1
        with:
          develocity-url: https://develocity.example.com
          develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}
```

Provide it in **`host=key` form** (e.g. `develocity.example.com=<key>`, or a
`;`-separated `host1=key1;host2=key2` list), matching the `DEVELOCITY_ACCESS_KEY` format
used across the Develocity ecosystem (the DV build agents, `keys.properties`, and
`setup-gradle`). A bare key value is not accepted. The action uses the entry for the
chosen endpoint host, so a key for another host is never sent to it.

The action is fail-safe: if the key is not in `host=key` form or has no entry for the chosen
endpoint host, the action **warns and skips all cache activity for the job** rather than
failing the build.

### Workload identity

Requires Develocity **2026.3 or later**. The action uses workload identity when
`develocity-auth` is `workload-identity`, or when `develocity-access-key` is not set. Instead
of a long-lived access key kept in a secret, it mints a short-lived OIDC token from
[GitHub's OIDC provider](https://docs.github.com/en/actions/concepts/security/openid-connect)
and exchanges it with Develocity, so there is no access key to store as a GitHub Actions
secret. Three things have to be in place:

1. **A workload identity rule on the Develocity server** that accepts the token. Develocity
   validates it against the issuer, audience, and claim requirements on the matching rule,
   and the rule decides what the token grants. The issuer is
   `https://token.actions.githubusercontent.com` on GitHub.com, and
   `https://HOSTNAME/_services/token/actions` on GitHub Enterprise Server. Set the rule's
   audience to the origin of the URL the action targets, such as
   `https://develocity.example.com` (see the audience paragraph below). An administrator
   configures it; see
   [Workload Identity](https://develocity.ai/help/workload-identity) in the Develocity
   documentation.
2. **The `id-token: write` permission on the job.** Without it GitHub offers the job no OIDC
   token, and the action warns that workload identity failed and skips all cache activity.
   Declaring `permissions` replaces the defaults, so also list the ones the job already
   relies on, such as `contents: read` for `actions/checkout`.
3. **`develocity-edge-url`, not `develocity-url`.** Workload identity currently works only
   when the action targets an Edge directly. Through `develocity-url` the Artifact Cache CLI
   discovers its Edge first, and discovery rejects a workload identity token. Develocity
   2026.1 and later has a built-in Edge, so the Develocity server's own URL works as
   `develocity-edge-url`, as in the example below.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write # lets the action mint a GitHub OIDC token
    steps:
      - uses: actions/checkout@v7

      - uses: gradle/develocity-artifact-cache-github-action@v1
        with:
          develocity-edge-url: https://develocity.example.com
          # no develocity-access-key: the action falls back to workload identity

      - run: ./gradlew build
```

The token's audience defaults to the origin of the chosen endpoint, `develocity-edge-url`
here (`https://develocity.example.com` above, with no path or trailing slash). A non-default
port stays in it: `https://develocity.example.com:8443/` gives the audience
`https://develocity.example.com:8443`. It must match the audience on the workload identity
rule exactly. If the rule uses a different one, for
instance because the Edge has its own host while the rule names the server's, set
`develocity-oidc-audience`:

```yaml
        with:
          develocity-edge-url: https://develocity.example.com
          develocity-oidc-audience: https://github.com/acme
```

A GitHub OIDC token lasts about five minutes, so the action mints it and exchanges it for a
Develocity token straight away, in its main step. `develocity-token-expiry` governs how long
*that* token lives. Size it from the action's step through to the post-step store, not from
the start of the job.

The action's token is not shared with other steps. If a later step needs Develocity too (for
example `setup-gradle`, to publish Build Scans), it needs its own credential. Workflows that
mint an OIDC token for such a step can pass the same token to this action's
`develocity-access-key`, in `host=token` form.

### Short-lived access tokens

Develocity access keys are long-lived, so exposing one to a whole workflow is a risk if it
leaks. To avoid that, the action immediately exchanges its credential (your access key, or
the OIDC token it minted) for a **short-lived Develocity token**. It hands the token to the
Artifact Cache CLI through a private, subprocess-scoped environment variable for cache restore
and store, and, when the AC CLI is downloaded via the edge route (see
[Downloading the Artifact Cache CLI](#downloading-the-artifact-cache-cli)), sends it as the
bearer credential on that download request. The token is used only for these action-internal
purposes — the credential it was exchanged for never leaves this action. If the exchange
fails, the action warns and skips rather than falling back to that credential.

The action does **not** set `DEVELOCITY_ACCESS_KEY` (or the token) for any other step. A cache
action shouldn't propagate credentials to the rest of the build, and — unlike Gradle, which
has `setup-gradle` — other build tools have no equivalent that would expect it. If a later step
needs to authenticate with Develocity (for example, to publish a Build Scan), give it its own
credentials: `setup-gradle`, for instance, performs the same key-to-short-lived-token exchange
and exports it for Gradle builds.

The token is valid for **2 hours** by default; raise `develocity-token-expiry` only if a build
runs long enough that the token could expire before the post-step store runs.

## Security

Whichever credential the action authenticates with, the access key you supply or the OIDC
token it mints, is exchanged for a short-lived Develocity token, and only that token reaches
the Artifact Cache CLI, through the host-scoped `DEVELOCITY_ACCESS_KEY` environment variable.
The credential itself is never passed to the CLI, and the action does not fall back to it
when the exchange fails. To limit its exposure:

- on the access-key path, supply the key through a repository or organization **secret**, as
  above, rather than a literal, and prefer an access key **scoped to only the servers and
  permissions** it needs;
- on the [workload identity path](#workload-identity) there
  is no key to store at all: the action mints a short-lived GitHub OIDC token, and the
  matching workload identity rule on the Develocity server decides what it grants.

Even without `develocity-auth: workload-identity`, the action uses workload identity whenever
`develocity-access-key` is unset or blank and the job grants `id-token: write`. It mints a
token whose audience is the Develocity endpoint, so the token is of no use anywhere else, and
it lasts about five minutes. An audience set with `develocity-oidc-audience` is used instead; if it is one that
other services also accept (GitHub's default, the repository owner URL, for example), the
token is valid there too. The token only authenticates if a workload identity rule on that
server matches the repository's claims. A misspelled secret name in such a repository
therefore authenticates with the rule's permissions rather than failing; the log line naming
the credential in use shows which one it was.

The action sends a credential only over `https` unless you opt in. An `http` endpoint makes
the action warn and skip all cache activity before it resolves or mints anything; set
`develocity-url-allow-insecure: true` only for a trusted internal network without TLS. For an
endpoint with a self-signed or internal certificate, prefer installing the certificate on the
runner. `allow-untrusted: true` turns verification off for the endpoint instead, for the token
exchange, the CLI download from its Edge node, and the CLI. Both inputs match the Jenkins
plugin's `develocityUrlAllowInsecure` and `allowUntrusted`.

Both the supplied credential and the exchanged token are registered with the runner as masked
values, so they are redacted from the workflow log.

## Downloading the Artifact Cache CLI

The action downloads the Artifact Cache CLI JAR once, in its main step, and the post step
reuses it. It takes the JAR from the first of these sources that applies:

1. **`cli-repository`**, when set, sending `cli-repository-header` if you configured one.
   It names the directory holding the JARs, and the action requests
   `<cli-repository>/develocity-artifact-cache-cli-<version>.jar`, the same layout as the
   Jenkins plugin's `cliRepository`, so one mirror serves both.
2. **The edge route**, when the endpoint you set (`develocity-url` or `develocity-edge-url`)
   is served by a Develocity Edge node at version **2.3.0 or later**, connected to Develocity
   2026.3 or later. The action recognizes such a node from the `X-Edge` header on the
   short-lived-token exchange response, then downloads
   `<endpoint>/edge/assets/artifact-cache-cli/<version>` with that token as the bearer
   credential. You host nothing, and an Edge node that already holds the version serves it
   without the runner reaching the public host. An Edge node older than 2.3.0 is logged and
   not used.
3. **The public Develocity download host.**

Checksum verification works the same whichever source provides the JAR: it is checked against
`cli-sha256`, or else the checksum the action ships for that version. A version with neither
runs unverified, and the action logs a warning.

The action does not fall back from one source to another: if the chosen source is still
unreachable after retries, it warns and skips all cache activity for the job. To take the
Edge node out of the download path, set `cli-repository`.

Before version 1.1.0, the action requested
`<cli-repository>/develocity-artifact-cache-cli/develocity-artifact-cache-cli-<version>.jar`,
one directory further down. A mirror laid out that way still works for now: when the current
layout answers 404, the action retries the old one and warns with the `cli-repository` value
to set instead. Update the value, because the old layout will stop working in a future
release.

## Cache coverage and observability

The action caches **Gradle, Maven, npm, pip, and Sonar** automatically: it invokes the Artifact Cache
CLI, which autodetects the matching project content on the runner and caches each type it
finds, with no per-type enable input. To exclude a type (for example, to skip npm in a
large monorepo), pass `--no-autodetect <type>` through `additional-cli-args`.

Cache activity is visible in two places:

- The **job summary** is the closest observability layer: a run's cache activity is
  summarized as a table (see [Job summary](#job-summary) below).
- For finer-grained cache operation metrics, inspect the build's **Build Scan** in
  Develocity. Gradle and Maven builds that apply the Develocity Gradle plugin or Maven
  extension surface cache metrics there directly.

### Job summary

After a run, the action adds a single cache-activity table to the GitHub **job summary**,
sourced from the metrics the Artifact Cache CLI writes. The table has a **Phase** column
and one row per phase:

- **Restore**: Restored, Not found, Failed, Size, Duration.
- **Store**: Stored, Already present, Failed, Size, Duration.

A successful restore followed by a store shows both rows; when the store is skipped —
read-only cache, a restore that did not succeed, or a **failed build** — the table shows
the restore row only. An `Errors` column is added when a phase reports a message, and a
dash marks a field the CLI did not report. The post step always runs, so the table
appears even when the build fails (with the restore row only, since a failed build never
stores). The summary carries cache counts only, not a Build Scan link (Gradle and Maven
surface the same metrics in the Build Scan through the separate path above).

### Warnings

Cache activity is best-effort: a restore or store problem is surfaced as a **warning**,
never a build failure. Only two things fail the step: a JAR checksum mismatch, which is a
security guard (see [Security](#security)), and an unknown `develocity-auth` value. The
action warns when an operation reports a problem:

- **Failures**: a non-zero failed-artifacts or failed-images count, or a transient error
  recorded by the CLI, warns that some cache activity did not complete. The build
  continues, just less cached.
- **Rejected access key**: when the Develocity server rejects the key (unauthorized), the
  action warns specifically and points at the `develocity-access-key` input; cache
  activity is skipped for that step and the build still runs.
- **Workload identity problems**: when the action uses workload identity, it warns when the
  job has no OIDC token to mint (check `permissions: id-token: write`), when Develocity does
  not accept the token (naming the audience to compare with the workload identity rule), when
  a restore or store is rejected (check what the rule grants), and when `develocity-url` is
  set instead of `develocity-edge-url`. Cache activity is skipped where it cannot proceed,
  and the build still runs.

A cache **miss** is expected: it is not a failure and never warns.

## Cache image names

Each cache is keyed by an **image name**. By default you set none: the Artifact Cache CLI
auto-generates one from the workflow, job, git ref, runner operating system, and runner
architecture, so every job gets its own cache and jobs on different operating systems or
architectures stay isolated.

### Fallback to another branch's cache

On **restore** the action tries the primary name and then a fallback, so a branch with no
cache of its own can still restore from a related branch:

- **Pull requests** fall back to the **base branch**'s cache, so a pull request's first
  build restores from its target branch.
- **Push and manually dispatched events** fall back to the repository **default branch**'s
  cache, so the first build of a new branch restores from your default branch instead of
  starting cold.

**Store** always writes to the single primary name.

### Overriding with `image-names`

Set `image-names` (one name per line) to take control of naming. It is a single
ordered list:

- the **first** entry is the **primary**, the name the cache is **stored** under and the
  first that **restore** tries;
- the **rest** are **restore-only fallbacks**, tried in order, stopping at the first hit.

Setting `image-names` **replaces** the auto-generated names entirely, including the
pull request and push fallbacks above, so list any fallbacks you want as additional
entries yourself:

```yaml
- uses: gradle/develocity-artifact-cache-github-action@v1
  with:
    develocity-url: https://develocity.example.com
    image-names: |
      my-project-${{ github.ref_name }}
      my-project-main
```

### Matrix builds share a cache across axes other than operating system and architecture

The auto-generated cache name isolates by job, runner operating system, and runner
architecture, but **not** by other matrix axes such as a Java or build-tool version. Jobs
that differ only in such an axis therefore **share one cache** and can overwrite each
other's content.

To **isolate** a sensitive axis, add a dedicated `image-names` entry that includes its
value. Since this replaces the auto-generated names, add your own fallback entry to keep
the cross-branch seeding described above:

```yaml
    image-names: |
      my-project-java${{ matrix.java-version }}-${{ github.ref_name }}
      my-project-java${{ matrix.java-version }}-main
```

## Compatibility

The action runs on the **Node 24** action runtime and invokes the Artifact Cache CLI under a JDK that
meets a **minimum floor** (currently **Java 21**). On GitHub-hosted runners
it resolves that JDK from the pre-set `JAVA_HOME_<major>_<arch>` variables, so **no
`setup-java` step is required for the action itself**, and a build pinned to an older JDK still caches as long
as the runner provides a floor-satisfying JDK. If none is found, the action skips caching
for the job rather than failing the build.

### Supported runners

The action is supported on the following runners, provided each has a floor-satisfying
JDK (GitHub-hosted runners do; see above):

| Runner operating system | Architecture |
| --- | --- |
| Linux | x64 |
| macOS | x64, ARM64 |
| Windows | x64 |

It also runs on self-hosted Linux (x64) runners. All five cache types (Gradle, Maven,
npm, pip, Sonar) are supported on every listed runner.

### JVM warmup

The Artifact Cache CLI runs on the JVM, so each `restore` (main step) and `store` (post step) pays a
**one-time JVM startup cost**, typically a few seconds per invocation, independent of
cache size.

## Versioning

The action pins a **default Artifact Cache CLI version** (`cli-version` defaults to that
pinned general-availability version), and that default advances with each Artifact Cache
CLI release. Two consequences for consumers:

- The action tracks a **tested default version**, so most users need not set
  `cli-version` at all.
- **A change to the minimum required JDK is a breaking change and forces a major version
  bump of the action.** Pinning a newer CLI whose floor your runners do not meet makes the
  action skip caching (it never fails the build), so a new major version signals a
  JDK-floor change you must accommodate.

To pin or override, set `cli-version` (and `cli-sha256` to verify a version the
action does not already ship a checksum for).

## Configuration handled by the CLI

Some configuration is handled by the CLI rather than exposed as an action input:

- **Cache cleanup** is automatic, so there is no cleanup input.
- **Cache-type selection** is by autodetection: the CLI detects Gradle, Maven, npm, pip,
  and Sonar. To exclude a type, pass `--no-autodetect <type>` through
  `additional-cli-args`.
- **Tool homes** are resolved from the standard environment variables CI sets. For a
  non-standard Maven local repository, pass `--maven-repository <path>` through
  `additional-cli-args`.

## Migrating from a custom Artifact Cache CLI setup

If you invoke the Artifact Cache CLI yourself today, typically a custom "setup + restore" step plus a
separate "store" step, configured through `DV_SERVER` / `ARTIFACT_CACHE_*` environment
variables, this action replaces all of it with **one step**.

The migration has the same shape wherever the CLI is used:

1. **Collapse two steps into one.** Replace the restore step, and the separate store step,
   with a single `uses:` of this action, placed where restore ran. Its post step stores at
   the end of the job automatically.
2. **Map the CLI configuration to inputs** using the table below.
3. **Drop the plumbing.** Remove the manual JAR download, checksum verification, and JDK
   wiring; the action does all of it.
4. **Move any restore-time condition.** If the old restore step had an `if:` gate for
   whether to use the cache, put the same condition on the action step. A
   store-only-on-success gate (`if: success()` on the old store step) needs nothing — the
   action always stores only when the job succeeds.
5. **Matrixed jobs:** if your matrix varies beyond the operating system and architecture,
   add a dedicated `image-names` entry to isolate that axis so those jobs do not share
   one cache (see
   [Matrix builds](#matrix-builds-share-a-cache-across-axes-other-than-operating-system-and-architecture)).

A custom setup may also do things this action does not cover, for example uploading the
CLI log as an artifact, or supplying Develocity credentials that other steps (such as
Build Scan publishing) also rely on. This action replaces only the cache restore and
store; **keep, or add, whatever additional steps your workflow needs for behavior outside
that scope.**

| Current (custom CLI setup) | This action |
| --- | --- |
| separate **restore** step **and** **store** step | one `uses:` step (main restores, post stores) |
| `DV_SERVER` | `develocity-url` input |
| `DEVELOCITY_ACCESS_KEY` (env) | pass it to the `develocity-access-key` input (the action does not read the env var) |
| `--image-name` / `ARTIFACT_CACHE_IMAGE` | `image-names` input |
| extra CLI flags (for example `--gradle-home`, often via `ARTIFACT_CACHE_OPTS`) | `additional-cli-args` input (one per line) |
| CLI version pin (for example `ARTIFACT_CACHE_CLI_VERSION`) | `cli-version`, usually omitted (defaults to the pinned version) |
| manual JAR download, SHA-256 verify, and JDK wiring | built in |

Two behavior differences to note:

- **Store timing.** The post step stores only when the **job succeeds**, the same effect
  as an `if: success()` gate on the old store step (best-effort, never failing the build).
  Set `cache-read-only: true` to never store.
- **Logs.** The action does not upload an `artifact-cache.log` artifact; cache activity
  appears in the **job summary** and workflow log. Drop any log-upload step, or add your
  own if you still want the artifact.

**Before**, two custom steps plus environment:

```yaml
env:
  DV_SERVER: https://develocity.example.com
  DEVELOCITY_ACCESS_KEY: ${{ secrets.DEVELOCITY_ACCESS_KEY }}
  ARTIFACT_CACHE_OPTS: "--gradle-home=/home/runner/.gradle"
steps:
  - uses: ./.github/actions/setup-and-restore-artifact-cache # download CLI + restore
  - run: ./gradlew build
  - uses: ./.github/actions/store-artifact-cache             # store
    if: success()
```

**After**, one step:

```yaml
steps:
  - uses: gradle/develocity-artifact-cache-github-action@v1
    with:
      develocity-url: https://develocity.example.com
      develocity-access-key: ${{ secrets.DEVELOCITY_ACCESS_KEY }}
      additional-cli-args: --gradle-home=/home/runner/.gradle
  - run: ./gradlew build
```

## Learn more

- [Develocity Artifact Cache documentation](https://docs.develocity.ai/artifact-cache/):
  product overview, CLI reference, and the GitHub Actions setup guide.
