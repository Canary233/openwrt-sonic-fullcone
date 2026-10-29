# openwrt-sonic-fullcone

[简体中文](./README.md) | English

SONiC-style Full Cone NAT for OpenWrt, with per-zone and per-protocol granularity.

## What is this

A port of the [SONiC fullcone NAT kernel patch](https://github.com/sonic-net/sonic-linux-kernel/blob/master/patches-sonic/Support-for-fullcone-nat.patch) (Akhilesh Samineni / Broadcom) to OpenWrt, bundled with the firewall and LuCI integration needed for fine-grained control.

### How it works

The kernel patch keeps a second hash table (`nat_by_manip_src`) in the conntrack NAT extension (`struct nf_conn_nat`), keyed by the **post-translation 3-tuple** (protocol, source IP, source port).  The index is allocated lazily when the first fullcone rule is installed and is backfilled with the existing SNAT mappings of that netns; configurations without fullcone rules never allocate the table and `struct nf_conn` stays at 248 bytes / 4 cachelines:

- **SNAT direction**: 3-tuple uniqueness guarantees that the same (proto, src_ip, src_port) is never reused by a different connection — this gives Endpoint-Independent Mapping (EIM). **The guarantee only holds when the SNAT rule carries a port range**; the rules fw4 generates do, see [Why SNAT rules carry a port range](#why-snat-rules-carry-a-port-range).
- **DNAT direction**: reverse lookup — inbound packets find the original internal host through the hash table, providing Endpoint-Independent Filtering (EIF).
- **Full L4 protocol coverage**: TCP, UDP, ICMP, GRE, SCTP, DCCP, UDPlite.
- **`masq_src` / `masq_dest` keep working**: fullcone srcnat rules carry the same source/destination matches as regular masquerade rules.
- **Zero overhead for non-fullcone traffic**: the fullcone flag is per-rule and the index is allocated on demand; no second table is maintained when no fullcone rule exists.

### Supported kernel versions

6.6, 6.12, 6.18 (all three share the same patch — the relevant `nf_nat_core.c` layout is identical across them).

## Installation

### Prerequisites

- An OpenWrt source tree (master / 25.12 / 24.10)
- Kernel 6.6 / 6.12 / 6.18
- `git` and `curl` available on the build host

> 23.05 is no longer supported: its iptables 1.8.8 has no `extensions/libxt_NAT.c` (added in 1.8.9), and two hunks of the firewall4 patch no longer apply.

### One-shot install

From the root of your OpenWrt source tree:

```bash
# Update feeds first (required for the LuCI patch)
./scripts/feeds update -a
./scripts/feeds install -a

# Apply all patches
curl -sSL https://raw.githubusercontent.com/Canary233/openwrt-sonic-fullcone/master/add_sonic_fullcone.sh | bash

# Build
make menuconfig   # No extra options needed — fullcone is compiled into the existing nft_masq.ko / xt_MASQUERADE.ko
make -j$(nproc)
```

The script clones the repo, detects the kernel version, copies the patches into the right places, and cleans up temporary files when done.

> **Note**: besides copying patches, the script also adds `PKG_FIXUP:=autoreconf` to `package/libs/libnftnl/Makefile`.
> The libnftnl patch touches `src/Makefile.am`, which makes automake try to regenerate `Makefile.in` using the exact version that produced the tarball (`automake-1.17` for libnftnl 1.3.1). That version is usually absent from the build environment, so the build fails with `automake-1.17: command not found`.
> This edits the OpenWrt source tree itself, so a `git pull` of the tree reverts it — just **re-run this script** afterwards. If the build already failed once, also run `make package/libs/libnftnl/clean` before rebuilding.

### Uninstall

Remove the patch files and rebuild:

```bash
rm -f target/linux/generic/hack-*/984-add-sonic-fullcone-*.patch
rm -f target/linux/generic/hack-*/985-add-sonic-fullcone-*.patch
rm -f target/linux/generic/hack-*/986-add-sonic-fullcone-*.patch
rm -f package/network/utils/iptables/patches/901-sonic-fullcone.patch
rm -f package/network/config/firewall*/patches/001-sonic-fullcone.patch
rm -f feeds/luci/applications/luci-app-firewall/patches/001-add-fullcone-options.patch

make target/linux/clean
make package/network/utils/iptables/clean
make package/network/config/firewall/clean
make package/network/config/firewall4/clean
make package/feeds/luci/luci-app-firewall/clean
make -j$(nproc)
```

Note: the `libnftnl` and `nftables` patches (which add the `fullcone` keyword) can stay — they do not affect normal behavior.

## Configuration

### Configuration logic

`defaults.fullcone` is the **global master switch**:

- **Off** (default): all fullcone functionality is disabled, settings inside zones have no effect.
- **On**: the feature is available, but each zone still has to **opt in** with its own fullcone checkbox.

This mirrors how OpenWrt's `flow_offloading` works.

### Web UI (LuCI)

The LuCI patch wires the fullcone options directly into OpenWrt's native firewall configuration pages — no extra LuCI app needed.

**Global settings** (Network → Firewall → General Settings):
- "Fullcone NAT" checkbox — global master switch.

![Global settings](img/luci-global-settings.png)

**Zone editing** (Network → Firewall → Zones → Edit):
- **General tab**: "Fullcone NAT" checkbox (only shown when masquerading is enabled), visible both in the zone list and the edit dialog.
- **Advanced tab**: "Fullcone protocols" multi-select (TCP, UDP, UDP-Lite, SCTP, DCCP) — restrict by protocol.

![Zone editing](img/luci-zone-edit.png)

### UCI command line

```bash
# 1. Turn on the global master switch (required)
uci set firewall.@defaults[0].fullcone='1'

# 2. Enable fullcone on the wan zone
uci set firewall.@zone[1].fullcone='1'

# 3. Optional: limit to UDP only (recommended for gaming / P2P)
uci add_list firewall.@zone[1].fullcone_proto='udp'

# Apply
uci commit firewall
/etc/init.d/firewall restart

# Verify
nft list ruleset | grep fullcone
```

### UCI config example

**Basic**: fullcone on all zones, all protocols

```
config defaults
    option fullcone '1'           # global master switch

config zone
    option name 'wan'
    option masq '1'
    option fullcone '1'           # enable fullcone for this zone
```

Generated nftables rules:

```nft
chain srcnat_wan {
    meta nfproto ipv4 meta l4proto { tcp, udp, sctp, dccp, udplite, icmp, gre } fullcone to :1024-65535
    meta nfproto ipv4 fullcone                   # other protocols
}
chain dstnat_wan {
    meta nfproto ipv4 fullcone
}
```

**UDP only**:

```
config zone
    option name 'wan'
    option masq '1'
    option fullcone '1'
    list fullcone_proto 'udp'     # only UDP gets fullcone, the rest goes through standard masquerade
```

Generated nftables rules:

```nft
chain srcnat_wan {
    meta nfproto ipv4 meta l4proto udp fullcone to :1024-65535  # UDP → fullcone
    meta nfproto ipv4 masquerade                                # other → standard masquerade
}
chain dstnat_wan {
    meta nfproto ipv4 meta l4proto udp fullcone                 # UDP-only reverse mapping
}
```

### Advanced: hand-written nftables rules

For more complex match conditions, drop a file into `/etc/nftables.d/`:

```nft
# /etc/nftables.d/10-fullcone-custom.nft
table inet fullcone-custom {
    chain srcnat {
        type nat hook postrouting priority srcnat + 1; policy accept;
        oifname "eth1" ip saddr 192.168.1.0/24 meta l4proto udp fullcone to :1024-65535
    }
    chain dstnat {
        type nat hook prerouting priority dstnat + 1; policy accept;
        iifname "eth1" meta l4proto udp fullcone
    }
}
```

Notes:
- When using custom rules, disable the fw4 fullcone option for that zone to avoid conflicts.
- fw4 emits its dstnat fullcone rules after the `chain-append` user includes, so custom dstnat rules take precedence.
- **The SNAT side must carry a port range** (`to :1024-65535`), or 3-tuple uniqueness is not enforced; see the next section. The DNAT side needs none.

### Why SNAT rules carry a port range

The SNAT rules fw4 generates carry `to :1024-65535` for every protocol that has a port (or an ICMP id, or a GRE key). This is not decoration — **without it, 3-tuple uniqueness is not enforced at all**.

The kernel patch is a faithful port of SONiC's, and its `get_unique_tuple()` checks 3-tuple uniqueness only in the `NF_NAT_RANGE_PROTO_SPECIFIED` branch; when the rule has no port range, a translated port is accepted on an ordinary 5-tuple check. SONiC itself never takes that path: its [natmgr](https://github.com/sonic-net/sonic-swss/blob/master/cfgmgr/natmgr.cpp) only adds `--fullcone` when a port range is configured.

What happens without a port range (two inside hosts, both on local UDP 40000, talking to different destinations):

| | no range | with `to :1024-65535` |
|---|---|---|
| host 1 → destination A | external port 40000 | external port 40000 |
| host 2 → destination B | external port **40000**, shared with host 1 | another port |
| unknown sender → external port 40000 | delivered to **host 2** | delivered to host 1 |
| host 1 → destination B | **another port**; the mapping depends on the destination | external port 40000 |

The host that mapped first loses inbound reachability on its own port, and its external port changes with the destination — exactly the two things full cone is meant to prevent.

nft only accepts a port range after a match on a protocol, so SNAT is split into two rules: the protocols with a port, id or key get the range, everything else is as before. DNAT carries no range, as in SONiC.

**The range starts at 1024**, so an inside host using a source port below 1024 is mapped above it. That is deliberate: full cone forwards every inbound packet sent to a mapped external port to the inside host, so if a LAN client goes out from source port 500 (IPsec IKE) while the router itself runs IPsec on the WAN, that mapping takes the router's port 500 (reasoned from the mechanism, not tested). Router services above 1024 (WireGuard's 51820, for example) are not protected by the range.

## Verification

After flashing the firmware:

```bash
# 1. Confirm the nftables fullcone expression is available (fw4)
echo 'add table ip test; add chain ip test t { fullcone; }' | nft -c -f -
# No error → kernel support is in place

# 2. Check that rules were generated (depends on UCI configuration)
nft list ruleset | grep fullcone                            # fw4
iptables -t nat -S | grep FULLCONE                          # fw3

# 3. Test NAT type from a LAN host
pystun3   # or NatTypeTester on Windows
# Expected result: "Full Cone"

# 4. Test a TCP connection opened from outside (run Natter on a LAN host)
python3 natter.py   # no arguments: test mode, prints Please check [ http://PUBLIC_IP:PORT ]
# "WAN > PUBLIC_IP:PORT" should read [ OPEN ] ([ UNKNOWN ] if the check service is unreachable).
# More direct: open that address from outside the LAN (a phone with Wi-Fi off) and expect "It works!"
# Then, on the router, find the outside device's entry (its first src= is that device):
grep "dport=PORT " /proc/net/nf_conntrack
# Expect ESTABLISHED with [ASSURED]; SYN_SENT with [UNREPLIED] means replies were not NATed
```

Step 3 only proves that packets sent from outside get in: the STUN server answers from a different address, and the client reports Full Cone as soon as that answer arrives, without replying. Whether the LAN host's replies to a connection opened from outside are translated is what step 4 checks. TCP needs a three-way handshake, so when this path is broken the connection never comes up, and a capture on the WAN shows a SYN+ACK leaving with the LAN source address; that is what [#6](https://github.com/mufeng05/openwrt-sonic-fullcone/issues/6) ran into. Step 4 uses [Natter](https://github.com/MikeWang000000/Natter) and needs a public IP on the router's WAN, or a full-cone NAT upstream.

Note: connections that already exist keep the NAT behaviour they started with; `firewall restart` does not change them. Rebooting the router before testing is the safe choice. On platforms that hand connections to hardware forwarding, such as NSS/ECM, flushing conntrack does not remove connections that are already accelerated.

## Known limitations

### fw3 (iptables)

- **fw3 only generates IPv4 NAT rules**: in firewall3's `zones.c`, `print_zone_rule` checks `if (zone->masq && handle->family == FW3_FAMILY_V4)` inside the `FW3_TABLE_NAT` branch, and IPv6 takes a different path. As a result, this project's fw3 patch only emits IPv4 fullcone rules.

  **Adding IPv6 fullcone manually**: kernel-side patches 984/985 and the userspace 901 patch are protocol-family agnostic — the `FULLCONE` target is registered for both IPv4 and IPv6. Add the rules in `/etc/firewall.user` (the port range on the POSTROUTING side is explained in the next item):

  ```bash
  ip6tables -t nat -A POSTROUTING -o wan -p tcp -j FULLCONE --to-ports 1024-65535
  ip6tables -t nat -A POSTROUTING -o wan -p udp -j FULLCONE --to-ports 1024-65535
  ip6tables -t nat -A POSTROUTING -o wan -j FULLCONE
  ip6tables -t nat -A PREROUTING  -i wan -j FULLCONE
  ```

- **The FULLCONE rules fw3 generates carry no port range**: `-j FULLCONE` is emitted without `--to-ports`, so the kernel never sees `NF_NAT_RANGE_PROTO_SPECIFIED` and 3-tuple uniqueness is not enforced; the behaviour and the reason are in [Why SNAT rules carry a port range](#why-snat-rules-carry-a-port-range). fw4 is fixed; fw3 is not yet (judged from the code, not tested on fw3).

### General

- **Port parity preservation (RFC 4787 REQ-3b) is not implemented**: fullcone port allocation does not guarantee odd-to-odd / even-to-even mapping. Modern applications (gaming, WebRTC, VoIP) almost never depend on this.

- **Hairpinning (NAT loopback) is out of scope**: a LAN host reaching another LAN host through the WAN address requires a separate NAT reflection rule.

## Comparison with other fullcone implementations

| | SONiC (this project) | xt_FULLCONENAT | nft-fullcone | bcm-fullconenat |
|---|---|---|---|---|
| Mapping storage | Inside conntrack itself | Parallel hash table | Parallel hash table | conntrack expectation table |
| Read-path locking | RCU (no spinlock) | Global spinlock | Global spinlock | Global expect lock |
| Lookup complexity | O(1) hash | O(1) hash | O(1) hash | **O(N) full scan** |
| Protocol coverage | All L4 | UDP only | UDP only | UDP only |
| Cleanup | Automatic (conntrack lifecycle) | Workqueue GC | Workqueue GC | Expectation timeout |
| Per-rule control | Yes (flag bit) | No | No | No |
| EIM + EIF | Dual hooks (PRE+POST) automatically | Dual hooks automatically | Dual hooks automatically | Relies on the helper mechanism |
| Extra kernel module | None (embedded in nf_nat / nft_masq / xt_MASQUERADE) | Standalone .ko | Standalone .ko | None (in-tree patch) |

## Relationship with turboacc

This project and [turboacc](https://github.com/coolsnowwolf/luci/tree/openwrt-23.05/applications/luci-app-turboacc) are not the same kind of thing and cannot be directly compared.

### Different scope

| | openwrt-sonic-fullcone | turboacc |
|---|---|---|
| What it is | Does one thing: full cone NAT | A bundle of acceleration engines (Flow Offloading + SFE + fullcone + BBR) |
| Fullcone implementation | SONiC kernel patch (Broadcom, production-grade) | Chion / llccd's xt_FULLCONENAT + bcm-fullconenat + nft-fullcone (community implementations) |

### Side-by-side on fullcone

For detailed technical comparison see the [fullcone implementations table](#comparison-with-other-fullcone-implementations) above; here is the diff against turboacc specifically:

| | sonic-fullcone (this project) | turboacc's fullcone |
|---|---|---|
| Granularity | per-zone, per-proto | global toggle |
| fw3 support | Embedded in xt_MASQUERADE.ko | Standalone xt_FULLCONENAT.ko, or embedded in nf_nat_masquerade (bcm) |
| fw4 support | Embedded in nft_masq.ko | Standalone nft_fullcone.ko |
| LuCI | Integrated into the native firewall pages | Separate luci-app-turboacc page |
| Code provenance | Official SONiC kernel patch | Patchwork from several community projects |

### Where each one wins

**sonic-fullcone is better at**:
- Cleaner kernel implementation (RCU, no separate module, every protocol)
- Clear code provenance (official SONiC, written by Broadcom engineers)
- Focused feature set, easier to maintain

**turboacc is better at**:
- It's not just fullcone — also Flow Offloading, SFE, BBR, and other acceleration features
- Mature community ecosystem, large user base, well battle-tested
- Chion's xt_FULLCONENAT only supports UDP, but UDP covers ~99% of fullcone use cases and has years of production usage behind it

### Bottom line

- **Fullcone only** → sonic-fullcone (better architecture, more complete coverage)
- **Full acceleration bundle** → turboacc (fullcone is just one of its sub-features)
- **Can the two coexist?** → Not recommended; the fullcone kernel patches conflict.

For most home router users, turboacc's Chion fullcone (UDP only) is enough — gaming, P2P, and VoIP are all UDP. The full-protocol coverage and RCU read path of sonic-fullcone matter more under high concurrency, which home users rarely notice.

That said, in today's **fw4 (nftables)** environment, the rest of turboacc is no longer irreplaceable:

- **Flow Offloading**: shipped with OpenWrt — just enable `kmod-nft-offload` in `menuconfig`.
- **BBR**: in the kernel — pick `TCP_CONG_BBR` in `menuconfig`, or set it via a one-line `sysctl`.
- **SFE**: depends on the iptables conntrack chain events interface, incompatible with fw4 / nftables, only usable on fw3.

So for fw4 users, turboacc's unique value is essentially fullcone. Its real value lives on **older fw3 firmware**: SFE + fullcone + a unified LuCI panel is genuinely convenient there. But since OpenWrt 23.05+ defaults to fw4, that value keeps shrinking.

## File layout

```
kernel/
  984-add-sonic-fullcone-support.patch      # nf_nat_core.c: 3-tuple hash table, EIM/EIF
  985-add-sonic-fullcone-to-ipt.patch       # xt_MASQUERADE.c: iptables FULLCONE target (PRE+POST)
  986-add-sonic-fullcone-to-nft.patch       # nft_masq.c: nftables fullcone expression (PRE+POST)

patches/
  iptables/901-sonic-fullcone.patch         # libxt_NAT.c: register the FULLCONE userspace target
  libnftnl/001-libnftnl-*.patch             # fullcone expression serialization
  nftables/002-nftables-*.patch             # nft CLI "fullcone" keyword
  luci-app-firewall/001-add-*.patch         # LuCI Web UI integration

translations/
  zh_Hans.po                                # Simplified Chinese translation (appended to po files at install time)

firewall/
  firewall3/001-sonic-fullcone.patch        # fw3: per-zone, per-proto, EIM+EIF
  firewall4/001-sonic-fullcone.patch        # fw4: per-zone, per-proto, EIM+EIF
```

## Acknowledgements

- Kernel patch: Akhilesh Samineni (Broadcom), from [sonic-net/sonic-linux-kernel](https://github.com/sonic-net/sonic-linux-kernel)
- nftables / libnftnl expression interface: Syrone Wong (fullcone-nat-nftables)
- OpenWrt integration: openwrt-sonic-fullcone contributors

## License

Kernel patches: GPL-2.0 (in line with the Linux kernel license)
Userspace patches: GPL-2.0
