# Code Review — codraw/aws-tool-kit

## Fixes applied (2026-07-20)

- **composer.json:** PHP version constraint changed from unbounded `>=8.5` to `^8.5` (version-compatibility debt: prevents a future PHP 9 from installing against this package; no effect on any currently existing PHP version).
- **H1** — `composer.json`: added `"psr/log": "^3"` to `require` (used by `EventListener/NewestInstanceRoleCheckListener.php`); moved `"symfony/http-client-contracts": "^3.4"` from `require-dev` to `require` (hard type-hinted by both IMDS clients); added a `suggest` entry for `symfony/http-client` as the implementation needed by the IMDS clients.
- **M1** — `composer.json`: added `"php": ">=8.5"` to `require`, matching sibling packages.
- **M2** — `Command/CloudWatchLogsDownloadCommand.php`: `fopen()` result is now checked; a `false` return throws a `\RuntimeException` naming the file and mode instead of crashing later with a `TypeError` in `fwrite()`.
- **M3** — `Command/CloudWatchLogsDownloadCommand.php`: both `strtotime()` results are now validated; unparseable `--startTime`/`--endTime` values throw an `\InvalidArgumentException` instead of silently becoming the epoch. (The suggested `startTime < endTime` cross-check was not added — it would reject inputs currently accepted.)
- **L1** — `Command/CloudWatchLogsDownloadCommand.php`: the download loop is wrapped in `try/finally` so the file handle is always closed if `getLogEvents()` throws mid-loop. (Partial-file-on-failure behavior is unchanged.)

Not fixed (deliberately): M4, M5, L2–L6 (behavioral/design changes with consumer-facing risk), and no declared dependencies were removed.

### Validation pass (2026-07-20)

- `composer install` resolves cleanly with the updated `composer.json` constraints (no adjustment needed).
- Full test suite passes (25 tests, 167 assertions, exit code 0) with the fixes applied — no test-expectation updates were required. The 15 PHPUnit notices ("mock object without expectations") are pre-existing PHPUnit 12 style notices in untouched test files.
- PHPStan reports one pre-existing error unrelated to the applied fixes (`DependencyInjection/AwsToolKitIntegration.php:78`, `NodeDefinition::children()` not found — a type-flow limitation of chaining `->validate()…->end()->children()` on the config builder). Present with and without the fixes (verified via `git stash`); left as is.
- `markdownlint-cli2` is clean for all package markdown files (its `--fix` pass only normalized this file).

## Overall Assessment

This is a small, focused utility package (5 production classes) providing three AWS-related features: a CloudWatch Logs download command, EC2 IMDS (instance metadata) clients, and a "newest instance in role" console-command gate for cron deduplication in auto-scaling pools. The code is clean, modern PHP with good unit test coverage of every production class and a well-designed DI integration with config validation. The main problems are packaging-level: undeclared runtime dependencies (`psr/log`, `symfony/http-client-contracts`) and a missing `php` version constraint, plus several error-handling gaps in the CloudWatch command (unchecked `fopen()`, unvalidated `strtotime()`), and a couple of edge-case robustness issues (IMDSv2 token never refreshed, EC2 `describeInstances` pagination ignored). No exploitable security flaws were found; the tools are operator-facing CLI utilities.

Overall grade: **B** — good with real but mostly medium-severity issues.

## Findings

### High

#### **[FIXED]** H1. Undeclared runtime dependencies: `psr/log` and `symfony/http-client-contracts`

`composer.json:10-23`

Production code references packages that are not in `require`:

- `EventListener/NewestInstanceRoleCheckListener.php:7-9` imports `Psr\Log\LoggerInterface`, `LogLevel`, and `NullLogger`, and instantiates `NullLogger` in the constructor (line 41). `psr/log` appears nowhere in `composer.json` — not even in `require-dev`. It only works today because some transitive dependency happens to pull it in. Symfony Console 6.4 does not hard-require `psr/log`, so a consumer installing only this package's declared dependencies can get a fatal "class not found" when the listener is instantiated.
- `Imds/ImdsClientV1.php:5` and `Imds/ImdsClientV2.php:5` type-hint `Symfony\Contracts\HttpClient\HttpClientInterface`, but `symfony/http-client-contracts` is only in `require-dev` (line 22). Any consumer enabling the documented `imds_version` feature needs it at runtime. Even if you consider the IMDS clients "optional" (a consumer must supply an HTTP client implementation anyway), the contracts package should be a hard `require` (it is tiny) or at minimum listed under `suggest` — currently the `suggest` section only mentions `aws/aws-sdk-php-symfony`.

### Medium

#### **[FIXED]** M1. Missing `php` version constraint in `composer.json`

`composer.json:10-14`

The `require` block declares no `php` constraint at all. The code uses PHP 8.1+ features (`final public const` in `NewestInstanceRoleCheckListener.php:23`, first-class `match` in `AwsToolKitIntegration.php:72`, constructor property promotion throughout). Sibling packages in the same monorepo (e.g. `codraw/dependency-injection`) declare `"php": ">=8.5"`. Without the constraint, Composer will happily install this package on an incompatible PHP version and fail at runtime with parse/compile errors.

#### **[FIXED]** M2. `fopen()` result unchecked in CloudWatchLogsDownloadCommand

`Command/CloudWatchLogsDownloadCommand.php:50`

```php
$handle = fopen($input->getArgument('output'), $input->getOption('fileMode'));
```

If the output path is unwritable, its directory does not exist, or the `fileMode` string is invalid, `fopen()` emits a warning and returns `false`. The subsequent `fwrite($handle, ...)` at line 64 then throws a `TypeError` (PHP 8 rejects `false` where a resource is expected). A mistyped output path is entirely normal CLI use; it should produce a clear error message and a non-zero exit code instead of a crash. Note also that a read-only `fileMode` such as `r` passes `fopen()` but makes every `fwrite()` silently fail, producing an empty download that returns exit code 0.

#### **[FIXED]** M3. `strtotime()` results unvalidated — invalid dates silently become the epoch

`Command/CloudWatchLogsDownloadCommand.php:39-40`

```php
$startTime = strtotime($input->getOption('startTime')) * 1000;
$endTime = strtotime($input->getOption('endTime')) * 1000;
```

`strtotime()` returns `false` for unparseable input, and `false * 1000 === 0`. A typo like `--startTime="-1 dya"` silently means "since 1970-01-01", potentially downloading an enormous log range; a bad `--endTime` silently means "up to 1970", producing an empty file with a success exit code. There is also no check that `startTime < endTime`. Both option values should be validated and rejected with an error.

#### M4. IMDSv2 token cached forever, never refreshed after its TTL

`Imds/ImdsClientV2.php:9, 28-44`

The client requests a token with `X-aws-ec2-metadata-token-ttl-seconds: 3600` and memoizes it in `$this->token` with no expiry tracking. In a long-running process (Messenger worker, daemon, or any process alive more than an hour) every `getCurrentInstanceId()` call after token expiry will fail with HTTP 401 for the remaining lifetime of the service instance, with no recovery path. The client should record the fetch time and re-request the token when it is near/expired (or on a 401).

#### M5. `describeInstances` pagination not handled

`EventListener/NewestInstanceRoleCheckListener.php:109-128`

`getNewestInstanceIdForRole()` reads `$result['Reservations']` from a single `describeInstances` call and never follows `NextToken`. The EC2 API paginates results (and callers can be throttled into smaller pages). In an account where the filtered set spans multiple pages, the newest instance can be absent from the first page, causing the wrong instance — or no instance — to consider itself "newest". For the typical small-pool use case this won't trigger, but it is a correctness bug in the exact scenario (large auto-scaling groups) the feature targets. Using the SDK's `getPaginator('DescribeInstances', ...)` fixes it cheaply.

### Low

#### **[FIXED]** L1. File handle leaked and partial output on exception

`Command/CloudWatchLogsDownloadCommand.php:50-70`

If `getLogEvents()` throws mid-loop (throttling, expired credentials, wrong group/stream name), the exception propagates and `fclose($handle)` at line 70 is never reached. The process exits anyway so the OS reclaims the handle, but a partially written file is left behind with no indication it is incomplete. A `try/finally` (and possibly writing to a temp file then renaming) would make this robust.

#### L2. Event subscription keyed by event class relies on FrameworkBundle alias mapping

`EventListener/NewestInstanceRoleCheckListener.php:27-34`

`getSubscribedEvents()` returns `ConsoleCommandEvent::class` as the event name, but Symfony Console dispatches this event under the name `ConsoleEvents::COMMAND` (`'console.command'`). The class-name key only works when the dispatcher was built with FrameworkBundle's `event_dispatcher.event_aliases` mapping (via `RegisterListenersPass`). Wiring this component's listener into a plain `EventDispatcher` used with a standalone Console `Application` silently never fires the check — a dangerous silent failure for a component-level (not bundle-level) package. Using `ConsoleEvents::COMMAND` as the key would work in both contexts.

#### L3. `ImdsClientInterface::getCurrentInstanceId(): ?string` contract never returns null

`Imds/ImdsClientInterface.php:7`, `Imds/ImdsClientV1.php:13-19`, `Imds/ImdsClientV2.php:15-26`

Both implementations either return a non-empty string or throw (Symfony HttpClient's `getContent()` throws on non-2xx and transport errors). The nullable return forces callers such as `NewestInstanceRoleCheckListener::checkNewestInstance()` (line 72) to handle a `null`/empty case that cannot occur, while the real failure mode (exception) is what actually needs handling. Tightening the contract to `string` (documented as throwing) would be clearer.

#### L4. Fail-closed on transient AWS errors silently skips the job on every instance

`EventListener/NewestInstanceRoleCheckListener.php:64-88`

Any `Throwable` from IMDS or the EC2 API disables the command (log-only, level error). Because every instance in the pool performs the same check, a transient EC2 API outage or throttling event means the cron is skipped fleet-wide for that tick. This is a defensible fail-closed design, but worth documenting: jobs that must not be missed need external monitoring, since the only signal is a log line and exit code 113.

#### L5. No timeout configured on IMDS HTTP requests

`Imds/ImdsClientV1.php:15-18`, `Imds/ImdsClientV2.php:17-40`

The clients rely on the injected HttpClient's default timeout (typically 60s from `default_socket_timeout`). On a non-EC2 host (developer machine, container in another cloud) where 169.254.169.254 blackholes, a command run with `--aws-newest-instance-role` hangs for the full default timeout before the listener's catch disables it. The AWS SDKs use ~1s timeouts for IMDS for this reason; passing `timeout`/`max_duration` options (or documenting that the injected client should be configured with a short timeout) would help.

#### L6. Inherent check-then-run race in the newest-instance strategy

`EventListener/NewestInstanceRoleCheckListener.php:107-140`

The tie-breaking (`ksort` by launch time, then `sort` on instance IDs, line 134-137) is nicely deterministic when all instances see the same snapshot. But instances run the check at slightly different moments; if a new instance launches (or the current newest terminates) between two hosts' checks, either two hosts or zero hosts can conclude they are "newest" for that tick. This is an accepted limitation of the pattern the README describes, but it deserves a documentation caveat for jobs that must run exactly once.

## Strengths

- **Small, single-purpose classes** with clear separation: command, listener, IMDS clients, DI integration. Easy to audit end to end.
- **Modern, consistent PHP**: constructor property promotion, `match`, readonly-style immutable collaborators, typed properties throughout; the phpstan baseline is empty (`phpstan-baseline.neon`).
- **Every production class has a dedicated unit test**, including both branches of the compiler pass and per-`imds_version` DI integration cases with service ids and alias assertions (`Tests/DependencyInjection/AwsToolKitIntegrationTest.php`).
- **Deterministic newest-instance selection**: grouping by launch timestamp and sorting instance IDs (`NewestInstanceRoleCheckListener.php:123-139`) guarantees all instances that see the same EC2 snapshot elect the same winner, including launch-time ties.
- **Sensible fail-closed behavior with structured logging** in the listener — on any doubt (unreachable IMDS, empty instance id, EC2 error) the command is disabled rather than risking duplicate execution.
- **DI configuration validation** (`AwsToolKitIntegration.php:68-85`) prevents an invalid wiring state: enabling `newest_instance_role_check` without choosing an `imds_version` is rejected at config-compile time, and the unused IMDS client definition is removed from the container.
- **Compiler pass degrades gracefully** (`AddNewestInstanceRoleCommandOptionPass.php:16-20`): the `--aws-newest-instance-role` option is only added to console commands when the listener service actually exists.

## Test Coverage

Coverage is qualitatively good for a package this size:

- **Well covered**: `NewestInstanceRoleCheckListener` (8 tests: no option, null role, IMDS failure, empty instance id, no instances, not-newest, EC2 error, newest — `Tests/EventListener/NewestInstanceRoleListenerCheckTest.php`); `CloudWatchLogsDownloadCommand` happy paths including pagination via `nextForwardToken` and append `fileMode`, plus the missing-client guard; both IMDS clients (request shape and headers asserted); the compiler pass (both presence branches); the DI integration (service registration, aliasing, definition removal per config).
- **Gaps**: no test for `fopen()` failure or a read-only `fileMode` (M2); no test for invalid `--startTime`/`--endTime` strings (M3); `ImdsClientV2` token memoization across multiple `getCurrentInstanceId()` calls is untested (only a single call is exercised), so M4's caching behavior has no coverage; the empty-string role branch (`NewestInstanceRoleCheckListener.php:54-58`) is untested; no test covers the `imds_version: null` + listener-enabled validation error message in `AwsToolKitIntegration::addConfiguration()`.
- Tests are pure unit tests with mocks; there is no integration test against real HTTP/AWS behavior (acceptable for this kind of package, but it means contract drift with the AWS SDK response shapes would go unnoticed).
