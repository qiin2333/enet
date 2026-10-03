# Optional outbound admission

`ENetHost` has optional `sendAdmissionContext`, `sendAdmission` and
`sendCompletion` fields. Hosts initialize them to zero. Install both callbacks
to share an external sending budget with other transports; leaving either
callback null preserves ordinary outbound sending.

Admission runs on the host service thread before ACK or command serialization.
It receives a peer and maximum datagram payload reservation, including room
for an optional checksum. A nonpositive result defers that peer without
dequeueing ACKs or outgoing commands. Reliable timeout processing continues,
so a denied retransmission remains queued until admission succeeds.

A denied peer contributes a bounded 2 ms retry wait in `enet_host_service`,
including when its ping is already due. This does not fabricate a send timestamp
or delay an explicit owner flush or retry triggered by incoming socket activity.

Every admitted datagram has exactly one completion. `attempted=1` reports the
final serialized payload bytes and socket result, including socket failure.
`attempted=0` reports an unused reservation, with zero bytes and result, when
no datagram was submitted. The caller owns reservation release and transport
header accounting; these callbacks do not provide IP-level QoE measurements.

Callbacks must finish immediately, must not reenter ENet, and must not throw.
Install or clear callbacks only from the service owner while it is not servicing
the host. This adds fields to the public host structure, so rebuild ENet and
all users together rather than mixing existing binaries with the new headers.

The opt-in regression target uses actual IPv4 loopback peers and verifies
denied ACK/command preservation, timeout/retransmission, fragmented final-byte
accounting and null-callback legacy delivery:

```sh
cmake -S . -B build -DENET_BUILD_TESTS=ON
cmake --build build
ctest --test-dir build --output-on-failure
```

Install a CMake GTest package or pass `-DGTEST_SOURCE_DIR=/path/to/googletest`.
Tests do not establish application pacing, fairness, partial OS submissions,
notification costs or performance thresholds.
