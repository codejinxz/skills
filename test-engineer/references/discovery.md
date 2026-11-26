# 测试资产探测手册（摸底与证据源）

> 供主会话在「摸底」一步使用，也为各维度指明证据去哪找。全部手段**只读**：列目录、搜内容、读文本、读报告产物、看 git 历史；不执行测试、不装依赖、不构建。
> 目标项目可能只有其中一部分资产（例如无 CI、无覆盖率配置），缺失本身就是发现——按「缺口审计（维度 11）」「是否真执行过（维度 31）」处理，而不是跳过。

## 〇、多子项目与重复副本仓库（摸底最先做）

仓库里常有同一系统的多份副本、纯文档目录、无版本控制的子目录——不先理清，资产统计和发现计账都会翻倍。摸底第一步：

1. **子项目清点**：按"能否独立运行/构建"划分子项目；纯文档目录（论文、用例图、设计稿）计入画像但单列一行标注"无代码"——**没有可测对象，其零测试不构成缺口**。
2. **副本识别**：目录名相近、相同包名/groupId、关键文件 diff 仅数行差异 → 判定为同系统副本。
3. **权威副本判定**：有版本控制的优先于没有；都有或都没有时，选测试资产更完整的一份，并在画像与报告中注明副本关系。
4. **去重计账**：同一文件在多副本各有一份 → **测试资产只计一次**（按权威副本），发现按唯一文件报告，副本间的差异（如一处注解被注释、另一处没有）作注记——**差异本身可能正是发现（"副本漂移"），默认 P2~P3；差异对象为测试代码时归维度 7（测试与实现失同步的变体），文档失同步归维度 11**。**副本独有的测试资产**（权威副本没有的测试文件）：计入画像注记行、照常审计，发现归真实维度——它们是真实存在的测试，只是不在权威副本里；不参与"去重"（没有对应物可去）。
5. **对照源选择**：API 文档等契约对照源**取权威副本所在仓的那份**；多副本文档失同步本身记为文档失同步类发现。
6. **纪律核查的 fallback**：`git status --ignored` 只对 git 仓库可执行；非 git 子项目改用**审计前后的文件清单/mtime 对比**核查"未改动"，或在报告注明"该子项目无版本控制，改动核查以文件比对为准"。

## 一、五类资产的探测清单

### 1. 测试代码

> **统计口径**："测试文件数/行数"只计**包含测试用例定义**的文件（`test(`/`it(`/`@Test` 等所在文件）；测试框架配置（`vitest.config.*`、`tsconfig.vitest.*`、`jest.config.*`）**不计入**文件数，但"配置存在而零测试实现"要单独记录——它是维度 11（声明与现实的差集）的直接证据。

| 栈 | 常见位置 | 识别特征 |
|---|---|---|
| 通用 | `test/` `tests/` `spec/` `__tests__/` `src/test/` `*_test.go` `*.test.ts` `*.spec.ts` `test_*.py` `*_test.py` `*Test.java` `*Spec.kt` `*_spec.rb` | 与被测源码同构的目录镜像 |
| 分层线索 | `unit/` `integration/` `e2e/` `contract/` `acceptance/` `smoke/` `perf/` `load/` | 目录名或 tag/pytest marker/JUnit tag |
| 辅助资产 | `fixtures/` `__fixtures__/` `testdata/` `mocks/` `stubs/` `conftest.py` `setup.ts` `test-utils/` | 决定维度 15、13 的证据 |

统计：测试文件数、总行数、各层级数量。**测试代码行数与被测代码行数的比值**可作为覆盖密度的粗信号（不与覆盖率混为一谈，覆盖率只看报告）。

### 2. 测试策略与计划文档

`TESTING.md`、`docs/testing*`、`docs/测试*`、`docs/qa*`、`CONTRIBUTING.md` 的测试章节、`README` 的 Test 章节、`docs/adr/` 中涉及测试决策的记录、DoD/提测清单。

摘录要点：声明的测试层级、声明的覆盖目标、声明的必测模块、声明的执行时机、声明的 flaky 处置流程。**这些是维度 11「缺口审计」和维度 19「缺失层级审计」的对照源。**

### 3. 测试配置

| 用途 | 常见文件 |
|---|---|
| 框架配置 | `jest.config.*` `vitest.config.*` `playwright.config.*` `cypress.config.*` `pytest.ini` `pyproject.toml [tool.pytest]` `tox.ini` `phpunit.xml` `.rspec` `karma.conf.js` `build.gradle` 的 test 段 |
| 覆盖率配置 | `.coveragerc` `coverage.config` `pyproject [tool.coverage]` `jest coverageThreshold` `codecov.yml` `.nycrc` `jacoco` 配置 `sonar-project.properties` |
| 测试环境 | `.env.test` `docker-compose.test.yml` `testcontainers` 配置 `Makefile` 的 test target |
| 门禁配置 | CI 配置中的 test job、branch protection 描述、`--passWithNoTests`、`continue-on-error`、`\|\| true`、`allow_failure` |

**重点扫描**：覆盖率阈值是否配置、阈值是否被 `continue-on-error` 或 `|| true` 抵消、`--passWithNoTests`、排除项（`omit`/`exclude`/`ignore`）的范围。

### 4. CI 与执行记录

`.github/workflows/*.yml`、`.gitlab-ci.yml`、`Jenkinsfile`、`azure-pipelines.yml`、`.circleci/config.yml`、`.drone.yml`、`bitbucket-pipelines.yml`、`buildkite`。

逐项确认：
- 测试是否在流水线中有独立 job，触发条件是什么（PR / push / 定时 / 手动）
- job 是否 `allow_failure` / `continue-on-error` / 被注释掉 / 有 `if: false`
- 是否有 flaky 重跑逻辑（`retry`、`--rerun-failed`、`pytest-rerunfailures`），重跑是否有标记
- 是否有分片与并行（`--shard`、`parallel`、`matrix`、`pytest-xdist`）
- 是否有超时设置与缓存
- 是否有历史执行结果文件被提交进仓库（如 `test-results/`、`.ci-status`）

CI 平台上的实际运行历史（绿/红、耗时趋势）通常**拿不到**——这类只能放「待核实」，不得当作"测试未执行"的结论，也不得当作"已通过"的证据。

### 5. 覆盖率与报告产物

| 产物 | 文件 | 能回答什么 |
|---|---|---|
| 单元测试报告 | `surefire-reports/` `test-results/` `junit.xml` `*.trx` | 用例级结果、耗时、skip 数、失败原因（维度 30、31） |
| 覆盖率 | `coverage/lcov.info` `coverage-final.json` `coverage.xml` `jacoco.xml` `.coverage` `htmlcov/` `clover.xml` | 行/分支/函数口径、排除项、文件级缺口（维度 9、33） |
| E2E 产物 | `playwright-report/` `cypress/videos` `cypress/screenshots` `allure-results/` | 是否真跑过、失败现场（维度 31、6） |
| 性能产物 | `k6` 摘要、`jmeter` 结果、`locust` 报告、benchmark 输出 | 基线、阈值、趋势（维度 22） |
| 安全产物 | `sast`/`dependabot`/`audit` 报告、`zap` 报告 | 扫描接入与处置（维度 23） |

**可信度判定（维度 33）**：读覆盖率报告时确认口径字段（`lines`/`branches`/`functions`）、生成时间、被排除的文件清单。只有行覆盖而没有分支覆盖数据时，不得据此判定"分支已覆盖"。

## 二、跨栈等价换算对照表

| 概念 | JVM | Python | JS/TS | Go | Rust | Ruby | PHP | .NET |
|---|---|---|---|---|---|---|---|---|
| 测试框架 | JUnit / TestNG / Spock | pytest / unittest | Jest / Vitest / Mocha | testing / testify | built-in `#[test]` | RSpec / Minitest | PHPUnit / Pest | xUnit / NUnit |
| 参数化 | `@ParameterizedTest` | `@pytest.mark.parametrize` | `test.each` / `it.each` | 表驱动 + `t.Run` | 循环 + 子测试 | `shared_examples` | data provider | `[Theory]` |
| skip 语法 | `@Disabled` `@Ignore` | `@pytest.mark.skip` `skipif` | `it.skip` `test.todo` | `t.Skip` | `#[ignore]` | `xit` `pending` | `markTestSkipped` | `[Fact(Skip=...)]` |
| mock 边界 | Mockito / MockK | `unittest.mock` / `pytest-mock` | `jest.mock` / `vi.mock` / msw | 接口 fake / `gomock` | `mockall` | `allow(...).to receive` | `createMock` | Moq / NSubstitute |
| 容器化依赖 | Testcontainers | testcontainers-python | testcontainers-node | testcontainers-go | testcontainers-rs | testcontainers-ruby | testcontainers-php | Testcontainers.NET |
| 时钟控制 | `Clock` 注入 / `@Clock` | `freezegun` / `time-machine` | `jest.useFakeTimers` | 注入 `now func()` | 注入 clock | `travel_to` | `ClockInterface` | `TimeProvider` |
| 覆盖率工具 | JaCoCo | coverage.py | istanbul / c8 / v8 | `-cover` | `tarpaulin` / `llvm-cov` | SimpleCov | PHPUnit coverage | coverlet |
| 断言调用形态（统计断言数按本行搜） | `assertEquals`/`assertTrue`/`assertThat`/MockMvc 的 `.andExpect(...)` | `assert` 语句 / `assertEqual` | `expect(...)`/`assert.equal` | `t.Errorf`/`t.Fatal`/`assert.*` | `assert_eq!`/`assert!`/`should` | `expect(...).to` | `assertEquals`/`assertThat` | `Assert.*`/`Should().*` |
| 分层标签 | `@Tag` | `markers` | `describe` / project | build tag | `#[cfg(test)]` | `type: :request` | `@group` | `[Trait]` |
| 报告产物 | surefire / jacoco.xml | junit-xml / .coverage | junit / lcov.info | `-json` | junit | junit | junit / clover | trx / cobertura |

「单元测试用 E2E 实现」「跨服务集成用全 mock 单测」这类错配（维度 18），在各栈的表现形式不同但判定逻辑一致：看这条测试**实际启动了什么**，而不是看它在哪个目录里。

> **警告：统计断言数禁止只 grep "assert" 前缀**。换算表"断言调用形态"行是各栈的完整形态——JVM 链式断言、MockMvc 的 `.andExpect()`、JS 的 `expect()`、Go 的 `t.Errorf` 都不含 "assert" 字样，只按前缀统计会大量漏计，导致把健康项目误报为"零断言"。统计前先按该行确认本栈的全部断言形态，必要时抽查人工复核。

## 三、维度 → 证据源对照

> 派发子代理时，连同本表对应行一起摘入指令，避免子代理在自己不熟悉的栈上乱找。

| 维度 | 首选证据 |
|---|---|
| 1 断言强度 | 测试函数体的断言语句；搜索 `assert True`、`expect(true)`、`assertNotNull` 后无后续断言、`assertDoesNotThrow` 单一断言 |
| 2 mock 边界 | mock/stub/spy 声明处 vs 被测函数体；`jest.mock('被测模块')`、`patch('被测模块.函数')` |
| 3 独立可重复 | 全局 `beforeAll`/`setUpClass` 写共享状态；测试内改环境变量；测试顺序依赖（后测依赖前测造的数据） |
| 4 flaky | `sleep`/`time.sleep`/`setTimeout`/`Thread.sleep` 出现处；重试注解；`flaky` 清单文件；issue 索引 |
| 5 同构复读 | 测试内重算一遍实现公式；参数化数据直接引用实现常量 |
| 6 失败诊断 | 断言库的 message 参数是否为空；参数化用例 id；自定义断言封装是否丢上下文 |
| 7 代码质量 | 测试文件重复块、魔法数、单用例行数、helper 定义与调用点 |
| 8 需求映射 | 测试名/注释中的需求编号；`docs/` 需求文档条目 vs 测试搜索命中 |
| 9 分支异常 | 覆盖率报告的分支/行数据；错误码常量在测试中的出现次数 vs 定义次数 |
| 10 契约覆盖 | OpenAPI/proto/GraphQL schema 的端点清单 vs 测试中的路径字符串 |
| 11 缺口审计 | 策略文档声明项逐条 grep 验证存在性；配置中启用但无产物的检查。**声明项查无时，用 `git log --all --diff-filter=D -- <path>` 反查是否曾存在后被删除——"写过又删且文档未同步"与"从未写过"严重度不同**，前者另记一条文档失同步 |
| 12 数据代表性 | fixture 数据文件、测试内构造的数据（是否只有 happy path 值、边界值缺失） |
| 13 数据合规 | fixture/`testdata/` 中的邮箱、手机号、身份证、token、私钥、真实域名；`.env.test` 是否含真凭据 |
| 14 等价类边界 | 参数化表的数据分布；是否只有一组值；边界值（0/负数/MAX/空/超长）出现处 |
| 15 fixture 隔离 | fixture scope（session/class/function）；DB 清理策略；临时目录唯一性；teardown 是否有 try/finally 或 yield 后清理 |
| 16 时间随机 | `Date.now()`/`datetime.now()`/`time.Now()`/`rand()` 在测试中的裸用；时区硬编码；本机绝对路径 |
| 17 命名组织 | 测试文件路径结构 vs 源码结构；命名模式一致性抽样 |
| 18 层级匹配 | 测试内是否起容器/起浏览器/连真实库；单测目录里是否有 E2E 级操作 |
| 19 缺失层级 | 各层目录是否存在与数量；CI 中跑了哪些层；策略文档声明的层 |
| 20 回归集 | CI 中的回归 job 定义；`affected`/`--changed` 配置；缺陷修复提交是否带测试变更 |
| 21 冒烟集 | `smoke` 目录/tag；PR 触发的 job；阻断配置 |
| 22 性能 | `perf/`、`load/`、benchmark 文件、k6/jmeter 脚本、阈值断言 |
| 23 安全 | 鉴权矩阵测试（未登录/低权/高权 × 端点）；注入类用例；SAST/DAST/audit 配置 |
| 24 可靠性 | 超时/重试/熔断/幂等的测试；故障注入脚本；重复请求用例 |
| 25 兼容性 | CI matrix；多版本测试配置；字符集/时区用例 |
| 26 可访问性 | `axe`/`pa11y`/`lighthouse` 依赖与配置；E2E 中的 a11y 断言 |
| 27 CI 集成 | workflow 文件；job 阻断配置；超时设置；执行时长记录 |
| 28 并行分片 | `--shard`/`parallel`/`xdist`/`-n` 配置；共享资源的并发冲突迹象（固定端口、固定库名） |
| 29 环境依赖 | 测试启动脚本；`docker-compose.test.yml`；硬编码 host/port/path；local vs CI 差异 |
| 30 报告趋势 | 报告产物是否归档；是否有历史目录；报告是否含用例级明细 |
| 31 真执行过 | 报告产物时间戳 vs `git log` 最近提交时间；skip 清单；长期无变更的产物 |
| 32 通过记录 | CI 配置的触发与阻断；产物中的版本/commit 字段；绕过门禁的痕迹（`[skip ci]`、force merge 记录） |
| 33 覆盖率可信 | 覆盖率配置的阈值与排除项；报告口径字段；是否有变异测试/断言覆盖工具 |
| 34 缺陷闭环 | 缺陷编号在测试中的引用；修复提交是否含测试；失败处置流程文档；测试债务登记 |

## 四、只读核验的边界

**允许**（不产生副作用）：

- 列目录、按模式找文件、统计文件数与行数
- 全文搜索、读取任意文本文件（含 JSON/XML/HTML 报告）
- `git log` / `git show` / `git diff` / `git status` / `git status --ignored` / `git blame`（只读；`--ignored` 用于发现被 gitignore 的产物目录，如 `coverage/`）
- 读 CI 配置文件、读覆盖率报告、读测试报告产物
- 读 `package.json` / `pom.xml` / `build.gradle` / `pyproject.toml` 中的测试相关声明

**禁止**（会让评审结论失真，且违反"不执行测试"）：

- 运行任何测试命令：`npm test`、`pnpm test`、`yarn test`、`pytest`、`tox`、`nox`、`mvn test`、`gradle test`、`go test`、`cargo test`、`dotnet test`、`rspec`、`phpunit`、`playwright test`、`cypress run`
- 带覆盖率开关的重跑（`--coverage`、`-cover`、`jacocoTestReport`）
- 安装依赖（`npm install`、`pip install`、`bundle install`）、启动容器或服务、构建
- 任何写操作：改配置、生成快照、`--updateSnapshot`、`--fix`

**禁止（续）：测试枚举命令。** `pytest --collect-only` 会 **import 测试模块**，触发模块级代码——建连接、发请求、写临时文件、改全局状态都可能发生，既破坏只读，结果也不可信。`go test -list`、`rspec --dry-run` 同理。`jest --listTests` 虽然只读文件系统，但为纪律一致一并禁用。

**替代做法**：用文件匹配按命名约定自己列（`**/*test*`、`**/*_test.py`、`**/*.spec.ts`、`**/*Test.java`），准确度足够且零副作用。这条**没有"仅在……时可用"的例外**——收益（一份文件清单）远小于风险。

若用户明确要求"跑一遍测试看看"，那超出本 skill 范围：先说清这里只做只读审计，再另行确认是否单独执行；执行结果不得回写进本次报告充当证据。

## 五、变更集 → 测试文件的映射

"只审最近改动"模式的核心动作：从代码变更反查"该由哪些测试兜住"。三条路径取并集：

**1. 路径匹配**：变更文件 `src/payment/refund.ts` → 找 `test/payment/refund.test.ts`、`tests/payment/test_refund.py`、同目录下 `*_test.go`。同构目录的项目这一步命中率最高。

**2. 命名匹配**：取变更文件的主类名/主函数名（`RefundService`、`refund_order`），搜测试名与 `describe`/`class` 声明里出现该名字的地方。适用于测试不与源码同构的项目。

**3. 引用反查**：搜测试文件里 import/require 了该模块路径的地方（`from 'src/payment/refund'`、`import com.x.payment.RefundService`）。适用于测试按功能而非按路径组织、或一个测试覆盖多个模块的情况。

**结果怎么用**：

- 三条路径都为空 → **这是最重要的发现，不要跳过**。意味着这次改动没有任何测试与之对应，直接进缺口审计（维度 11）与假绿清单的前置检查。
- 只命中 E2E 或只命中单测 → 记下层级分布，供维度 18（层级匹配）判断。
- 命中多个但都是薄测试 → 转维度 1、2 深看。

**命令**（只读）：
```bash
git log --oneline -20
git diff --name-only <base>..HEAD
git show --stat <commit>
```
变更集模式下要在报告头部写明 base 与 HEAD，否则读者无法复现"审的是哪一批改动"。

## 六、证据可得性与缺失处理

本 skill 不执行测试，所以一部分维度的结论依赖**已有产物**。拿到什么决定能说什么，拿不到时不许硬判。

| 需要的证据 | 通常在哪 | 拿不到时怎么办 |
|---|---|---|
| CI 平台历史（通过率、时长趋势、是否真拦过） | CI 平台网页、API | 维度 32 只能评"配置是否具备拦截能力"，**不能评"是否真的拦过"**；进未决项 |
| 测试报告产物（junit xml、`test-results/`） | 仓库内、CI artifact | 维度 31 无法确认"真执行过"，只能评"是否存在可执行痕迹"；进未决项 |
| 覆盖率产物（`lcov.info`、`coverage.xml`） | 仓库内、CI artifact、覆盖率服务 | **三态：从未配置覆盖率 → 维度 33 判"有发现，定 P1"**（零测量本身即失守）；**已配置但残缺（无阈值无门禁，或代码已变更而产物未更新——产物 mtime 早于最后一次触及被测路径的提交；代码未变时产物与代码对应，不算过期）→ 判"有发现，定 P2"**；**已配置且产物缺失 → 判"证据不可得"**（数字无法核真假），只能评"配置与口径是否合理"。与 dimensions.md/SKILL 口径一致，勿漂移。维度 9 退化为纯代码推断——**有独立替代证据源（契约文档 ↔ 测试搜索差集）时不失效** |
| 缺陷 / issue 系统 | 外部系统 | 维度 34 只能评"测试里是否引用了缺陷号"，不能评"每个缺陷是否回补了用例" |
| 生产事故记录 | 外部系统 | 同上，进未决项 |

**关键区分：一维失效不等于"未发现"。**

- **未发现** = 查过了，没找到问题。这是结论。
- **证据不可得** = 没法查。这不是结论，是缺口，必须单列并计入恒等式的第四项。

把"没法查"写成"未发现"，等于用沉默冒充清白，是本 skill 最需要防的一种失真。
