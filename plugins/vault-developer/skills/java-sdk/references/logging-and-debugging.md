# Logging and Debugging Vault Java SDK Code

How to write log lines from SDK code and how to read them back out of Vault. None of this is
checked at deploy time; it is the observability side of the platform, and it is the fastest way
to confirm a class does what the developer thinks it does before it meets real volume.

## LogService

`LogService` is in `com.veeva.vault.sdk.api.core`; import it explicitly. Confirm the package from the developer's local Vault Java SDK `.jar` if it is present.
`ServiceLocator.locate(LogService.class)` sends messages to the Debug Log (and, for `info`/`warn`/
`error`, also the Runtime Log). Prefer the message-template overload over string concatenation:
templates are only constructed when the corresponding log level is enabled, where concatenation
always runs and has to be manually wrapped in an `isXEnabled()` check to get the same benefit.

```java
import com.veeva.vault.sdk.api.core.LogService;

private static final LogService LOG = ServiceLocator.locate(LogService.class);

LOG.debug("Skipping record {}: {} already set", recordId, fieldName);
```

The `messageTemplate` string takes `{}` placeholders; the number of `{}` instances must match the
number of trailing arguments, filled in order. `debug()` only reaches the Debug Log; `info()`/
`warn()`/`error()` reach both the Debug Log and the Runtime Log. A `Supplier<String>` overload also
exists for cases where building the message string itself is expensive.

`System.out` and `System.err` are rejected at deploy validation, so `LogService` is the only
logging path there is. The forbidden-constructs table in `platform-constraints.md` has the row.

## Reading the SDK debug log

**Admin > Logs > Developer Logs** in the Vault UI. The debug log isn't on by default: a Vault
Owner or System Admin must start a debug log session for the specific user who will reproduce the
issue (max 20 concurrent user sessions per Vault), then the developer reproduces the trigger's
firing condition (e.g. saves the record) and downloads the resulting log.

Log line types worth knowing: `DEBUG`/`INFO`/`WARN`/`ERROR` are your own `LogService` calls;
`SYSDATA`/`SYSINFO`/`SYSWRN` are Vault-platform messages; `SYSERR` covers platform exceptions and
any `RollbackException` thrown by custom code; `PERF` reports CPU/elapsed/memory usage. Each
trigger execution is bracketed by `Start Execution`/`End Execution` markers, the clearest way to
spot a nested trigger firing as a side effect of another one. Service-call lines report
`count`/`elapsed`/`avg`, which is the fastest way to confirm a loop is making one batched call
instead of N per-record calls. The Vault Toolbox IntelliJ plugin can create debug log sessions and
download/analyze the resulting log without leaving the IDE, as an alternative to the Admin UI.

The `count` figure on a service-call line is the one to check first when reviewing bulk safety:
run the class against a multi-record input and a `count` that matches the record count means the
service is being called once per item. See the bulk-safety section of `platform-constraints.md`
for the pattern that brings it down to one.

The same counts show where a class stopped. When it fails with no error in the log, and the
`Start Execution`/`End Execution` lines and `PERF` look clean, list the service calls the code has
to make and compare them with the counts. A step that comes up short threw there, often inside a
lambda whose exception the log does not print. Check that before calling it a platform problem.

Keep diagnostics beside the workspace, not inside it; `${CLAUDE_PLUGIN_ROOT}/reference/vault-cli.md`
says why.
