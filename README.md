# beacon-clock

Can a device tell the time if it has no clock?

Every CPU has a clock, but it is a crystal oscillator: a metronome that ticks millions
of times a second and knows nothing about the date. Knowing *what time it is* takes a
separate battery-backed real-time clock (RTC), and most Raspberry Pis do not have one.
Without a network, a Pi boots believing it is whenever it last shut down.

This project makes that explicit. The Pi runs with NTP switched off and no RTC, and
learns the time only from a broadcast, the way every computer really keeps time:

> current time = a time someone told you + ticks counted since then

**Status: work in progress.** No results yet. Measurements will be recorded in
`RESULTS.md` with hardware, network (Wi-Fi or Ethernet), commands and raw logs.

## Parts

- **`beacon`** runs on a machine with good time (a laptop synced by NTP). It broadcasts
  the current time over UDP every second, and answers timing requests.
- **`clock`** runs on the Raspberry Pi. It listens for the beacon, works out the network
  delay and its own crystal drift, keeps its own clock, and displays the time. It runs
  as a systemd service.

## The two problems it solves

**1. Network delay.** A one-way broadcast arrives late, and the receiver cannot tell by
how much. A single round trip fixes that. With four timestamps (t1 request sent, t2
received by beacon, t3 reply sent, t4 reply received), as in NTP:

```
offset = ((t2 - t1) + (t3 - t4)) / 2
delay  =  (t4 - t1) - (t3 - t2)
```

This assumes the path is equally slow in both directions; the error is at most half
the asymmetry.

**2. Drift.** Crystals are typically off by tens of parts per million, which is seconds
per day. The clock measures its own drift against the beacon and corrects for it, so it
keeps time during "holdover" when the beacon disappears.

## Why it matters on real devices

A field device with the wrong time cannot validate TLS certificates and stamps its
telemetry with the wrong time. Keeping time on devices without an RTC is an everyday
embedded Linux problem.

## Design

- Local time is measured with the monotonic clock (Go's `time.Time` carries a
  monotonic reading), so it never jumps when the system clock is set.
- UDP broadcast on the LAN; one goroutine receives, one computes, one displays;
  shutdown via `context` on SIGTERM.
- Setting the system clock needs only `CAP_SYS_TIME`, granted through systemd
  (`AmbientCapabilities=CAP_SYS_TIME`), not root.
- Static binary cross-compiled for the Pi (`GOOS=linux GOARCH=arm64` or `arm`,
  `CGO_ENABLED=0`).

## Roadmap

- [ ] M1 Beacon broadcasts time over UDP; Pi listens and prints beacon time vs its own
- [ ] M2 Round-trip requests; NTP-style offset and delay; keep a software clock
- [ ] M3 Drift estimate in ppm; holdover test (stop the beacon for an hour, measure error)
- [ ] M4 systemd service with `CAP_SYS_TIME`; set the system clock; Wi-Fi vs Ethernet comparison
- [ ] M5 Optional hardware: GPS module over UART (NMEA + PPS) as a beacon; I2C OLED display
