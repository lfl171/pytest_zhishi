# pytest 入门与实践：从第一个测试到可维护的测试体系

> 面向 Python 开发者与测试工程师的实战指南，涵盖安装、发现规则、断言、fixture、参数化、标记、配置、常用命令和团队实践。

**阅读对象**：刚开始写 Python 自动化测试的开发者、测试工程师，以及希望改进测试项目的团队。  
**预计阅读**：约 15 分钟 · **示例环境**：Python 3.10+、pytest

## 目录

1. [pytest 是什么](#1-pytest-是什么)
2. [安装与第一个测试](#2-安装与第一个测试)
3. [测试发现与命名](#3-测试发现与命名)
4. [assert 与失败信息](#4-assert-与失败信息)
5. [Fixture：组织前置条件和清理](#5-fixture组织前置条件和清理)
6. [参数化](#6-参数化用一份测试覆盖多组数据)
7. [Marker：给测试分类与筛选](#7-marker给测试分类与筛选)
8. [常用内置能力](#8-常用内置能力)
9. [配置与命令行](#9-配置与命令行)
10. [推荐项目结构与实践](#10-推荐项目结构与实践)
11. [常见问题](#11-常见问题)
12. [小结](#12-小结)

## 1. pytest 是什么

pytest 是 Python 生态中广泛使用的测试框架。它适用于从小型单元测试到集成测试的多种场景，也能运行许多基于 `unittest` 的现有测试。主要特点如下：

- **低门槛**：普通函数配合 Python 原生 `assert` 即可编写测试，无需继承特定测试基类。
- **清晰的失败诊断**：断言失败时展示表达式中的实际值，便于定位问题。
- **Fixture 机制**：复用依赖与资源准备，并管理生命周期和清理。
- **参数化与标记**：扩展输入组合，并按类别选择测试。
- **插件生态**：接入覆盖率、并行执行、浏览器自动化等能力。

pytest 是测试运行与组织工具，不会自动判断产品需求是否覆盖充分。测试质量仍取决于清晰的预期、有效的边界分析和可靠的测试数据。

## 2. 安装与第一个测试

推荐在项目虚拟环境中安装，避免依赖污染系统 Python：

```bash
python -m venv .venv
# Windows PowerShell
.venv\Scripts\Activate.ps1
# macOS / Linux: source .venv/bin/activate
python -m pip install pytest
```

创建 `test_calculator.py`：

```python
def add(a: int, b: int) -> int:
    return a + b


def test_add_two_positive_numbers():
    assert add(2, 3) == 5
```

在项目根目录运行：

```bash
python -m pytest
```

成功时通常会看到 `1 passed`。使用 `python -m pytest` 可明确通过当前 Python 环境执行模块；也可以直接运行 `pytest`。

## 3. 测试发现与命名

默认情况下，pytest 会递归发现符合约定的测试文件、类和函数：

- 文件名通常是 `test_*.py` 或 `*_test.py`。
- 函数名以 `test_` 开头。
- 测试类以 `Test` 开头，且通常不定义 `__init__`。
- 类中的测试方法以 `test_` 开头。

明确且一致的命名让收集行为可预测。可用 `pytest --collect-only` 先检查将运行哪些测试；只运行某个文件、类或函数时，可用节点路径：

```bash
pytest --collect-only
pytest tests/test_user.py
pytest tests/test_user.py::TestUser::test_create_user
```

测试函数名宜描述行为和场景，例如 `test_rejects_expired_token`，而不是 `test_case_03`。若项目采用 `src/` 布局，建议将产品代码作为已安装包导入，并在项目配置中设置测试路径，减少因当前工作目录不同导致的导入差异。

## 4. assert 与失败信息

直接使用 Python 的 `assert` 表达预期：

```python
def test_discounted_price():
    price = 100
    discount = 0.2
    final_price = price * (1 - discount)

    assert final_price == 80
    assert final_price < price
```

失败时 pytest 会重写断言表达式，展示参与比较的值。测试应验证对使用者有意义的行为，而非重复实现被测函数的内部算法。对浮点数，优先使用容差比较：

```python
assert actual == pytest.approx(0.3)
```

## 5. Fixture：组织前置条件和清理

Fixture 是 pytest 的依赖准备机制。测试函数把 fixture 名称写作参数，pytest 会找到对应 fixture 并注入其返回值。Fixture 可以依赖其他 fixture，形成可组合的准备流程。

```python
import pytest


@pytest.fixture
def user_record():
    return {"id": 7, "name": "Lin", "active": True}


def test_user_is_active(user_record):
    assert user_record["active"] is True
```

### Fixture 的作用范围

`scope` 控制同一 fixture 实例复用的生命周期：

| scope | 生命周期 | 常见用途 |
|---|---|---|
| `function`（默认） | 每个测试函数 | 隔离性强的临时数据 |
| `class` | 一个测试类 | 类内共享且安全的状态 |
| `module` | 一个测试模块 | 模块级昂贵准备 |
| `package` | 一个包 | 包内共享资源 |
| `session` | 整次 pytest 会话 | 只读客户端、公共静态资源 |

作用范围越大，创建次数越少，但共享状态和测试耦合的风险越高。只有资源允许安全共享时才扩大范围。

### 使用 yield 清理资源

`yield` 前准备资源，`yield` 后释放。即使测试断言失败，pytest 也会执行已到达的清理部分：

```python
@pytest.fixture
def temp_connection():
    connection = open_connection()
    yield connection
    connection.close()
```

对于多步资源初始化，应让每一步初始化后就有对应清理保障，避免后续步骤失败时留下半初始化资源。Fixture 应保持职责清晰；避免构造执行大量隐式动作的“万能 fixture”。

在 `conftest.py` 中定义的 fixture 可被其目录及子目录中的测试发现，不需要显式导入。建议把共享 fixture 放在合理的目录层级，避免顶层 `conftest.py` 成为难以理解的全局依赖集合。

## 6. 参数化：用一份测试覆盖多组数据

当测试逻辑一致、输入不同时，可用 `@pytest.mark.parametrize`：

```python
import pytest


@pytest.mark.parametrize(
    "text, expected",
    [
        ("hello", 5),
        ("", 0),
        ("你好", 2),
    ],
)
def test_character_count(text, expected):
    assert len(text) == expected
```

每组参数会作为独立用例报告，失败时能直接看到是哪组数据出错。对边界、非法输入、等价类尤其有用。给参数命名，避免把所有组合塞进一个测试函数；组合数量过多时，按风险优先级取舍，防止套件膨胀。也可参数化 fixture，或者为单条参数指定 `pytest.param(..., marks=...)`。

## 7. Marker：给测试分类与筛选

内置 marker 可表达常见执行条件：

```python
import pytest


@pytest.mark.skip(reason="功能尚未支持")
def test_future_feature():
    ...


@pytest.mark.skipif(not is_windows(), reason="仅 Windows 支持")
def test_windows_behavior():
    ...


@pytest.mark.xfail(reason="上游缺陷尚未修复")
def test_known_bug():
    ...
```

项目也可以定义自有标记（例如 `slow`、`integration`），然后按标记筛选：`pytest -m "not slow"`。自定义标记应在配置文件中注册，便于团队理解其意图并避免拼写错误。

标记是分类和选择机制，不应把长期失败测试无限期 `xfail` 或 `skip` 掩盖起来。标注原因，定期清理过期标记。

## 8. 常用内置能力

### 预期异常

用 `pytest.raises` 验证异常类型；必要时进一步检查异常消息：

```python
def test_invalid_age_is_rejected():
    with pytest.raises(ValueError, match="age must be positive"):
        create_profile(age=0)
```

将操作放在 `with` 内部的最小范围，避免其它代码意外触发同一个异常，造成误通过。

### 临时文件与目录

`tmp_path` fixture 为每个测试提供独立临时目录：

```python
def test_export_writes_json(tmp_path):
    output = tmp_path / "report.json"
    export_report(output)

    assert output.exists()
    assert '"status": "ok"' in output.read_text(encoding="utf-8")
```

测试不必在仓库中创建、维护和清理临时文件。

### Monkeypatch

内置 `monkeypatch` fixture 可在测试期间临时替换属性、字典项、环境变量或工作目录，并在测试结束后恢复：

```python
def test_reads_api_key_from_environment(monkeypatch):
    monkeypatch.setenv("API_KEY", "test-key")
    assert load_api_key() == "test-key"
```

优先替换系统边界（时间、网络、环境变量、文件系统入口），避免过度模拟内部细节。

## 9. 配置与命令行

pytest 支持 `pyproject.toml`、`pytest.ini`、`tox.ini` 等配置形式。新项目可把相关设置集中在 `pyproject.toml`：

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = "-ra"
markers = [
  "slow: 运行时间较长的测试",
  "integration: 需要外部服务或多个组件的测试",
]
```

常用命令：

| 命令 | 用途 |
|---|---|
| `pytest` | 运行发现到的测试 |
| `pytest -q` | 简洁输出 |
| `pytest -v` | 显示每个用例名称 |
| `pytest -x` | 第一次失败后停止 |
| `pytest --maxfail=2` | 最多遇到指定失败数后停止 |
| `pytest -k "user and not slow"` | 按名称表达式筛选 |
| `pytest -m integration` | 按 marker 筛选 |
| `pytest --collect-only` | 只收集，不执行 |
| `pytest --durations=10` | 显示最慢的若干用例 |
| `pytest --tb=short` | 使用较短的回溯信息 |

参数可以组合，例如 `pytest -q -m "not slow"`。CI 中建议保留清晰的失败信息；`-x` 适合快速反馈，不一定适合作为唯一的完整回归运行方式。

## 10. 推荐项目结构与实践

```text
my_project/
├── pyproject.toml
├── src/
│   └── my_project/
│       └── calculator.py
└── tests/
    ├── conftest.py
    ├── unit/
    │   └── test_calculator.py
    └── integration/
        └── test_api.py
```

实用原则：

1. **先测可观察行为**：输入、输出、状态变化和错误处理比私有实现细节稳定。
2. **每个测试可独立运行**：避免依赖执行顺序或共享可变数据。
3. **失败要可诊断**：测试名描述场景，断言定位明确，测试数据具有代表性。
4. **管理外部依赖**：将网络、数据库、时间等边界隔离；集成测试使用可控环境。
5. **合理分层**：快速单元测试、较慢集成测试分开组织，并按需要选择运行集合。
6. **把运行方式自动化**：在持续集成中安装项目依赖并运行测试，失败时保留日志。
7. **插件按需添加**：插件能扩展能力，也会带来依赖和维护成本；选择有明确价值的插件。

## 11. 常见问题

### 为什么没有发现测试？

检查文件名、函数名是否符合约定，确认运行目录和配置中的 `testpaths`，然后执行 `pytest --collect-only -q` 查看收集结果。

### 为什么本地能导入，CI 却报模块找不到？

常见原因是本地工作目录或 `PYTHONPATH` 偶然提供了导入路径。将项目按规范安装到虚拟环境，统一 CI 工作目录和安装步骤，并检查 `src/` 布局及配置。

### 为什么测试之间互相影响？

通常是共享可变状态、数据库记录未清理、临时文件重名、环境变量未恢复，或测试依赖运行顺序。使用 function-scope fixture、独立数据标识和可靠清理来修复根因。

### 单元测试还是端到端测试？

两者解决的问题不同。单元测试反馈快、定位明确；集成或端到端测试验证更多真实组件的协作，但运行成本和环境复杂度更高。应按风险分层组合，而不是期待一种测试覆盖全部风险。

## 12. 小结

pytest 从简单的 `assert` 起步，借助 Fixture 复用准备逻辑，使用参数化覆盖输入空间，再用标记和配置组织不同运行集合。框架提供结构与反馈，可靠的测试仍来自独立、明确、贴近用户行为的验证。

### 官方资料

- [pytest 官方文档：Get Started](https://docs.pytest.org/en/stable/getting-started.html)
- [pytest 官方文档：Fixtures](https://docs.pytest.org/en/stable/explanation/fixtures.html)
- [pytest 官方文档：参数化](https://docs.pytest.org/en/stable/how-to/parametrize.html)
- [pytest 官方文档：命令行用法](https://docs.pytest.org/en/stable/how-to/usage.html)
- [pytest 官方文档：配置选项](https://docs.pytest.org/en/stable/reference/customize.html)
