# Troubleshooting Log: Site-to-Site IPsec VPN (Packet Tracer)

This log documents the problems I hit while building the HQ ↔ Branch IPsec tunnel, how I diagnosed each one, and how I fixed it. I followed a structured approach: check what the device actually has configured, compare both ends of the tunnel, and verify the physical layer.

**Final result:** tunnel established (`QM_IDLE`, ACTIVE), traffic encrypted and decrypted (`#pkts encaps: 9`, `#pkts decaps: 8`).

---

## Issue 1: `securityk9` license would not activate

**Symptom**
- Running `license boot module c2900 technology-package securityk9` produced no license agreement and no `ACCEPT? [yes/no]` prompt.
- After `write` and `reload`, `show version` still showed:
  ```
  security      disable       None          None
  ```
- `show run | include license` showed no `license boot` line, so the command was never saved.
- Same behaviour on every router.

**Diagnosis**
- Used `?` at each level (`license boot module c2900 technology-package ?`) to confirm the syntax and that `securityk9` was a valid option. The syntax was correct, so the problem was not a typo.
- Since every router behaved the same, it was not a single faulty device but something about the environment.

**Resolution**
- <!-- TODO: write what fixed it, e.g. "rebuilt the topology in a new .pkt file" or "updated Packet Tracer" -->
- Verified before reloading that `Next reboot` showed `securityk9`, then saved and reloaded.
- After reload:
  ```
  security      securityk9    Evaluation    securityk9
  ```
- `crypto ?` now listed `isakmp`, `ipsec` and `map`.

**Note:** the ISP router does not need the security license. It only forwards packets between the two public (10.x) networks and never runs crypto commands.

---

## Issue 2: Packets from HQ-PC stopped at the ISP router

**Symptom**
- `ping 192.168.20.10` from HQ-PC failed; Simulation mode showed the packet being dropped at the ISP.

**Diagnosis**
- `show ip route` on the ISP showed only the two connected 10.x networks, with no route to 192.168.x.x. **This is correct by design**: a real ISP doesn't know about private LANs.
- So the real question was why the packet arrived at the ISP **unencrypted** (as plain ICMP to 192.168.20.10) instead of as ESP between 10.1.1.1 and 10.2.2.1.
- On HQ-R, `show crypto map`, `show access-lists 110` and `show run | section crypto` all returned **nothing**: the crypto configuration had never been applied (the pasted block did not go through).

**Resolution**
- Re-entered the configuration in small chunks (Phase 1, then Phase 2 + ACL, then crypto map + interface), confirming each part.
- `show crypto map` then showed the peer, ACL 110, transform set `VPN-SET`, and `Interfaces using crypto map VPN-MAP: GigabitEthernet0/1`.

**Lesson:** always verify that configuration actually landed on the device. Never assume a paste worked.

---

## Issue 3: Phase 1 failing with `MM_NO_STATE`, caused by a non-mirrored crypto ACL

**Symptom**
```
dst             src             state          conn-id slot status
10.2.2.1        10.1.1.1        MM_NO_STATE          0    0 ACTIVE (deleted)
```
- `#pkts encaps: 0`, `#pkts decaps: 0`.
- HQ-R was recognising the "interesting traffic" and trying to contact the peer, but negotiation failed.

**Diagnosis**
- Compared both routers side by side:
  - `show crypto isakmp policy`: identical (AES 256, SHA, pre-shared key, DH group 5, 86400 s)
  - `show run | include isakmp key`: same key, correct peer addresses
  - `show crypto map`: on BR-R, ACL 110 was wrong:
    ```
    access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.10.0 0.0.0.255
    ```
    The source should have been the Branch LAN (192.168.20.0). The two crypto ACLs must be **mirror images** of each other.

**Resolution (BR-R)**
```
no access-list 110
access-list 110 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
```

**Lesson:** a non-mirrored crypto ACL is one of the most common real-world site-to-site VPN mistakes.

---

## Issue 4: Still `MM_NO_STATE`, caused by the WAN cable on the wrong port

**Symptom**
- After fixing the ACL, Phase 1 still failed. All 10 pings from HQ-PC timed out.

**Diagnosis**
- Re-verified everything else: pre-shared key, transform sets (`esp-aes esp-sha-hmac` on both sides), policies and ACLs all matched.
- Packet Tracer does not support `clear crypto isakmp` / `clear crypto sa`, so I saved and reloaded both VPN routers to clear any stale state. It still failed.
- Checked the physical layer: **BR-R's Gi0/1 (the WAN interface, holding 10.2.2.1 and the crypto map) was connected to BR-SW instead of the ISP.** HQ-R's IKE requests could never reach BR-R. This was most likely introduced when BR-R was replaced during Issue 1 and the cables were reconnected.

**Resolution**
- Re-cabled BR-R: **Gi0/0 → BR-SW**, **Gi0/1 → ISP Gi0/1**.
- Verified with `show cdp neighbors` (ISP on Gi0/1, BR-SW on Gi0/0) and `show ip interface brief` (both up/up).

**Lesson:** start troubleshooting at Layer 1. Configuration can be perfect and still fail if a cable is in the wrong port. `show cdp neighbors` confirms what is actually connected where.

---

## Final verification

```
HQ-R#show crypto isakmp sa
dst             src             state          conn-id slot status
10.2.2.1        10.1.1.1        QM_IDLE           1084    0 ACTIVE
```

```
#pkts encaps: 9, #pkts encrypt: 9
#pkts decaps: 8, #pkts decrypt: 8
#send errors 1
```

- **QM_IDLE / ACTIVE**: Phase 1 (IKE) completed successfully.
- **encaps / decaps > 0**: traffic is being encrypted and decrypted through the tunnel.
- **Inbound and outbound ESP SAs ACTIVE** with `esp-aes esp-sha-hmac` in Tunnel mode: Phase 2 (IPsec) is established.
- **1 send error**: the first packet was dropped while the tunnel was being negotiated, which is expected behaviour.

## Key takeaways

1. Verify configuration on the device (`show` commands) instead of assuming it was applied.
2. Compare both ends of a VPN side by side: policy, key, transform set, peer address and mirrored ACLs must all match.
3. Check the physical layer early (`show cdp neighbors`, `show ip interface brief`).
4. Understand what "normal" looks like: the ISP having no route to private networks, and the first ping timing out during negotiation, are both expected.
