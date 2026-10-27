# Security Policy

[English](#english) | [简体中文](#简体中文)

## English

### Security contact

This project is maintained by [@mufeng05](https://github.com/mufeng05), who is also its security contact. Please report vulnerabilities privately as described below, not in public issues, pull requests or discussions.

### Reporting a vulnerability

Use GitHub private vulnerability reporting: open the [Security](https://github.com/mufeng05/openwrt-sonic-fullcone/security) tab and click **Report a vulnerability**, or go to <https://github.com/mufeng05/openwrt-sonic-fullcone/security/advisories/new>.

Please include:

- OpenWrt version or commit, kernel version, target/architecture, and whether you use firewall3 (iptables) or firewall4 (nftables)
- the relevant parts of `/etc/config/firewall` (zones and fullcone options)
- steps to reproduce, a proof of concept if you have one, and the impact

I will acknowledge your report as soon as I can, keep you informed while it is confirmed and fixed, and agree on a disclosure date with you. After a fix is available I will publish a GitHub Security Advisory and credit you, unless you prefer not to be named.

### Supported versions

There are no tagged releases. Security fixes go to the latest commit on `master`, for the OpenWrt branches (master / 25.12 / 24.10) and kernels (6.6 / 6.12 / 6.18) listed in the README. Please check that the issue still reproduces on the latest `master`.

### Scope

In scope:

- the kernel patches under `kernel/`, for example memory-safety, reference-counting or locking bugs in the conntrack/NAT changes, or mapping lookups that return the wrong internal host
- the userspace patches under `firewall/` and `patches/` (firewall3, firewall4, iptables, libnftnl, nftables, luci-app-firewall) and `add_sonic_fullcone.sh`
- traffic reaching an internal host or port that fullcone NAT was not enabled for, for example by bypassing the per-zone or per-protocol settings

The kernel patch is ported from [SONiC's fullcone NAT patch](https://github.com/sonic-net/sonic-linux-kernel/blob/master/patches-sonic/Support-for-fullcone-nat.patch). If a vulnerability also affects the original patch, please consider reporting it to SONiC as well.

Not vulnerabilities:

- Endpoint-independent mapping and filtering is how full cone NAT works: once an internal host opens a mapping on a zone and protocol with fullcone enabled, any external host can send packets to it.
- Issues in unmodified OpenWrt, the Linux kernel or other upstream projects; please report those to the project concerned.

## 简体中文

### 安全联系人

本项目的维护者兼安全联系人是 [@mufeng05](https://github.com/mufeng05)。发现安全漏洞请按下面的方式私下报告，不要发在公开的 Issue、Pull Request 或 Discussion 里。

### 报告漏洞

请使用 GitHub 私密漏洞报告：打开仓库的 [Security](https://github.com/mufeng05/openwrt-sonic-fullcone/security) 页签，点击 **Report a vulnerability**，或直接访问 <https://github.com/mufeng05/openwrt-sonic-fullcone/security/advisories/new>。

请尽量提供：

- OpenWrt 版本或 commit、内核版本、目标平台/架构，以及使用的是 firewall3（iptables）还是 firewall4（nftables）
- `/etc/config/firewall` 中相关的配置（zone 和 fullcone 选项）
- 复现步骤、PoC（如有）以及影响

我会尽快确认收到报告，在确认和修复期间同步进展，并与你商定公开时间。修复发布后，我会发布 GitHub 安全公告（Security Advisory）并致谢报告者，除非你希望匿名。

### 支持的版本

本项目没有打标签的发布版本。安全修复只进入 `master` 分支的最新提交，覆盖 README 中列出的 OpenWrt 分支（master / 25.12 / 24.10）和内核（6.6 / 6.12 / 6.18）。报告前请先确认问题在最新的 `master` 上仍能复现。

### 范围

属于本项目的范围：

- `kernel/` 下的内核补丁，例如 conntrack/NAT 改动中的内存安全、引用计数或加锁问题，或映射查找返回了错误的内部主机
- `firewall/` 和 `patches/` 下的用户态补丁（firewall3、firewall4、iptables、libnftnl、nftables、luci-app-firewall）以及 `add_sonic_fullcone.sh`
- 流量到达了未启用全锥形 NAT 的内部主机或端口，例如绕过了按 zone 或按协议的设置

内核补丁移植自 [SONiC 的 fullcone NAT 补丁](https://github.com/sonic-net/sonic-linux-kernel/blob/master/patches-sonic/Support-for-fullcone-nat.patch)。如果漏洞同样影响原始补丁，也请考虑向 SONiC 报告。

不属于漏洞：

- 端点无关映射和端点无关过滤是全锥形 NAT 的工作方式：内部主机在启用了 fullcone 的 zone 和协议上建立映射后，任何外部主机都可以向该映射发送数据包。
- 未经本项目修改的 OpenWrt、Linux 内核或其他上游项目中的问题，请向对应项目报告。
