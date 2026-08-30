# CP2K Materials Workflow

Agent-neutral, version-aware automation guidance and helper scripts for CP2K
materials calculations. It turns a structure and a scientific request into a
reviewable calculation plan, a version-matched input, validation gates,
property checks, and reproducible provenance.

Suggested GitHub short description:

> Agent-neutral, version-aware CP2K workflows with input validation, convergence gates, property checks, and reproducible provenance.

## English

### What this repository is

This repository is a reusable skill package for agents such as Codex, Claude
Code, and other tools that can read Markdown and execute local Python scripts.
The workflow is deliberately independent of a particular agent vendor. The
portable contract is `SKILL.md`; `agents/openai.yaml` is optional metadata for
hosts that display OpenAI-style skill cards and is not required at runtime.

The package helps an agent:

- inspect a local or remote CP2K environment;
- route natural-language requests to finite workflows;
- keep CP2K 2024.1 and 2026.2 syntax separate;
- resolve the exact official manual branch;
- audit CIF, XYZ, PDB, POSCAR, and CONTCAR structures without overwriting them;
- create a hash-linked calculation manifest;
- extract literature observations without silently adopting values;
- render and statically lint CP2K inputs;
- plan convergence for the requested property;
- prepare strict SSH and scheduler commands;
- parse normal termination, SCF, energy, and geometry evidence;
- calculate adsorption-energy bookkeeping and validate/subtract density cubes;
- preserve provenance and clearly distinguish `PASS`, limitations, and failure.

It does not contain CP2K itself, a basis/potential distribution, a remote
cluster configuration, a private key, or universal scientific parameters.

### Quick start

The core scripts use only the Python standard library. Use `python3` on Unix
systems or `python` on Windows; Python 3.9 or newer is recommended.

From this directory:

```text
python scripts/self_check.py
python scripts/task_router.py --task "geometry optimization and band structure"
python scripts/environment_audit.py --output runs/environment.json
python scripts/cp2k_version_detect.py --executable /path/to/cp2k.psmp
python scripts/manual_resolver.py --version 2024.1 --sections FORCE_EVAL/DFT/SCF
```

The last command returns `MANUAL_REQUIRED` until the exact official manual is
retrieved. That is an intentional safety state, not a broken installation.
When network access is allowed, retrieve only an allowlisted official branch:

```text
python scripts/manual_cache.py --version 2024.1
```

`runs/` and `manual_cache/` are runtime directories and are ignored by Git.
Do not commit them unless they are intentionally curated evidence for a
separate project repository.

### Typical agent prompt

Give an agent the path to this folder and ask it to use `SKILL.md`, for example:

```text
Use the CP2K Materials Workflow at /path/to/cp2k-materials-workflow.
Audit my structure, determine whether CP2K is available, route the requested
property, and stop at every unresolved scientific or execution decision.
Do not submit a remote job without showing me the final input and asking for
approval.
```

An agent with a native skills directory may copy this complete folder into that
directory. An agent without a skill loader can read `SKILL.md` directly and
invoke the scripts by absolute path. No hook, plugin, slash command, or
vendor-specific API is required.

### End-to-end workflow

1. Provide the structure, requested property, run target, and optional paper.
2. Run task routing and read-only environment/structure audits.
3. Detect the exact CP2K executable and version.
4. Resolve and hash the matching official CP2K manual branch.
5. Record charge, spin, functional, dispersion, basis, potential, cell,
   k-point, cutoff, and convergence decisions with their sources.
6. Select the matching versioned template and render all explicit slots.
7. Run internal lint, an optional external validator, and a real smoke test.
8. Build the property-specific convergence plan.
9. Submit through the scheduler only after explicit approval; monitor output.
10. Validate artifacts, calculate derived properties, hash the finished files,
    and write the report.

### Main commands

```text
# Route a request
python scripts/task_router.py --task "DOS and projected density of states"

# Create a calculation manifest without changing the structure
python scripts/calculation_init.py --structure input_structure/source.cif --task "geometry optimization" --run-target LOCAL --choices-json '{"charge": 0, "charge_source": "USER_SPECIFIED"}' --output-dir calculation

# Audit a structure (ASE is optional; XYZ has a minimal fallback)
python scripts/structure_audit.py input_structure/source.cif --output calculation/structure-audit.json

# Render a version-locked template after making a complete values JSON
python scripts/render_versioned_template.py --version 2024.1 --workflow single_point --values-json assets/values.example.json --output calculation/inputs/example.inp

# Lint the rendered input
python scripts/input_lint.py calculation/inputs/example.inp --version 2024.1

# Parse a CP2K output; a normal-looking file is not enough for PASS
python scripts/cp2k_output_parser.py calculation/outputs/example.out --return-code 0

# Run one local input only after review; the flag is an explicit execution gate
python scripts/run_cp2k.py --executable /path/to/cp2k.psmp --input calculation/inputs/example.inp --output calculation/outputs/example.out --expected-version 2024.1 --allow-run

# Compute E_ads = E_complex - E_host - E_adsorbate (energies in Hartree)
python scripts/compute_adsorption_energy.py --complex-energy-au -10 --host-energy-au -7 --adsorbate-energy-au -2

# Subtract three scalar cube files only after their grids and atom frames match
python scripts/subtract_cube_density.py --complex complex.cube --host host.cube --adsorbate adsorbate.cube --output difference.cube --audit-output cube-audit.json
```

The example values are intentionally placeholders. Basis files, potential
files, cell dimensions, coordinates, charge, spin, k points, and thresholds
must be replaced and scientifically reviewed before submission.

### Version and status policy

The bundled registry records official manual sources, template paths, and
capability status. The public package contains no matching cluster execution
artifacts, so capabilities begin as `NOT_VALIDATED`. This is deliberate:

- `OFFICIAL_SYNTAX_CONFIRMED` means the keyword or section is documented by the
  exact versioned manual.
- `LOCAL_EXECUTION_CONFIRMED` requires a matching executable, exact input,
  successful return, expected artifact, and validation.
- `SUPPORTED` and `SUPPORTED_WITH_LIMITATIONS` are claims about recorded
  evidence, not about the presence of a template.
- `MANUAL_REQUIRED`, `SCIENTIFIC_DECISION_REQUIRED`, and
  `USER_CONFIRMATION_REQUIRED` are blocking states.

The two included template families are not interchangeable. CP2K 2024.1 uses
the legacy sibling DOS/PDOS layout; CP2K 2026.2 uses the recorded nested
DOS/CURVE/PDOS layout. A restart file is kept with the exact version/build
that created it.

### Scientific limitations

The workflow automates deterministic work, but it does not decide scientific
models on the user's behalf. In particular, it will stop for unresolved:

- total charge, multiplicity, spin, magnetic ordering, or periodic
  countercharge treatment;
- DFT+U element/orbital/U/J and its source;
- partial occupancy, disorder, or unsafe structure interpretation;
- slab orientation, termination, thickness, vacuum, or dipole policy;
- adsorption reference state and adsorbate charge;
- NEB endpoint mapping and intermediate definition.

For a work function, a bulk cell with extra vacuum is not automatically a
validated slab. For charges, Mulliken, Lowdin, Hirshfeld, periodic RESP, and
REPEAT-like outputs remain separate methods. For adsorption energy, the
default bookkeeping is `E_ads = E_complex - E_host - E_adsorbate`; the sign
convention and geometry policy must be stated in the report.

### Remote execution and security

The remote helper requires a real private-key file and a non-empty verified
`known_hosts` file. It uses strict host-key checking and never accepts a new
host key automatically. Scheduler submission is represented as a command and
requires an explicit approval flag at the boundary. Never place private keys,
passwords, tokens, or copied cluster configuration in this repository.

See [references/remote_execution.md](references/remote_execution.md) and
[assets/remote_server.example.yaml](assets/remote_server.example.yaml).

### Repository layout

```text
SKILL.md                         agent-neutral operating protocol
agents/openai.yaml               optional host metadata
scripts/                         portable orchestration and validation tools
references/                      detailed, progressive scientific guidance
assets/templates_2024/           CP2K 2024.1 skeletons
assets/templates_2026/           CP2K 2026.2 skeletons
assets/template_registry.json    version and capability registry
assets/*.example.*               safe local/remote/config examples
tests/                            unit and contract tests
```

Use [CONTRIBUTING.md](CONTRIBUTING.md) for changes and
[THIRD_PARTY_ATTRIBUTION.md](THIRD_PARTY_ATTRIBUTION.md) for bundled-resource
attribution.

## 中文

### 这是什么

这是一个与具体 Agent 无关的 CP2K 材料计算工作流 Skill，适用于 Codex、
Claude Code 以及其他能够读取 Markdown 并运行本地 Python 脚本的 Agent。
真正的跨 Agent 入口是 `SKILL.md`；`agents/openai.yaml` 只是部分宿主用于
显示 Skill 卡片的可选元数据，不影响运行。

它可以帮助 Agent：

- 检查本地或远程 CP2K 环境；
- 把自然语言任务路由到有限的工作流；
- 分离 CP2K 2024.1 和 2026.2 的输入语法；
- 解析并记录对应版本的官方手册；
- 审计 CIF、XYZ、PDB、POSCAR、CONTCAR 结构，且不覆盖原文件；
- 创建带哈希关联的计算清单；
- 从论文中提取参数观察，但不自动采用；
- 渲染并静态检查 CP2K 输入；
- 为不同性质制定收敛计划；
- 生成严格的 SSH 和作业调度命令；
- 解析正常结束、SCF、能量和几何优化证据；
- 计算吸附能以及检查/相减电子密度 cube；
- 保存可复现 provenance，并区分通过、有限通过和失败。

它不包含 CP2K 程序、基组/赝势库、任何具体集群配置、私钥或通用的
“万能参数”。

### 快速开始

核心脚本只依赖 Python 标准库。Unix 使用 `python3`，Windows 使用
`python`；建议 Python 3.9 及以上：

```text
python scripts/self_check.py
python scripts/task_router.py --task "geometry optimization and band structure"
python scripts/environment_audit.py --output runs/environment.json
python scripts/cp2k_version_detect.py --executable /path/to/cp2k.psmp
python scripts/manual_resolver.py --version 2024.1 --sections FORCE_EVAL/DFT/SCF
```

在下载对应官方手册之前，最后一个命令会返回 `MANUAL_REQUIRED`。这是有
意设置的安全状态。当允许联网时可以运行：

```text
python scripts/manual_cache.py --version 2024.1
```

`runs/` 和 `manual_cache/` 是运行时目录，已被 Git 忽略；除非要在单独的
项目仓库中保存证据，否则不要把它们提交到这个公共 Skill 仓库。

### 给 Agent 的使用方式

把整个文件夹放入 Agent 的 Skill 目录，或者直接把 `SKILL.md` 的绝对路径
告诉 Agent。例如：

```text
使用 /path/to/cp2k-materials-workflow/SKILL.md。
请审计我的结构，检查 CP2K 是否可用，路由目标性质，并在所有未解决的
科学决策或执行决策处停止。展示最终输入并得到我的批准后，才能提交远程作业。
```

不支持 Skill 自动发现的 Agent 也可以直接读取 `SKILL.md` 并按绝对路径调用
脚本；不依赖插件、Hook、斜杠命令或特定厂商 API。

### 典型流程

提供结构、目标性质、运行位置和可选论文；然后依次执行环境/结构审计、
CP2K 版本检测、精确手册解析、科学参数确认、版本模板渲染、三重输入检查、
性质相关收敛、调度提交、输出验证和 provenance 归档。

### 版本与状态

公共版本不携带某个集群的完整执行证据，因此能力矩阵默认是
`NOT_VALIDATED`。只有在匹配的 CP2K 可执行文件上完成精确输入、正常结束、
预期产物、解析和性质检查后，才能在用户自己的计算记录中升级状态。

2024.1 使用旧式并列 DOS/PDOS 结构，2026.2 使用注册表记录的嵌套
DOS/CURVE/PDOS 结构；两者不能混用，重启文件也不能跨版本直接复用。

### 科学决策边界

总电荷、自旋/磁性、DFT+U、无序结构、slab 与真空、吸附参考态、周期电荷
定义以及 NEB 端点映射没有足够证据时，工作流会停止并标记
`SCIENTIFIC_DECISION_REQUIRED`。它自动化确定性检查，但不会为了让程序运行
而替用户选择物理模型。

### 远程安全与目录

远程辅助脚本强制要求有效私钥文件和经过验证的 `known_hosts`，启用严格主机
密钥检查，不自动接受新主机密钥。提交作业是外部状态变更，必须单独获得明确
批准。不要把私钥、密码、Token 或真实集群配置放入仓库。

主要目录为：`SKILL.md`、`scripts/`、`references/`、`assets/templates_2024/`、
`assets/templates_2026/`、`assets/template_registry.json` 和 `tests/`。

参见 [CONTRIBUTING.md](CONTRIBUTING.md) 与
[THIRD_PARTY_ATTRIBUTION.md](THIRD_PARTY_ATTRIBUTION.md)。
