# Finalizing the accept() established-race fix

**This file is working state, not upstream material.** It sits in its own commit
on top of the fix so it can be dropped before the patch is sent: take the fix
commit alone with `git format-patch`, and leave this one behind.

There are now TWO fix commits and ONE unresolved defect:

1. `tcp: don't destroy a connection whose handshake beat accept()` - validated
   on hardware, 120/120. Ready to upstream.
2. `tcp: accept() must not return a socket that is still handshaking` -
   correct, but NOT sufficient on its own. See "The unresolved defect" below.
3. TLS handshakes still fail roughly half the time, cause unknown.

What follows is what is still owed before any of this goes to the wolfIP
maintainers, and the context needed to finish it on another machine.

## What is already established

The defect is confirmed on hardware, not inferred. A TC4Dx running the FreeRTOS
socket port answered **8 of 16** rapid TCP connections in a perfectly
alternating pattern, and 100% of them once spaced a second apart. Three
independent pieces of evidence agree on the cause:

- **Packet capture.** On every failing connection the board completed the
  handshake and *acknowledged the client's data*, then sent nothing further -
  no echo, no FIN, no RST. The RST that eventually appears is the client giving
  up. Failing streams carried exactly one SYN-ACK; succeeding ones carried two.
- **Application trace.** The server printed `accepted` and then `peer closed`
  for connections whose peer had not closed and was still blocked reading. That
  is `wolfIP_sock_recv()` returning 0 on a live connection.
- **Targeted instrumentation.** A `printf` in each `accept()` failure branch
  showed the `ESTABLISHED && is_listener` branch being taken for the large
  majority of connections, and the other two branches never.

After the fix, on the same board: 16/16 rapid, 4/4 across two connections with
two round trips each, 12/12 across a gap sweep from 0 to 3000 ms, and 120/120
over three payload sizes from 10 to 460 bytes, with ping at 0% loss throughout.

## The second defect: accept() returned an unconnected socket

`wolfIP_sock_accept()` returns as soon as it has cloned the connection. The
clone is in `SYN_RCVD`, before the peer's final ACK, and the core knows it - it
clears `CB_EVENT_WRITABLE` at the clone site with the comment *"Don't signal
writable until connection fully established"*. The FreeRTOS BSD wrapper then
handed that descriptor straight to the application.

A plaintext echo server tolerates this: by the time it calls `recv()` the ACK
has usually arrived. A TLS server does not, because `wolfSSL_accept()` goes
straight for the ClientHello. The symptom was `recv()` returning 0 on a
connection the peer had not closed, surfacing as wolfSSL error -308.

Fixed by `wolfIP_sock_is_connected()` plus a bounded poll in `accept()`.
Measured on a TC4Dx, accept() never once returned an unestablished socket over
several dozen TLS connections and never hit its timeout - so the fix does what
it says. It did NOT change the TLS failure rate.

## The unresolved defect

TLS 1.3 handshakes against this stack fail roughly 50 to 70 percent of the
time, in no stable pattern. Everything below is measured, not inferred.

**Server side**, per failing connection: `accept()` returns an established
socket, the application calls `recv()`, no data ever arrives, and the socket
reports closed. The ClientHello never reaches the accepted socket.

**Client side**: `wolfSSL_connect error -311, unknown type in record hdr`. The
client reads bytes that are not a valid TLS record header. So the server does
put something on the wire, and it is malformed or misframed.

**The lead worth following.** In the socket-callback trace the LISTENING
descriptor is seen going READABLE and then CLOSED:

    [sock_cb] ifd=256 events=0x0001 ...   <- listener READABLE
    [sock_cb] ifd=256 events=0x0010 ...   <- listener CLOSED
    [sock_cb] ifd=257 events=0x0010 ...   <- the accepted socket, closed

`ifd=256` is the listener; `257` is the accepted clone. A listening socket has
no business becoming readable or closed. That is consistent with inbound
segments for an established connection being matched to the listener instead of
the clone, which would explain both halves at once: the clone starves while the
client receives something it cannot frame. `tcp_input`'s socket-matching loop
is where to look - it matches on local port first, then discriminates by state
and 4-tuple, and the listener's tuple is wildcarded back to ANY when it reverts.

**Also unexplained.** After a few dozen connections the board began refusing
new ones outright while still answering ping: the listener stops accepting.
`MAX_TCPSOCKETS` was 4 in that configuration, and this stack has a
`TCP_TIME_WAIT` state but no 2MSL timer that returns those sockets to the pool.
Sustained connection churn exhausting a small socket table would look exactly
like this. Worth confirming, because if true it is a separate defect from the
matching problem above and would bite any long-running server.

**Ruled out**, each checked directly rather than assumed:

- Not the driver or the link: ping stayed at 0 percent loss throughout, and the
  MAC, PHY and DMA rings are exercised continuously by the same traffic.
- Not stale socket buffers on reuse: `tcp_new_socket()` re-initialises both the
  TX fifo and the RX queue.
- Not the TLS layer: every connection that does get a usable socket completes a
  genuine TLS 1.3 handshake, TLS_AES_128_GCM_SHA256 over SECP256R1.
- Not accept() returning unconnected sockets: fixed, verified, rate unchanged.

## Still to do

### 1. Confirming run on the stripped build (required)

The 120/120 figures above were measured with the temporary `printf` still
present in `accept()`. That print *slows* accept(), which makes the race more
likely rather than less, so the instrumented run was the harder test - but the
stripped build is what ships and it has only been compiled, not run. Flash the
clean build and re-run the burst before sending the patch.

A portable reproducer, adjust the address and port:

```
for i in $(seq 1 16); do
  printf 'probe%d-0123456789' "$i" | timeout 3 nc -q1 192.168.1.11 7 \
    && printf . || printf T
done; echo
```

Back-to-back with no sleep is the point. Before the fix this prints an
alternating `.T.T.T.T...`; after it, sixteen dots. Inserting `sleep 1` between
iterations makes it pass either way, which is exactly why hand testing never
caught this.

### 2. Regression test (required before upstreaming)

Not written. `libcheck` was unavailable on the machine where the fix was made,
so a test could be neither compiled nor run, and shipping an unverified test to
maintainers is worse than shipping none.

It belongs in `src/test/unit/unit_tests_tcp_state.c`. The sequence to drive:

1. socket / bind / listen on a port
2. inject a SYN, poll - the listener goes to `TCP_SYN_RCVD`
3. call `wolfIP_sock_accept()`, poll to flush - this is what sends the SYN-ACK
4. inject the peer's final ACK **and** a data segment, poll
5. close the accepted socket without reading, to return the port to LISTEN
6. repeat 2-4, but this time deliver the final ACK *before* calling accept()
7. assert `accept()` returns a valid descriptor, that the socket is
   `TCP_ESTABLISHED`, and that `wolfIP_sock_recv()` returns the payload sent in
   step 6 rather than 0

Step 7 is the regression: before the fix, accept() returns -1 and the payload is
lost.

Two traps that cost time and are not obvious from the harness code:

- **`accept()` only queues the SYN-ACK.** It is transmitted on the next
  `wolfIP_poll()`. Reading `last_frame_sent` straight after `accept()` yields
  the previous frame, or zeroes.
- **Nothing transmits until ARP is resolved.** The mock link's `poll` returns no
  frames, so the ARP reply has to be injected by hand -
  `unit_tests_tcp_state.c` already does this around line 2790; copy that block.

An attempt to build a standalone harness outside the `check` framework, by
stubbing `check.h` and including `unit_shared.c`, compiled and ran but never
transmitted a frame even with ARP primed. Whatever else the mock link needs was
not identified. Worth a few minutes before writing the test proper, because a
harness that cannot transmit cannot exercise this path at all.

### 3. Find the remaining defect

This is the blocking item and it will not be solved by inspection - four
successive hypotheses about it were contradicted by the next hardware run. It
needs a host harness that can actually drive a full handshake, which is also
what item 2 needs. Instrument `tcp_input`'s socket-matching loop to log which
socket each inbound segment is matched to, and check whether data for an
established connection is ever delivered to the listener.

### 4. Regression-check the existing suites

Neither was run on the machine where the fix was made:

- `make unit` - needs `libcheck` and `pkg-config`
- the POSIX/TAP tests (`build/tcpecho` and friends) - need a TAP device and
  wolfSSL

The fix changes a path every TCP server reaches, so both should be green before
the patch goes out.

## Review points for the maintainers

Flag these explicitly rather than letting them be discovered in review:

- **Buffer copy cost.** The clone takes `memcpy` of both `RXBUF_SIZE` and
  `TXBUF_SIZE`. Copying the memory, rather than re-initialising the queues, is
  what preserves payload the peer already sent and this stack already
  acknowledged; re-homing the `data` pointers keeps `head`, `tail` and
  `seq_base` exact. It runs only on this path, but on a configuration with large
  socket buffers it is not free, and a maintainer may prefer to move only the
  occupied bytes.
- **Exhaustion fallback.** When no socket is free the old behaviour is kept -
  revert the listener, return -1 - so a port cannot be pinned in `ESTABLISHED`.
  The connection is lost either way in that case. If wolfIP would rather keep
  the connection and have the caller retry, that is a one-line change, but it
  risks pinning the port when the pool never drains.
- **Initial events.** The clone is marked `CB_EVENT_WRITABLE`, plus
  `CB_EVENT_READABLE` when data is already queued. Confirm that matches what the
  socket layer expects from a freshly accepted, already-established socket.

## Related issue, found but not fixed

`wolfIP_sock_listen()` does `(void)backlog;` - the backlog argument is ignored
entirely, so a caller asking for 1 gets the same behaviour as one asking for
128. Unrelated to this bug and deliberately left alone, but worth raising
separately.

## Provenance

Found while bringing up a wolfIP TCP echo server on an Infineon TC4D7 Lite
(TC4Dx, FreeRTOS, `src/port/freeRTOS/bsd_socket.c`). The submodule was at
`d70271b` and otherwise unmodified.
