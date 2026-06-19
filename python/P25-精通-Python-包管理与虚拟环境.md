# 精通 Python 包管理与虚拟环境

> Python 的依赖管理曾是"一团乱麻"——全局装包污染、版本冲突、环境不可复现。虚拟环境 + 现代工具(尤其 2026 的 **uv**)解决了这些。面试问:"为什么要虚拟环境?""pyproject.toml 是什么?"本篇梳理依赖与打包,对照 [Go modules G17](../golang/INDEX.md)、[Java Maven](../java/INDEX.md)。
>
> **📅 基准:2026 年 6 月,uv 成主流,pyproject.toml 标准化。**

---

## 一、虚拟环境:隔离是基础

不同项目依赖不同版本的包,装在**全局**会互相冲突、污染系统 Python。**虚拟环境**给每个项目一个独立的、隔离的包目录:

```bash
python -m venv .venv          # 创建虚拟环境(标准库 venv)
source .venv/bin/activate     # 激活(Windows: .venv\Scripts\activate)
pip install requests          # 装到这个环境,不污染全局
deactivate                    # 退出
```

- **`venv`**(标准库)创建隔离环境;每个项目一个 `.venv`。
- 激活后 `python`/`pip` 指向环境内的;`which python` 验证。
- **永远在虚拟环境里装包**,别往系统 Python 装(会污染、需 sudo、易冲突)。

---

## 二、pip 与 requirements.txt

`pip` 是包安装器:

```bash
pip install requests==2.31.0       # 装指定版本
pip install -r requirements.txt    # 按清单装
pip freeze > requirements.txt      # 导出当前环境所有包及精确版本
pip list                           # 看已装
```

- **`requirements.txt`**:列出依赖。但 `pip install pkg` 不锁版本——下次装可能得到不同版本,**不可复现**。
- **`pip freeze`** 导出**精确版本**(含传递依赖)可提高复现性,但它混合了直接与间接依赖、不够清晰。
- 这正是现代工具(锁文件)要解决的问题(见下)。

---

## 三、依赖解析与冲突

项目的依赖有**传递依赖**(你装 A,A 依赖 B、C…)。当两个包要求**同一个依赖的不兼容版本**时,产生**依赖冲突(dependency hell)**。

- pip 有依赖解析器(会尝试找一组兼容版本),但 `requirements.txt` 手动维护难保证一致。
- **锁文件(lock file)**:记录**整棵依赖树的精确版本**(含哈希),保证任何人、任何时候装出**完全相同**的环境——这是可复现构建的关键(类比 [Go 的 go.sum](../golang/INDEX.md)、npm 的 lock)。
- 现代工具(poetry/uv)自动解析 + 生成锁文件。

---

## 四、pyproject.toml:标准化配置

**`pyproject.toml`(PEP 518/517/621)** 是现代 Python 项目的**统一配置文件**——声明项目元数据、依赖、构建系统、工具配置,取代散落的 `setup.py`/`setup.cfg`/`requirements.txt`:

```toml
[project]
name = "myapp"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = ["fastapi>=0.110", "httpx"]

[project.optional-dependencies]
dev = ["pytest", "ruff", "mypy"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.ruff]                  # 工具配置也集中在这里
line-length = 100
```

`[project]` 声明依赖(PEP 621)、`[build-system]` 指定构建后端、`[tool.*]` 放各工具(ruff/mypy/pytest)配置——一个文件搞定。

---

## 五、工具:pip / poetry / uv

| 工具 | 定位 |
|---|---|
| **pip + venv** | 标准库自带,基础但需手动管理锁/环境 |
| **poetry** | 一体化:依赖管理 + 锁文件 + 虚拟环境 + 打包,曾是主流 |
| **uv**(Rust) | **2026 主流**:极快(比 pip/poetry 快 10-100×),一体化管理环境/依赖/锁/Python 版本/工具,兼容 pip 接口 |

```bash
# uv:2026 的明星工具(Rust 实现,极速)
uv venv                     # 建虚拟环境
uv add fastapi              # 加依赖(自动更新 pyproject.toml + uv.lock)
uv sync                     # 按锁文件精确安装(可复现)
uv run pytest               # 在环境里跑命令
uv python install 3.13      # 连 Python 版本也管(取代 pyenv)
```

**uv** 用 Rust 写、速度碾压,把 pip/venv/poetry/pyenv/pipx 的功能整合,并生成 `uv.lock` 锁文件保证复现——2026 新项目的推荐选择。

---

## 六、打包与发布

把项目打包成可分发格式发到 PyPI:

```bash
python -m build              # 构建 sdist(源码包 .tar.gz)+ wheel(.whl)
python -m twine upload dist/*  # 上传到 PyPI(或 uv publish)
```

- **wheel(`.whl`)**:预构建的二进制分发格式,**装得快**(无需在用户机器编译),是首选分发格式。
- **sdist(源码分发)**:源码包,装时可能需要编译。
- 版本遵循**语义化版本(SemVer)**:`主.次.补丁`;依赖约束用 `>=`/`~=`/`==` 表达兼容范围。

---

## 陷阱清单

- **往全局/系统 Python 装包**:污染、冲突、可能需 sudo、破坏系统工具。永远用虚拟环境。
- **不锁版本**:`requirements.txt` 只写包名 → 不同时间装出不同版本,环境不可复现、"在我机器上能跑"。用锁文件(uv.lock/poetry.lock)。
- **`pip freeze` 当依赖声明**:它混了直接和传递依赖、平台相关包,难维护;声明直接依赖用 pyproject.toml,复现用锁文件。
- **手动维护复杂依赖**:依赖冲突难手动解;用 poetry/uv 自动解析。
- **提交 `.venv` 到 git**:体积大、平台相关;`.gitignore` 掉,提交 pyproject.toml + 锁文件。
- **忽视 `requires-python`**:不声明支持的 Python 版本,用户装错版本出错。
- **直接传 setup.py 时代的老配置**:迁移到 pyproject.toml(PEP 621)。

---

## 2026 现状

- **uv(Rust)成为主流**:极速、一体化(环境/依赖/锁/Python 版本/工具),很多项目和 CI 已从 pip/poetry/pyenv 迁过来;`uv.lock` 保证复现。
- **pyproject.toml 是事实标准**:依赖、构建、工具配置统一在此(PEP 621);`setup.py` 基本退役。
- **wheel 是分发标准**;PyPI 上传用 twine/uv publish;私有源/镜像常见。
- **ruff(Rust,lint+format)**、**mypy/pyright(类型)** 配置都进 pyproject.toml,组成现代工具链。
- 与 Go/Java 对照:Go modules([G17](../golang/INDEX.md))、Java Maven/Gradle 早有统一的依赖+锁+构建;Python 经历了从 setuptools/pip 散乱到 pyproject.toml + uv 收敛的过程,2026 终于体验接近。

---

## 练习题

1. 为什么要用虚拟环境?不用会有什么问题?

<details><summary>参考答案</summary>

**虚拟环境为每个项目提供一个独立、隔离的 Python 包安装空间**,使不同项目的依赖互不干扰。**不用(直接往全局/系统 Python 装包)的问题**:①**版本冲突(依赖地狱)**——不同项目往往需要同一个库的不同版本(项目 A 要 Django 4、项目 B 要 Django 5),装在全局只能有一个版本,必然冲突、相互破坏;②**污染系统 Python**——很多操作系统自带 Python 并被系统工具依赖,往系统 Python 乱装/升级包可能**破坏系统功能**;而且全局装包常需要 `sudo`(权限风险);③**环境不可复现/不透明**——全局环境混入了各种项目的依赖,搞不清某个项目到底需要哪些包,难以在别的机器/CI 上重建一致环境;④**难以清理**——卸载困难、残留多。**虚拟环境解决这些**:用 `python -m venv .venv` 为项目创建一个独立目录,激活后 `pip install` 只装到这个环境里、`python` 也用环境内的解释器,与全局和其他项目完全隔离;每个项目一个 `.venv`,依赖清晰、可随时删除重建、不需 sudo、不污染系统。配合 `pyproject.toml`/锁文件,还能在任何机器上精确重建相同环境。**最佳实践**:**永远在虚拟环境里开发和装包**,绝不往系统 Python 装项目依赖;`.venv` 加入 `.gitignore`(不提交),而提交 `pyproject.toml` + 锁文件让别人能复现。现代工具如 uv/poetry 会自动帮你管理虚拟环境。

</details>

2. requirements.txt 和锁文件(lock file)有什么区别?为什么需要锁文件?

<details><summary>参考答案</summary>

**`requirements.txt`** 是一个列出项目依赖的简单文本文件,但它的内容和用法比较随意:可以只写包名(`requests`)、写宽松约束(`requests>=2.0`)、或写精确版本(`requests==2.31.0`)。问题在于:①如果不写精确版本(只写包名或范围),`pip install -r requirements.txt` 在**不同时间**执行,可能解析安装到**不同的版本**(因为新版本发布了),导致环境不可复现;②它通常**只记录直接依赖**,而依赖的**传递依赖(依赖的依赖)**的版本没有被锁定,即使你锁了直接依赖,间接依赖仍可能变化;③手动维护容易遗漏或不一致。**锁文件(lock file,如 `uv.lock`、`poetry.lock`、`Pipfile.lock`)** 则不同:它由工具**自动生成**,记录的是经过依赖解析后**整棵依赖树(直接 + 所有传递依赖)的精确版本号,通常还带内容哈希**。**为什么需要锁文件——为了可复现构建(reproducible builds)**:有了锁文件,任何人、在任何机器、任何时间执行安装(如 `uv sync`/`poetry install`),都会装出**完全相同**的依赖版本组合,彻底消除"在我机器上能跑、在你那/CI 上出错"的版本漂移问题;它还记录哈希用于校验完整性(防篡改)。这类似 Go 的 `go.sum`、JavaScript 的 `package-lock.json`/`yarn.lock`。**实践分工**:用 `pyproject.toml`(或 requirements.in)声明你**关心的直接依赖及其宽松约束**(表达"我要 fastapi 0.110 以上"),用**锁文件**记录解析出的**精确全量版本**用于复现安装;开发者更新依赖时重新解析生成新锁文件并提交。所以现代做法是 pyproject.toml + 锁文件,而不是手写一个混杂的 requirements.txt(`pip freeze` 出来的也不适合当依赖声明,因为它混了直接和间接依赖、且平台相关)。

</details>

3. pyproject.toml 是什么?它取代了哪些东西?

<details><summary>参考答案</summary>

`pyproject.toml` 是现代 Python 项目的**统一的、标准化的配置文件**(基于 PEP 518/517/621 等),用 TOML 格式集中声明项目的**元数据、依赖、构建系统和各种工具的配置**。它的主要组成:①**`[build-system]`(PEP 518/517)**——声明构建这个项目需要哪个**构建后端**(如 hatchling、setuptools、flit、pdm 等)和构建依赖,使得构建过程标准化、不再依赖直接执行 `setup.py`;②**`[project]`(PEP 621)**——声明项目的标准元数据:`name`、`version`、`description`、`requires-python`(支持的 Python 版本)、`dependencies`(运行依赖列表)、`optional-dependencies`(可选/分组依赖,如 dev 组)、作者、许可证、入口点(scripts)等,格式被所有现代工具识别;③**`[tool.*]`**——各种开发工具的配置也集中放这里,如 `[tool.ruff]`、`[tool.mypy]`、`[tool.pytest.ini_options]`、`[tool.uv]`/`[tool.poetry]` 等,一个文件管所有工具配置。**它取代了什么**:在 pyproject.toml 之前,Python 项目的配置散落在多个文件里——`setup.py`(可执行的构建脚本,声明元数据和依赖,但因为是任意 Python 代码而难以静态解析、有安全和复现问题)、`setup.cfg`(setup 的声明式配置)、`requirements.txt`(依赖清单)、`MANIFEST.in`、以及各工具自己的配置文件(`.flake8`、`pytest.ini`、`mypy.ini` 等)。pyproject.toml 把这些**收敛到一个标准化的声明式文件**:元数据和依赖用 `[project]`、构建用 `[build-system]`、工具配置用 `[tool.*]`,大大简化了项目结构、提升了可解析性和工具互操作性。如今 **setup.py 基本退役**(仅在需要复杂自定义构建时还用),新项目都用 pyproject.toml;uv、poetry、hatch、pdm、ruff、mypy 等现代工具都围绕它工作。

</details>

4. pip、poetry、uv 各是什么?2026 推荐用哪个?

<details><summary>参考答案</summary>

①**pip + venv**:Python **标准库自带**的基础组合——`venv` 创建虚拟环境,`pip` 安装包。它们是最基础、最通用的工具,但功能较"原始":pip 本身不管理虚拟环境、不自动生成锁文件、不做项目级的依赖声明管理,这些都要你手动配合 requirements.txt、`pip freeze`、单独建 venv 来做,容易出现版本不锁、环境不可复现等问题。②**poetry**:一个流行的**一体化项目/依赖管理工具**——它统一管理依赖声明(在 pyproject.toml 的 `[tool.poetry]` 或标准 `[project]`)、自动**依赖解析并生成锁文件 `poetry.lock`**(保证可复现)、自动管理虚拟环境、以及打包发布。它大大改善了 pip 时代的体验,曾是社区主流的现代方案。③**uv**:用 **Rust 编写**的新一代工具(由 Astral 出品,同公司还做了 ruff),**2026 的明星/主流**——它把虚拟环境管理、依赖安装与解析、锁文件(`uv.lock`)、甚至 **Python 版本管理(取代 pyenv)**、工具运行(取代 pipx)等功能**整合在一个工具里**,且**速度极快**(比 pip/poetry 快 10-100 倍,得益于 Rust 实现和高效缓存/并行),同时**兼容 pip 的命令接口**便于迁移。常用:`uv venv`、`uv add 包`、`uv sync`(按锁文件精确安装)、`uv run 命令`、`uv python install 3.13`。**2026 推荐**:**新项目优先用 uv**——它速度快、一体化、生成锁文件保证复现、还管 Python 版本,体验和性能都最好,正在被大量项目和 CI 采用;poetry 仍是成熟可用的选择(已有项目可继续);pip+venv 作为标准库永远可用、是最低共同基础(简单脚本或不想引入额外工具时用,但要自己注意锁版本和环境隔离)。总体趋势是从"pip+venv 手动 / poetry"向 **uv** 收敛。

</details>

5. wheel 和 sdist 有什么区别?语义化版本是什么?

<details><summary>参考答案</summary>

**sdist(source distribution,源码分发)**和 **wheel(`.whl`,构建分发)**是 Python 包在 PyPI 上的两种分发格式。**sdist** 是**源码包**(通常是 `.tar.gz`),里面包含项目源代码和构建所需文件;用户安装它时,pip 需要在**用户的机器上执行构建过程**(运行构建后端),如果包含 C 扩展还需要本地有编译器和相应的头文件,因此**安装可能较慢、且可能因缺编译环境而失败**。**wheel** 是**预先构建好的二进制分发格式**(一个 zip 结构的 `.whl` 文件),里面是**已经构建/编译好、可直接解压安装**的内容(对纯 Python 包就是现成的 .py;对含 C 扩展的包则是已为特定平台/Python 版本编译好的二进制);安装时 pip **几乎只需解压拷贝、无需编译**,所以**装得快、可靠、不依赖用户的编译环境**(平台相关的 wheel 会标注适用的 OS/架构/Python 版本)。所以 **wheel 是首选的分发格式**(尤其有 C 扩展的包),发布时通常**同时提供 wheel 和 sdist**(wheel 给大多数用户快速安装,sdist 作为后备/供从源码构建)。用 `python -m build` 可同时生成两者,用 twine/uv publish 上传。**语义化版本(Semantic Versioning,SemVer)**是一种版本号约定,格式为 **`主版本.次版本.补丁版本(MAJOR.MINOR.PATCH)`**,规则:**MAJOR** 在做**不兼容的 API 变更**(破坏性变更)时递增;**MINOR** 在**向后兼容地新增功能**时递增;**PATCH** 在**向后兼容地修 bug**时递增(还可带预发布/构建元数据如 `1.2.0rc1`)。它让使用者能从版本号判断升级的风险:补丁/次版本升级应当安全,主版本升级可能要改代码。依赖约束正是基于此表达兼容范围,如 `~=1.4`(兼容 1.4.x)、`>=1.4,<2.0`(不跨主版本)、`==1.4.2`(精确)。遵循 SemVer 能让依赖解析和升级更可预测(Python 包大多遵循,但并非强制)。

</details>

6. Python 的依赖管理和 Go modules / Java Maven 相比如何?

<details><summary>参考答案</summary>

**历史与现状对比**:Go 和 Java 较早就有了**统一、内建/官方的依赖+构建体系**,而 Python 长期是"工具碎片化、逐步收敛"的状态。**Go modules(见 G17)**:Go 1.11+ 内建,`go.mod` 声明依赖(含最低版本)、`go.sum` 记录精确版本和哈希(锁定+校验),依赖解析用 MVS(最小版本选择)算法,`go build`/`go test` 自动管理,**无需虚拟环境**(依赖按版本缓存在全局模块缓存、构建时按 go.mod 选定),体验统一简洁、可复现性好。**Java Maven/Gradle**:成熟的构建+依赖工具,`pom.xml`/`build.gradle` 声明依赖和坐标(groupId:artifactId:version),从中央仓库下载,有传递依赖管理、依赖范围(scope)、插件体系;同样不需要"虚拟环境"(JVM 按 classpath 加载指定版本 jar)。**Python**:经历了 `setuptools`/`distutils` + `pip` + 手写 `requirements.txt` 的**碎片化时代**(没有官方锁文件、要手动建虚拟环境隔离、`setup.py` 是可执行脚本难复现),依赖冲突和"在我机器上能跑"问题突出。后来逐步标准化:**`pyproject.toml`(PEP 518/621)统一了配置**,**poetry/pdm 等带来了自动依赖解析 + 锁文件 + 环境管理**,到 2026 **uv(Rust)**进一步整合并极大提速,体验终于接近 Go/Java 的"一个文件声明 + 锁文件复现 + 工具统一"。**关键差异**:①**隔离方式**——Python 靠**虚拟环境**(每项目独立包目录)来隔离不同项目的依赖版本,而 Go/Java 靠"按版本缓存 + 构建时选定版本/classpath"天然隔离、无需虚拟环境;②**统一性**——Go modules 是语言内建官方方案(最统一),Java 有 Maven/Gradle 两大主流,Python 历史上工具众多(pip/poetry/pdm/uv…)在收敛中;③**锁与复现**——三者现在都有锁文件机制(go.sum / Maven 的版本锁定（结合 BOM/lock 插件) / uv.lock·poetry.lock）。总结:Python 起步碎片化、需要虚拟环境、工具多,但通过 pyproject.toml 标准化和 uv 等现代工具,2026 已基本补齐与 Go modules/Maven 相当的"声明依赖、锁定版本、可复现、统一工具"的体验。

</details>
