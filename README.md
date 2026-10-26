<div align="center">

# Adversarial REA: Evidence-First Binary Triage

### Local native telemetry, bounded MCP tools, and unverified review candidates.

**Start with what the binary says. Mark everything else as a question.**

[![npm version](https://img.shields.io/badge/version-0.1.0-cb3837?style=flat-square)](package.json)
[![MCP tool catalog](https://img.shields.io/badge/MCP-12_local_tools-5c4ee5?style=flat-square)](#mcp-tool-catalog)
[![Node.js requirements](https://img.shields.io/badge/Node.js-%3E%3D22.0.0-339933?style=flat-square&logo=nodedotjs&logoColor=white)](#what-you-can-analyze)
[![Swarm roles](https://img.shields.io/badge/Swarm-8_offensive_roles-ef4444?style=flat-square)](#agent-swarm-roles)
[![CWE coverage](https://img.shields.io/badge/CWE-review_patterns-f97316?style=flat-square)](#vulnerability-patterns-cwe-coverage)
[![MIT license](https://img.shields.io/badge/license-MIT-f4c430?style=flat-square)](LICENSE)

**[MCP Guide](MCP_USAGE.md) · [Tool Catalog](#mcp-tool-catalog) · [Showcases](#showcases)**

[Quick start](#quick-start) · [How Adversarial REA works](#how-adversarial-rea-works) · [What you can analyze](#what-you-can-analyze) · [Swarm roles](#agent-swarm-roles) · [Showcases](#showcases) · [Exploit chains](#exploit-chain-templates) · [FAQ](#faq) · [Documentation](#documentation)

<code>node bin/cli.js analyze ./target.elf --no-chain</code>

</div>

> [!IMPORTANT]
> Get **Adversarial REA** from [Cultistsid/adversarial-rea-mcp](https://github.com/Cultistsid/adversarial-rea-mcp).
> Native telemetry extraction runs locally. Imports, strings, call sites, gadget candidates, and generated review candidates are **not verified vulnerabilities**. Native source-to-sink flow and observed native triggers are not produced by this release; chains remain blocked without them. Separately, synthetic `.flow.json` fixtures can be correlated and replayed entirely in memory. Fixture observations cannot validate a native binary.

<div align="center">

<br />

<img src="docs/assets/adversarial-hero.svg" alt="Illustrative future analysis workflow, not a live exploit result" width="1200" />

<br /><br />

</div>

---

Need a first-pass review of a native binary without source code? Connect your MCP client to **Adversarial REA** or use `adv-rea` from a terminal. It extracts bounded symbols, strings, sections, and mitigation signals from local files, then offers follow-up questions rather than claiming that a risky import proves a bug.

Use it to narrow manual review: identify suspicious imports, inspect observed call instructions, and distinguish unknown mitigations from observed absences. Native ELF and Mach-O extraction use host tools; PE headers and mitigation *flags* can be previewed without host PE tools. Packages, archives, WebAssembly, firmware images, text, and JavaScript receive bounded metadata inspection, **not** deep code or exploit analysis.

---

## Quick start

Adversarial REA requires **Node.js `>= 22.0.0`**, `npm`, and standard host binary utilities (`objdump`, `nm`, `strings`, `file`; plus `otool` on macOS).

```bash
git clone https://github.com/Cultistsid/adversarial-rea-mcp.git
cd adversarial-rea
npm install
```

### Set up your MCP client

Add `adversarial-rea` to your MCP client configuration (`mcp.json`):

```json
{
  "mcpServers": {
    "adversarial-rea": {
      "command": "node",
      "args": ["/absolute/path/to/adversarial-rea/src/mcp/main.js"],
      "env": {}
    }
  }
}
```

Restart your MCP client after saving. All 12 inspection tools register over stdio JSON-RPC 2.0 and require **zero external API keys**.

### Ask your agent

```text
Run analyze_binary and trace_calls on ./target.elf. Compare import observations
with actual call instructions; do not claim an input flow or bug from either.
```

Replace `./target.elf` with a native ELF or Mach-O file. For a script or container, use `inspect_carrier` instead. PE code extraction depends on your host `objdump`/`nm` build.

### Use the terminal

Run local inspection without a key; the separate candidate-triage commands use a remote reasoning service and require its API key.

```bash
# Offline metadata for APK/IPA/JAR/ASAR, Wasm, JS/text, PE, firmware, or .app bundles
node bin/cli.js inspect ./package.apk --limit 50

# Reproduce a declarative fixture flow in memory, not a native program
node bin/cli.js inspect test/fixtures/flow-direct.flow.json --flow-fixture

# Offline call sites on native files, or byte-verified gadget candidates on ELF64 x86_64
node bin/cli.js inspect ./target.elf --calls --gadgets

# Candidate analysis on a native ELF file
node bin/cli.js analyze ./target.elf --agents 8 --trace-calls --no-chain --output results.json

# Deploy a targeted agent swarm with custom mission and offensive roles
node bin/cli.js swarm ./target.elf \
  --mission "Review possible heap corruption and address leaks" \
  --roles mem-corrupt,mitigation-bypass

# Triage selected CWE patterns (candidate findings only)
node bin/cli.js hunt ./target.elf --patterns UNCHECKED_BOUNDS,FORMAT_STRING --critical-only

# Attack surface enumeration only (vectors, trust boundaries, primitives)
node bin/cli.js surface ./libtarget.so --type elf
```

To install the `adv-rea` command globally for regular terminal use:

```bash
npm install --global .
adv-rea --help
```

---

## How Adversarial REA works

The native analyzer reads a regular file (up to 64 MiB), identifies its magic bytes, and invokes local tools with argument arrays, timeouts, and output limits. The separate carrier inspector reads bounded headers and indexes without unpacking or executing content. A separate fixture evaluator follows explicit single-argument steps and repeats a deterministic in-memory sink observation; it does not run binaries. Extraction failures remain visible; unknown mitigation status is not converted to "off." Optional reasoning only receives a bounded structural summary, never extracted secret-like string values by default.

<div align="center">

<img src="docs/assets/adversarial-flow.svg" alt="Illustrative architecture showing planned evidence-backed chain analysis" width="1100" />

</div>

1. **Local telemetry (`BinaryAnalyzer`, `CarrierInspector`)** — Native metadata or metadata-only previews for other file formats; carrier previews do not imply code parsing.
2. **Instruction evidence (`CallSiteExtractor`, `GadgetCandidateFinder`)** — Opt-in direct call instructions with target hashes and versioned evidence IDs; selected ELF64 x86_64 gadget candidates are checked against executable file bytes. Neither proves reachable input flow or exploitability.
3. **Review hypotheses (`AttackSurfaceEnumerator`, `VulnHunter`)** — Suggests possible vectors and CWE follow-up checks. Triage scores and remote answers remain unverified.
4. **Evidence gate (`ExploitChainBuilder`)** — Requires independently verified native flow, stage references, and an observed native trigger trace. Those native flow/trigger producers are not implemented; fixture-only records are excluded, so ordinary analyses return no validated exploit chain.

---

## What you can analyze

The file signature decides the native format; `--type` only asserts an expected native format. Use `inspect_carrier` or `adv-rea inspect` for metadata-only carriers; the native `analyze_binary` tool still rejects them.

| Input | Status | What works |
| ----- | ------ | ---------- |
| ELF native file | Native triage where `objdump`, `nm`, and `strings` are available | Bounded metadata, disassembly, RELRO signals, opt-in call sites, and byte-verified gadget candidates **only for ELF64 x86_64** |
| Mach-O native file | Native triage on macOS | Bounded metadata and mitigation signals; opt-in direct call sites select the host architecture by default (including `arm64e`) |
| PE native file (including drivers) | Experimental | `inspect_carrier` checks section raw extents and reads declared entry point, image base, certificate-table extent, and ASLR/NX/CFG flags. It does **not** verify signatures or enforcement; native code/symbol extraction depends on host tools; IOCTL/data-flow analysis is not implemented |
| Text / JavaScript | Metadata-only | Byte and line counts; **no** AST or dependency analysis |
| `.app` directories | Metadata-only | `Contents` entry listing without following child symlinks; no bundle code/IPC parser |
| APK / IPA / JAR / ZIP | Metadata-only | Bounded ZIP central-directory listing cross-checked against local names/methods/extents, suspicious path/symlink flags and declared expansion size; no unpacking or manifest analysis; ZIP64/multi-disk unsupported |
| ASAR archives | Metadata-only | Bounded header file index and declared file offsets; no script execution, AST, or IPC analysis |
| WebAssembly | Metadata-only | Version 1 section offsets/sizes; no instruction decoder |
| Raw `.bin` / `.fw` firmware | Metadata-only | First ELF magic offset and up to 16 structurally plausible ELF header candidates; no filesystem extraction or executable verification |
| Declarative `.flow.json` fixture | Fixture-only | One explicit source, one sink, literal/copy/allowlist steps, and two deterministic in-memory replays; no native flow or vulnerability verdict |

Large files (over 64 MiB) and non-regular files are rejected; flow fixtures have a separate 16 KiB / 32-step limit. Each native subprocess has a 15-second timeout and a 2 MiB output cap; truncated or unavailable evidence is not treated as proof of safety. For an MCP client that should see only one directory, set `ADV_REA_ALLOWED_ROOTS=/absolute/path/to/targets` in its server environment. Without it, any readable local regular file can be submitted for inspection.

<a id="mcp-tool-catalog"></a>

### MCP Tool Catalog

Every tool exposed by `src/mcp/main.js` runs locally with zero network dependencies:

| Tool | What it extracts | Key arguments |
| ---- | ---------------- | ------------- |
| `analyze_binary` | File identity (SHA-256), counts, mitigation signals, redacted string samples, import observations | `target_path`, `strings_min_len` (1–64) |
| `get_symbols` | Up to 100 symbols, optionally filtered by literal substring | `target_path`, `type`, `filter` |
| `get_strings` | Up to 100 categorized strings, sensitive categories redacted by default | `target_path`, `min_length` (1–64), `interesting_only`, `category`, `reveal_sensitive` |
| `check_security` | Known/unknown mitigation signals and a score only if all required checks succeeded | `target_path` |
| `disassemble` | Up to 500 disassembly lines (20 KiB returned) | `target_path`, `function`, `limit` |
| `get_sections` | Bounded native section listing | `target_path` |
| `get_headers` | Bounded native headers | `target_path` |
| `find_dangerous_patterns` | Unverified import/string observations and observed missing mitigation signals | `target_path` |
| `trace_calls` | Decoded direct native calls with target hash and provider version; input flow unknown | `target_path`, `sinks`, `arch`, `limit` |
| `find_gadget_candidates` | ELF64 x86_64 gadget candidates verified against executable file bytes; runtime addresses unknown | `target_path`, `limit` |
| `inspect_carrier` | Bounded metadata for ZIP/APK/IPA/JAR/ASAR, Wasm, JS/text, PE, firmware, and `.app` | `target_path`, `limit` |
| `inspect_flow_fixture` | Explicit fixture dependency path and repeatable in-memory sink reachability; excluded from native evidence gate | `target_path` (`.flow.json`) |

Responses are capped at 64 KiB. `get_strings` intentionally requires `reveal_sensitive: true` to show potentially sensitive string categories; handle that output locally.

### Platform-Normalized Mitigation Scoring

`check_security` reports `null` when any required check is unknown. Where checks succeed, RELRO is ELF-only and excluded from the Mach-O denominator.

| Mitigation | Weight | macOS (`Mach-O`) Detection | Linux (`ELF`) Detection |
| ---------- | :----: | -------------------------- | ----------------------- |
| **PIE** | `20` | `MH_PIE` flag via `otool -hv` | `ET_DYN` / `DYNAMIC` in `objdump -f` |
| **NX** | `25` | Absence of `ALLOW_STACK_EXECUTION` flag | Stack header without executable flag |
| **Stack Canary** | `25` | `__stack_chk_*` symbol | `__stack_chk_*` symbol |
| **RELRO** | `15` / `10` | `n/a` *(excluded from Mach-O max)* | `GNU_RELRO` + `BIND_NOW` (`full` = 15, `partial` = 10) |
| **FORTIFY** | `15` | `__memcpy_chk`, `__sprintf_chk`, `_*_chk` imports | `_*_chk` fortified symbol imports |
| **Stripped** | No score weight | `file` explicit label or `unknown` | `file` explicit label or `unknown` |

---

## Agent swarm roles

Eight specialist roles are available for `adv-rea swarm`; `adv-rea analyze` selects triage patterns rather than deploying all eight. A role's output is a review hypothesis, not proof.

| Role ID | Specialist Name | Offensive Focus Areas |
| ------- | --------------- | --------------------- |
| `mem-corrupt` | **Memory Corruption Hunter** | Stack/heap buffer overflows, use-after-free, double-free, type confusion, integer wraparound |
| `logic-flaw` | **Logic Flaw Analyst** | Authentication bypass, race conditions, TOCTOU, state machine abuse |
| `crypto-weak` | **Cryptographic Weakness Finder** | Weak ciphers (`DES`, `RC4`), hardcoded keys, IV reuse, padding oracles, timing side-channels |
| `input-inject` | **Input Injection Specialist** | OS command injection, uncontrolled format strings, SQL injection, unsafe deserialization |
| `mitigation-bypass` | **Mitigation Bypass Expert** | ASLR info leaks, DEP/NX bypass via ROP/JOP, CFG/CET evasion, stack canary leaks |
| `priv-esc` | **Privilege Escalation Hunter** | Kernel/driver IOCTL flaws, setuid abuse, DLL/dylib hijacking, service misconfigurations |
| `protocol-abuse` | **Protocol Abuse Analyst** | Protocol confusion, request smuggling, state desynchronization, parser differentials |
| `supply-chain` | **Supply Chain Analyzer** | Vulnerable dependencies, build tampering, signature verification bypass, update hijacking |

### Vulnerability Patterns (CWE Coverage)

<details>
<summary><strong>View the 16 CWE-oriented triage patterns</strong></summary>

<br />

| Area | Pattern IDs | What an indicator means |
| ---- | ----------- | ----------------------- |
| Memory | `UNCHECKED_BOUNDS`, `INTEGER_OVERFLOW`, `USE_AFTER_FREE`, `DOUBLE_FREE`, `TYPE_CONFUSION`, `UNINITIALIZED_MEMORY` | A possible code path to inspect; no reachability or memory violation has been established |
| Logic | `RACE_CONDITION`, `AUTH_BYPASS`, `AUTHZ_BYPASS` | Names and hypotheses to verify against control flow |
| Injection | `COMMAND_INJECTION`, `FORMAT_STRING`, `PATH_TRAVERSAL`, `DESERIALIZATION` | Sink presence alone does not establish attacker control |
| Crypto | `WEAK_CRYPTO`, `HARDCODED_SECRET`, `IV_REUSE` | Review flags only; extracted secrets are withheld from prompts |

The `--critical-only` CLI flag filters the *pattern's triage priority*, not a verified finding's severity.

</details>

---

## Showcases

<div align="center">

<img src="docs/assets/adversarial-showcases.svg" alt="Concept illustrations of future analysis scenarios, not verified tool output" width="1100" />

</div>

**The visual panels are illustrative, not generated results.** No addresses, chain scores, or exploit stages in the artwork correspond to a validated fixture.

### Native ELF import triage (available)

Run `analyze_binary` and `find_dangerous_patterns` to identify `printf`/`strcpy` imports for review, then `trace_calls` to locate decoded calls. Verify the controlled data flow and trigger separately; even an observed call does not prove a bug.

### Electron IPC boundary mapping (planned)

`inspect_carrier` lists bounded ASAR file entries. IPC brokers, renderer messages, and sandbox boundaries still require dedicated parsing and observed evidence.

### Firmware header preview

`inspect_carrier` reports ELF magic markers and limited plausible header fields in a raw image. Even a plausible header does not prove an embedded executable or extract firmware; analyze independently extracted native files separately.

---

## Exploit chain templates

Six chain definitions exist for future evidence-backed work. Only four have draft scaffold templates; `PRIV_ESC_WINDOWS` and `AUTH_BYPASS` are explicitly unsupported. Local call-site and constrained gadget evidence are available, but no implemented producer establishes end-to-end **native** input flow or an observed **native** trigger. Synthetic fixture replay does not satisfy that requirement. The generator returns a blocked result when evidence or required facts are missing; it does not export a runnable exploit.

| Template | Stage Progression | Target Objective |
| -------- | ----------------- | ---------------- |
| `CODE_EXEC_LINUX` | `info_leak` → `memory_corruption` → `control_flow_hijack` → `rop_chain` → `shellcode_exec` | Arbitrary code execution on Linux (ASLR + NX bypass) |
| `CODE_EXEC_WINDOWS` | `info_leak` → `memory_corruption` → `cfg_bypass` → `control_flow_hijack` → `rop_chain` → `shellcode_exec` | Arbitrary code execution on Windows (ASLR + DEP + CFG bypass) |
| `PRIV_ESC_LINUX` | `kernel_info_leak` → `kernel_vuln_trigger` → `cred_struct_overwrite` → `root_shell` | Local privilege escalation to `root` |
| `PRIV_ESC_WINDOWS` | `kernel_leak` → `driver_vuln` → `token_stealing` → `system_shell` | Local privilege escalation to `SYSTEM` |
| `SANDBOX_ESCAPE` | `renderer_exploit` → `ipc_message_forge` → `broker_vuln_trigger` → `sandbox_breakout` | Multi-process sandbox escape via IPC broker |
| `AUTH_BYPASS` | `user_enumeration` → `auth_logic_flaw` → `session_forgery` → `privileged_access` | Pre-auth administrative session takeover |

<details>
<summary><strong>Use the programmatic SDK instead</strong></summary>

```javascript
import { AdversarialREA } from 'adversarial-rea';

const rea = new AdversarialREA({
  maxAgents: 8,
  aggressiveMode: true,
  parallelAnalysis: true,
  timeout: 300000
});

const analysis = await rea.analyzeTarget('./target.elf', {
  type: 'elf',
  buildChains: true
});

console.log('Candidate triage score:', analysis.attackSurface.candidate_score);
console.log('Review candidates:', analysis.candidates.length);
console.log('Verified vulnerabilities:', analysis.vulnerabilities.length);
```

</details>

---

## FAQ

<details>
<summary><strong>Which MCP clients can use Adversarial REA?</strong></summary>

Any client or agent runtime that supports local MCP servers over stdio JSON-RPC 2.0. Point the client at `node /absolute/path/to/adversarial-rea/src/mcp/main.js` in `mcp.json`.

</details>

<details>
<summary><strong>Does the MCP server require an API key?</strong></summary>

No. The MCP server (`src/mcp/main.js`) executes local binary inspection tools (`objdump`, `nm`, `strings`, `otool`, `file`) directly on disk and requires zero API keys or network access. API credentials are only used when running standalone multi-agent LLM swarms via the `adv-rea` CLI or `AdversarialREA` SDK class.

</details>

<details>
<summary><strong>Do I need Ghidra, IDA, or Rizin installed?</strong></summary>

No. Core binary extraction needs compatible host utilities (`objdump`, `nm`, `strings`, `file`, and `otool` on macOS). The analyzer detects `rizin` and `radare2` but does not use them for deep decompilation.

</details>

<details>
<summary><strong>Does Adversarial REA upload my binary anywhere?</strong></summary>

The binary stays local. CLI reasoning sends bounded metadata (format, sizes, import names, mitigation signals, and string *category counts*) to the configured endpoint; no extracted string values are included by default. A user can explicitly request raw sensitive strings through the local MCP `get_strings` tool with `reveal_sensitive: true`.

</details>

<details>
<summary><strong>How do I run the verification test suite?</strong></summary>

Run the unit and live-binary integration suites with:

```bash
npm test                   # deterministic unit tests (zero network calls)
npm run test:integration   # Live binary & stdio MCP JSON-RPC integration tests
npm run test:all           # Full suite
```

</details>

---

## Documentation

- [MCP Server Usage Guide](MCP_USAGE.md): registration, tool parameters, example prompts, and extraction-to-findings workflow.
- [Agent Skill Specification](.agents/skills/adversarial-re/SKILL.md): offensive methodology, surface scoring formula, and JSON reporting schema.
- [MCP Server Manifest](server.json): machine-readable capability and tool declarations.

---

## Star history

<a href="https://www.star-history.com/?repos=Cultistsid%2Fadversarial-rea&amp;type=date">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://api.star-history.com/chart?repos=Cultistsid/adversarial-rea-mcp&amp;type=date&amp;theme=dark&amp;legend=top-left" />
    <source media="(prefers-color-scheme: light)" srcset="https://api.star-history.com/chart?repos=Cultistsid/adversarial-rea-mcp&amp;type=date" />
    <img alt="Adversarial REA GitHub star history" src="https://api.star-history.com/chart?repos=Cultistsid/adversarial-rea-mcp&amp;type=date" />
  </picture>
</a>

---

## License

[MIT](LICENSE)
