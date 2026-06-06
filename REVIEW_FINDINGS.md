# Static Review Findings

Date: 2026-06-06

## Scope

This was a read-only static review. The lab was not deployed, router commands
were not executed, and runtime behavior has not yet been verified.

Static checks completed:

- `nat-lab.clab.yml` parses as valid YAML.
- `nat-lab.clab.yml.annotations.json` parses as valid JSON.
- The topology contains 14 nodes and 13 links.
- Every node is referenced by at least one link.
- No duplicate link endpoints were found.

## Overall Assessment

The topology, addressing plan, and introductory NAT scenarios are organized
well. However, several advanced scenarios do not currently match their
documentation and are likely to fail or demonstrate different behavior than
intended.

## Findings

### 1. Twice NAT Is Incomplete

Severity: High

Relevant files:

- `configs/br2.partial.cfg`
- `configs/isp.partial.cfg`
- `configs/hq.cfg`
- `README.md`, Scenario 6

Problems:

- ISP has a route to BR2's translated source range, `10.10.10.0/24`, but no
  route to HQ's real `192.168.10.0/24` through `198.51.100.2`.
- HQ has no route back to `10.10.10.0/24`.
- HQ's dynamic NAT ACL permits all traffic sourced from `192.168.10.0/24`.
  Therefore, HQ reply traffic toward `10.10.10.10` may be translated by the
  HQ dynamic NAT pool instead of being returned unchanged to BR2.
- The README describes some required routes as manual additional setup even
  though the lab is advertised as preconfigured.
- The outside-local/outside-global terminology in the BR2 comments appears
  reversed.

Runtime validation:

1. Add or confirm the missing ISP and HQ routes.
2. From `br2-pc`, ping `10.20.20.10`.
3. Inspect `show ip nat translations` and `debug ip nat` on BR2.
4. Inspect `show ip nat translations` on HQ to determine whether HQ translates
   the reply.
5. Capture traffic on `hq-pc1` and confirm the received source is
   `10.10.10.10`.

Likely correction:

- Add the required routes to the startup configs.
- Replace HQ's standard dynamic NAT ACL with an extended ACL or route-map that
  exempts traffic destined for `10.10.10.0/24`.

### 2. Hairpin NAT Is Not Explicitly Implemented

Severity: High

Relevant files:

- `configs/hq.cfg`, Scenario 7 comments
- `README.md`, Scenario 7

The lab claims that two `ip nat inside` interfaces and the existing static NAT
entry automatically provide direct inside-to-inside hairpinning. The supplied
config does not contain an explicit NAT Stick or another clear hairpin
configuration.

The destination `198.51.100.10` is routed by HQ's default route toward ISP.
Traffic may potentially leave HQ, return from ISP, and then use the static NAT
entry. If so, that is outside-to-inside reflection through ISP, not the
documented direct `Gi3` to `Gi4` hairpin flow.

Runtime validation:

1. Run `traceroute 198.51.100.10` from `hq-pc1`.
2. Capture on HQ `Gi2`, `Gi3`, and `Gi4`.
3. Check whether the packet leaves HQ toward ISP.
4. Check `show ip nat translations` and `debug ip nat`.
5. Decide whether to implement NAT Stick, another supported hairpin method, or
   document the ISP-reflection behavior accurately.

Reference:

- Cisco IOS-XE NAT Stick documentation:
  <https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_nat-on-stick.html>

### 3. Scenario 4 Verification Targets the Wrong Device

Severity: High

Relevant file:

- `README.md`, Scenario 4 verification

The walkthrough curls `http://198.51.100.1`, which is the ISP router, not the
external HTTP server. The subsequent capture on `ext-srv` will not observe
that traffic and may wait indefinitely.

Likely correction:

- Use `http://203.0.113.100` as the destination.
- Start the `tcpdump` capture before generating traffic.
- Ensure the test does not overlap with Scenario 5 unless that is intentional.

### 4. Endpoint Web-Server Commands Conflict With the Base Image

Severity: High

Relevant file:

- `nat-lab.clab.yml`

The `ghcr.io/hellt/network-multitool` image already provides web services on
ports 80 and 443. The topology then starts Python HTTP servers on the same
ports, which are likely to fail with an address-in-use error.

Additionally, Python's `http.server` on port 443 is plain HTTP without TLS.
The README tests it with `https://`, so that test is likely to fail even if the
Python process successfully binds.

The topology also uses the mutable `latest` tag, reducing reproducibility.

Runtime validation:

1. Inspect `docker logs` for each Linux endpoint.
2. Run `ss -lntp` inside `ext-srv` and `dmz-srv`.
3. Test both `curl http://172.16.0.100` and `curl -k https://172.16.0.100`.
4. Either rely on the base image's existing services or use a dedicated,
   pinned server image with known HTTP/TLS behavior.

### 5. Policy NAT Does Not Produce a Different Source Address

Severity: Medium

Relevant files:

- `configs/br1.partial.cfg`
- `README.md`, Scenario 5

General PAT and policy NAT both translate through interface `Ethernet0/1`,
whose address is `100.64.0.2`. The comments claim policy traffic uses a
different source IP and a separate NAT pool, but no separate pool exists.

The route-map still demonstrates selective rule matching, but it does not
demonstrate meaningfully different translation behavior.

Runtime validation:

1. Generate partner and non-partner traffic.
2. Compare `show ip nat translations` and `show route-map POLICY-NAT`.
3. If different source addresses are intended, add a dedicated translated
   address or pool and the required ISP route.

### 6. Static NAT and Static PAT May Need `extendable`

Severity: Medium

Relevant file:

- `configs/hq.cfg`

The DMZ server `172.16.0.100` has:

- A one-to-one mapping to `198.51.100.10`.
- TCP port mappings to `198.51.100.2`.

These are ambiguous static translations involving the same inside-local host.
Depending on IOS-XE behavior, one or more commands may be rejected unless the
appropriate static entries use `extendable`.

Runtime validation:

1. Inspect the boot/configuration logs for rejected commands.
2. Run `show running-config | include ip nat inside source static`.
3. Confirm all three mappings exist.
4. Test static NAT and both static PAT ports independently.

Reference:

- Cisco IOS-XE NAT guide:
  <https://www.cisco.com/c/en/us/td/docs/routers/ios/config/17-x/ip-addressing/b-ip-addressing/m_iadnat-addr-consv-xe.html>

### 7. README Cleanup Is Needed

Severity: Low

Relevant file:

- `README.md`

Problems:

- A stray Markdown code fence exists near the end of the file.
- The final generated-summary section does not belong in the finished README.
- Some claims describe required manual setup despite advertising
  preconfigured startup configs.
- Some expected NAT output and terminology should be checked against actual
  IOS-XE output after deployment.

## Runtime Validation Order

Use this order on the machine that can deploy the lab:

1. Deploy and inspect startup/configuration logs for rejected commands.
2. Confirm Linux endpoint interfaces, routes, listening ports, HTTP, and TLS.
3. Confirm all router interfaces and static routes.
4. Test Scenario 1: static NAT.
5. Test Scenario 2: static PAT.
6. Test Scenario 3: dynamic NAT pool.
7. Correct and test Scenario 4 verification.
8. Test whether Scenario 5 produces distinct behavior.
9. Diagnose and correct Scenario 6 twice NAT.
10. Trace and capture Scenario 7 to determine the actual hairpin path.
11. Update configs and README to match verified behavior.

## Suggested Initial Commands

```bash
sudo containerlab deploy -t nat-lab.clab.yml
sudo containerlab inspect -t nat-lab.clab.yml

sudo docker logs clab-nat-lab-hq
sudo docker logs clab-nat-lab-br2
sudo docker logs clab-nat-lab-dmz-srv
sudo docker logs clab-nat-lab-ext-srv

sudo docker exec clab-nat-lab-dmz-srv ss -lntp
sudo docker exec clab-nat-lab-ext-srv ss -lntp
```

On each Cisco router, collect:

```text
show ip interface brief
show ip route
show running-config | include ip nat
show ip nat translations
show ip nat statistics
show access-lists
show route-map
```

## Notes

- No source files were changed during the static review.
- This document records hypotheses from static analysis. Runtime results should
  be treated as authoritative and used to update these findings.
