# Project 4: Reliable Transport Protocol

`4700send` reads data from STDIN and sends it over UDP to `4700recv` which
prints it to STDOUT in order, with no duplicates or missing data.

## High-level approach

Packets are JSON objects like in the starter code:

* Data: `{"type": "msg", "seq": n, "time": t, "data": "...", "check": c}`
* Ack:  `{"type": "ack", "seq": n, "expect": e, "time": t, "check": c}`

`seq` numbers packets (each carries up to 1375 characters),
`check` is a checksum, 
`expect` is the next packet the receiver still needs,
`time` is the sender's send time, which the receiver echoes back

**Receiver**
* Keeps packets that arrive early in a dictionary (`buffer`) and prints
  everything that is in order (Duplicates are ignored).
* Acks every data packet it receives, including duplicates, because a
  duplicate usually means an earlier ack was lost. The ack names the packet
  it received (`seq`) and the next packet it needs (`expect`) so any ack
  also confirms every packet before `expect`.

**Sender**
* Sliding window: keeps up to `window` unacknowledged packets in flight.
* RTT estimate from the echoed send time,
  `rtt = 0.875 * rtt + 0.125 * sample`. 
* Loss detection:
  * Timeout: no ack after 2 x RTT, resend
  * Fast retransmit: 3 newer packets were acked and this one has been
    outstanding for 1.5 x RTT, resend
* Congestion control (TCP): the window starts at 2 packets then grows by 1
  per ack in slow start (doubling every RTT) and by 1/window per ack after
  `ssthresh`. On a loss it is halved, at most once per RTT, so a burst of
  losses counts as one event.
* Exits once every packet has been acknowledged.

**Corrupted packets**
* Each packet carries a checksum: a sum of the packet's bytes, each multiplied
  by its position.
* A plain sum (like the Internet checksum from lecture) missed about 1 in 2300
  of the simulator's mangled packets in my tests. 

## Challenges

* Spurious retransmissions under jitter: With random delays, packets
  arrive out of order. My first fast retransmit rule (3 newer packets acked)
  fired for packets that were only late and each false alarm halved the
  window: 8-2 had 10 retransmissions and no real losses. Also requiring the
  packet to be 1.5 x RTT old fixed most of them.
* Small router buffers: Some tests buffer only about 3 packets, so a
  large initial burst gets dropped. Starting with a window of 2 and growing
  with each ack keeps sends spread out.

## Good properties

* Never prints wrong data: checksums, sequence numbers, and a receive buffer.
  (Malformed packets are dropped)
* Lost acks are harmless because every ack also carries `expect`.
* RTT is measured from echoed timestamps so samples from retransmitted
  packets are still correct.

## Testing

* `./test`: all 18 provided configs pass.
* Ran every provided config with seeds 1-5 (90 runs).
* Used `./run <config>` logs (retransmissions, drops, corrupt packets) to
  debug individual runs.