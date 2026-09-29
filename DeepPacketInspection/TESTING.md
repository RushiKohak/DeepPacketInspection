# DPI Engine — Testing Report

## 1. Test Environment

- Platform: Windows 11
- Compiler environment: MSYS2 UCRT64
- Language: C++17
- Test analyzer: Wireshark / TShark 4.6.9
- Input capture: `test_dpi.pcap`
- Input packets: 77
- Input bytes: 5,738
- Link type: Ethernet
- Snaplen: 65,535 bytes

## 2. Baseline Functional Test

### Command

```bash
./dpi_engine.exe test_dpi.pcap output.pcap
```

### Result

- Total packets: 77
- Total bytes: 5,738
- TCP packets: 73
- UDP packets: 4
- Forwarded: 77
- Dropped: 0
- Output file: `output.pcap`

### Status

**PASS**

The generated `output.pcap` was subsequently opened/read successfully with TShark.

---

## 3. Application Blocking Test

### Test Case

Block YouTube application traffic.

### Command

```bash
./dpi_engine.exe test_dpi.pcap blocked_output.pcap --block-app YouTube
```

### Engine Result

- Total packets: 77
- Forwarded: 76
- Dropped: 1
- YouTube was detected from the traffic/SNI.

### Independent Verification

TShark was used to search the generated `blocked_output.pcap` for:

```text
tls.handshake.extensions_server_name == www.youtube.com
```

No matching packets were returned.

### Status

**PASS**

The DPI engine reported one dropped packet and independent inspection confirmed that the YouTube TLS SNI packet was absent from the output capture.

---

## 4. Domain Blocking Test

### Test Case

Block the `youtube.com` domain.

### Command

```bash
./dpi_engine.exe test_dpi.pcap domain_blocked.pcap --block-domain youtube.com
```

### Engine Result

- Total packets: 77
- Forwarded: 76
- Dropped: 1
- `www.youtube.com` was detected.

### Independent Verification

TShark was used to search the generated `domain_blocked.pcap` for:

```text
tls.handshake.extensions_server_name == www.youtube.com
```

No matching packets were returned.

### Status

**PASS**

The domain-blocking rule resulted in one dropped packet, and the blocked YouTube SNI was absent from the resulting PCAP.

---

## 5. IP Blocking Test

### Test Case

Attempt to block DNS traffic to `8.8.8.8`.

### Command

```bash
./dpi_engine.exe test_dpi.pcap ip_blocked.pcap --block-ip 8.8.8.8
```

### Input Verification

TShark inspection of `test_dpi.pcap` showed four DNS queries directed to `8.8.8.8`:

- `www.google.com`
- `www.youtube.com`
- `www.facebook.com`
- `api.twitter.com`

### Engine Result

- Total packets: 77
- Forwarded: 77
- Dropped: 0

### Independent Verification

TShark inspection of `ip_blocked.pcap` still found all four DNS packets addressed to `8.8.8.8`.

### Status

**NOT PASSED / REQUIRES INVESTIGATION**

The rule was accepted by the CLI, but no packets were dropped in this test case. The existing core implementation was not modified during this testing phase.

---

## 6. PCAP Analysis / Classification Test

The test capture contains multiple traffic types and application indicators, including:

- HTTPS/TLS traffic
- HTTP traffic
- DNS traffic
- YouTube
- Facebook
- Instagram
- Twitter/X
- Amazon
- GitHub
- Discord
- Zoom
- Telegram
- TikTok
- Spotify
- Cloudflare
- Apple
- Google

The baseline engine reported:

- HTTPS: 39 packets
- Unknown: 16 packets
- DNS: 4 packets
- Twitter/X: 3 packets
- HTTP: 2 packets
- Several other identified applications with 1 packet each

### Status

**PASS**

The engine successfully classified multiple traffic/application types from the supplied PCAP.

---

## 7. Multithreaded Processing Test

The engine was executed using:

- 2 Load Balancers
- 2 Fast Paths per Load Balancer
- 4 Fast Paths total

Example baseline distribution:

```text
LB0 dispatched: 53
LB1 dispatched: 24

FP0 processed: 53
FP1 processed: 0
FP2 processed: 0
FP3 processed: 24
```

### Status

**PASS — Architecture Demonstrated**

The engine successfully executed its configured Load Balancer/Fast Path processing architecture.

The observed distribution is specific to the supplied test capture and should not be presented as a general load-balancing performance result.

---

## 8. Performance Benchmark

### Command

```bash
time ./dpi_engine.exe test_dpi.pcap benchmark_output.pcap
```

### Measured Result

```text
real    0m0.643s
user    0m0.015s
sys     0m0.015s
```

For the 77-packet, 5,738-byte test capture:

- Wall-clock execution time: **0.643 seconds**
- Approximate observed processing rate: **119.8 packets/sec**
- Approximate byte rate based on the test capture: **8.9 KB/sec**

### Important Benchmark Note

This is a benchmark of the supplied 77-packet test capture. It is **not a maximum-throughput measurement** and should not be presented as production network throughput. The capture is small, so startup, thread creation, synchronization, and file I/O contribute significantly to the measured wall-clock time.

### Status

**PASS — Benchmark Recorded**

---

## 9. Overall Test Matrix

| Test | Result |
|---|---|
| PCAP input | PASS |
| Packet parsing | PASS |
| TCP/UDP identification | PASS |
| SNI extraction | PASS |
| Application classification | PASS |
| Domain detection | PASS |
| Normal packet forwarding | PASS |
| Output PCAP generation | PASS |
| Application blocking | PASS |
| Domain blocking | PASS |
| IP blocking (`8.8.8.8`) | NOT PASSED / REQUIRES INVESTIGATION |
| Multithreaded architecture execution | PASS |
| Performance benchmark | PASS |

---

## 10. Known Limitation

During testing, the IP-blocking option accepted `8.8.8.8` as a configured blocked IP, but the four DNS packets to that address were still present in the generated output PCAP.

This behavior was recorded as a test result rather than changing the existing core implementation.

---

## 11. Testing Tools

### DPI Engine

The project executable was compiled and executed using the MSYS2 UCRT64 environment.

### Wireshark / TShark

TShark 4.6.9 was used for independent PCAP inspection and verification of blocked traffic.

This independent verification is important because it checks the generated PCAP separately from the DPI engine's own console statistics.

---

## 12. Conclusion

The DPI engine successfully demonstrated:

1. PCAP-based packet processing.
2. TCP/UDP traffic identification.
3. TLS/SNI-based application detection.
4. HTTP and DNS traffic identification.
5. Application classification.
6. Domain detection.
7. Application-level blocking.
8. Domain-level blocking.
9. Multithreaded Load Balancer/Fast Path execution.
10. Generation of processed output PCAP files.
11. Independent verification using Wireshark/TShark.
12. Basic performance benchmarking.

The IP-blocking behavior requires further investigation if that feature is required for the final implementation.
