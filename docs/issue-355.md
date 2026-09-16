Investigation and validation for issue #355

The reproduced regression is a session teardown/reuse race. In v6.0.0,
`sweepExpiredGridSessions` starts `killProviderAppiumSession` in a goroutine
and immediately calls `ReleaseFromAutomation`. A new request can claim the
same device while the previous driver's DELETE is still running. The provider
reverse proxy did not serialize these transitions. Its plugin removal callback
also cleared session state without checking which session ended.

`git diff v5.7.0 v6.0.0 -- hub/router/appiumgrid.go` traces the new asynchronous
DELETE to commit `2250658` ("phase 6 appium grid improv"). v5.7.0 released expired
claims but did not launch this competing background DELETE. The fix preserves
v6's actual session cleanup and waits for it before releasing the claim.

Before behavior changes, `TestExpiredSessionWaitsForCleanup` failed against the
original implementation: while the fake provider blocked A's DELETE, availability
was already true and the hub session ID was already empty. This is a deterministic
reproduction of the bad lifecycle handoff, not a reproduction on physical iOS.

A separate partial-creation path was also unsafe: the plugin awaited provider
registration after Appium had allocated a driver; a failed notification returned
an error without deleting that allocated session. Notifications had no network
timeout. Late unexpected-shutdown callbacks used a global current session ID
rather than the ending plugin instance's ID.

The stale state demonstrated here is the previous session's outstanding DELETE
versus an already-released hub claim, a provider session snapshot that an older
callback can erase, and an allocated Appium session after registration failure.
No physical-device evidence establishes a particular leaked native process,
listener, or WDA session as the reporter's exact residual resource. Longer iOS
teardown plausibly widens the race window; that explanation needs hardware
validation. Restarting the provider terminates its device contexts/processes and
reprovisions Appium/WDA, removing the overlapping session state, which explains
why it can temporarily help without making a provider restart the solution.

The implementation:

- Retains the device claim until session DELETE completes successfully (or reports
  an absent session). Cleanup failure keeps it reserved. Cleanup runs outside the
  device lock, and duplicate janitor passes cannot launch concurrent cleanup.
- Releases successful explicit DELETEs coherently and makes delayed error cleanup
  conditional on the original session ID. Device selection excludes cleanup in
  progress even if another path changes its availability flag.
- Serializes provider POST-session and DELETE-session operations per device.
  Client cancellation does not abort the upstream transition. If creation finishes
  after its caller disconnects, a deferred cleanup deletes that exact new session,
  including when response forwarding unwinds through a panic.
- Rolls back successful Appium allocation when plugin registration fails. Uses
  Appium's normal `next()` chain for explicit deletion; normal driver partial-
  creation and unexpected-shutdown resource teardown remain Appium's responsibility.
- Gives plugin callbacks a bounded HTTP timeout and an optional `session_id` in
  the existing removal request body. Older empty-body notifications still work;
  full protection against late callbacks requires the updated plugin as well as
  the updated provider. No public route or configuration was changed.
- Protects provider Appium session fields with a mutex and publishes/clears the
  ID, presence flag, and capabilities atomically.
- Adds DEBUG diagnostics for UDID, old/new session, cleanup and reservation,
  cancellation, Appium PID/port/start/exit, and failed-creation Appium/WDA HTTP
  health and WDA/MJPEG ports. No capability or credential bodies are added to logs.

Appium's umbrella driver already owns normal master-session cleanup; a direct
`driver.deleteSession` call was not established as bypassing that bookkeeping.
The plugin now uses the command chain for consistency, not as a claimed root
cause. Reference inspected: https://github.com/appium/appium/blob/master/packages/appium/lib/appium.ts

Changed production files:

- `hub/devices/hub_device.go`, `hub/router/appiumgrid.go`: cleanup reservation and ordering.
- `provider/devices/runtime.go`, `provider/devices/platform.go`: device transition lock and atomic session state.
- `provider/devices/appium.go`: process diagnostics.
- `provider/router/routes.go`, `provider/router/appium_lifecycle.go`: serialized proxy transitions and disconnected-client cleanup.
- `provider/router/appium_plugin.go`: session-aware removal notifications.
- `appium-plugin/gads-appium-plugin.js`, `appium-plugin/src/api/client.js`: rollback, callback identity, and timeout.

Regression coverage:

- `hub/router/session_cleanup_test.go`: blocked DELETE retains A's claim, B can
  claim after cleanup, and unsuccessful cleanup retains the reservation.
- `hub/router/appiumgrid_test.go`: existing expiry/deletion tests now require
  cleanup completion rather than immediate background release.
- `provider/router/appium_lifecycle_test.go`: sequential sessions, failed-test
  cleanup, driver partial-creation error, disconnected creation, DELETE surviving
  cancellation, unrelated device lock independence, idempotent session-aware
  removal, and concurrent session-state updates/snapshots.
- `appium-plugin/test/lifecycle.test.js`: five tests covering normal deletion,
  rollback after allocation, driver creation errors, late shutdown of A after B
  starts, and notification failure after successful deletion. External imports
  are faked; the shipped plugin methods themselves are executed.

Validation used temporary Go 1.26.0 and Node 22.14.0 runtimes in `/tmp`, because
neither runtime was initially on PATH. Go commands used GOCACHE=/tmp/gads-gocache
and GOMODCACHE=/tmp/gads-gomod. HTTP-listener tests required sandbox escalation.

- Focused lifecycle Go tests: passed; the ordering regression failed before the fix.
- `node appium-plugin/test/lifecycle.test.js`: 5 passed.
- `go test ./hub/router ./hub/devices ./provider/...`: passed.
- `go test -race ./hub/router ./hub/devices ./provider/router ./provider/devices`: passed.
- `go vet ./hub/router ./hub/devices ./provider/router ./provider/devices`: passed.
- `gofmt` on changed Go files and `git diff --check`: passed.
- `go test ./hub/... ./provider/...` and `go test ./...`: attempted, but the
  checkout lacks generated `GADS/docs`, and the auth integration tests wait for
  MongoDB. Those runs were interrupted after the blockage was identified.
- `go test -timeout 30s ./...`: confirmed the missing generated docs and a timeout
  in `hub/auth.TestAdminSecretKeyHistoryHandler` creating MongoDB indexes. Affected
  packages passed in this run. No generated docs or unrelated test changes added.

Remaining validation: run the updated provider, hub, and locally installed plugin
with a real iOS device and capture DEBUG logs across normal DELETE, killed test
client, failed app launch, and WDA loss, while another device has a healthy active
session. Verify the next session and observe the Appium/WDA listeners. A native
WDA runner or Appium process that is itself dead/hung, or an unacknowledged cleanup
beyond the bounded timeout, is not proven recoverable by these unit tests. This
patch does not reset such devices or restart the provider. A client that disappears
after creation is detected through the existing idle timeout; explicitly disabling
`newCommandTimeout` retains its existing semantics.

Suggested commit: `fix: serialize Appium session cleanup before device reuse (#355)`
