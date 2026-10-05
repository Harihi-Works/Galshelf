<a id="chinese"></a>

# Galguard · 文件安全与完整性

[返回 GalShelf 首页](README.md#chinese) · [Galdex](GALDEX.md#chinese) · [Galdrive](GALDRIVE.md#chinese) · [English](#english)

**知道游戏文件发生了什么变化，并保留可以恢复的处理方式。**

Galguard 在 GalShelf 0.2.6 中以 **GalShelf Guard** 名称介绍，是可选的独立安全伴侣。作品识别回答「这些文件属于哪部作品」，Guard 则关注文件变化、安全发现和处理方式。

> **现有版本**：GalShelf 0.2.6 的说明将 Guard 标为 **0.1.0 预发布测试版**。本页集中介绍这些已有功能，不发布新安装包，也不改变下载或信任配置。

**未来开发**：[Galguard 未来开发指南](docs/Galguard_Development_Guide.md)（中文导读 / English specification）包含技术调研、主动防护边界、恢复架构、成本模型和分阶段验收计划；这些是开发提案，不代表当前版本已支持。

## 文件基准与变化检查

为已安装作品记录经确认的文件清单，检查程序文件是否被修改、缺失或新增。新的变化不能仅因为文件仍能识别成同一部作品就自动被接受。

补丁或用户修改也可能导致变化。**文件变化不等于中毒**，威胁标记需要扫描结果作为依据。

## 扫描、可恢复隔离与修复

Guard 使用自己的规则分析文件，Microsoft Defender 是可选补充，而不是全部检查能力的前提。

可疑文件可以进入可恢复的隔离区，不直接永久删除；修复使用已经核验的本地副本，不从未知来源静默获取替代文件。

## 身份识别与安全判断分开

GalShelf 会提醒启动文件与已确认基准不一致的情况。存在明确安全发现时，不能用关闭普通检查或成功识别作品来绕过威胁阻止。

Galdex 数据可向兼容客户端提供作品身份依据，但不能代替扫描、改变安全结论，或自动接受一个新的可信文件基准。**知道属于哪部作品，不代表已经证明安全。**

## 安装与使用边界

Guard 是可选组件；不安装时，GalShelf 仍能管理游戏，并保留自身已有的启动文件变化提示。现有客户端可在「设置 › 隐私与安全」中检查、下载、校验并安装 Guard。

0.2.6 文档说明 Guard 安装包尚未做代码签名，安装时 Windows 可能请求管理员权限。具体指引与限制见 [GalShelf 首页](README.md#chinese)。功能说明不是安全认证或零风险保证。

---

<a id="english"></a>

# Galguard · File security and integrity

[GalShelf home](README.md#english) · [Galdex](GALDEX.md#english) · [Galdrive](GALDRIVE.md#english) · [简体中文](#chinese)

**Understand changes to game files and keep recovery possible.**

Galguard is introduced as **GalShelf Guard** in GalShelf 0.2.6. It is an optional, separate security companion. Work identification answers which work a file belongs to; Guard focuses on changes, security findings and handling.

> **Existing version**: the 0.2.6 documentation describes Guard as a **0.1.0 pre-release beta**. This page collects existing capabilities; it publishes no package and changes no download or trust configuration.

**Future development**: the [Galguard Future Development Guide](docs/Galguard_Development_Guide.md) covers research, protection boundaries, recovery architecture, capacity planning and milestone acceptance. It is a development proposal, not a list of shipped features.

## Baselines and file changes

Record a confirmed file list and detect changed, missing or added program files. A new state must not be accepted automatically simply because the files still identify as the same work.

Patches and user modifications can also cause changes. **Changed does not mean malware**; a threat label requires a scanner finding.

## Scanning, reversible quarantine and repair

Guard analyses files with its own rules. Microsoft Defender is an optional supplement, not a prerequisite for all checks.

Suspicious files can enter a reversible quarantine rather than being permanently deleted. Repair uses verified local copies, not replacements silently obtained from unknown sources.

## Keep identity separate from security

GalShelf warns when launch files differ from a confirmed baseline. An explicit security finding must not be bypassed by disabling an ordinary check or successfully identifying a work.

Galdex data can add identity evidence in compatible clients. It does not replace scanning, change security verdicts or automatically establish a new trusted baseline. **Known identity is not proof of safety.**

## Installation and boundaries

Guard is optional. Without it, GalShelf can still manage games and retain its own existing launch-file change warnings. The current client offers checking, downloading, verification and installation under **Settings › Privacy & Security**.

The 0.2.6 documentation notes that the Guard installer is not code-signed and Windows may request administrator approval. See the [GalShelf README](README.md#english) for existing instructions and limitations. A feature description is not a security certification or zero-risk guarantee.
