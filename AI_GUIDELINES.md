# AI_GUIDELINES.md
> 本文件用于指导 AI（如 Codex / Cursor / ChatGPT）在本项目中生成、修改和重构 **Java 代码** 时遵守统一规范。  
> 项目架构：**Spring Boot + MyBatis + Oracle**  
> 若历史代码与本规范冲突，以本规范为准。

---

## 0. 关键指令（AI 必须遵守）

1. **导包规范**
    - 禁止在代码体中使用全限定类名（FQCN）。
    - 必须在类文件头部 `import` 所需类型，就像在 IntelliJ IDEA 中编写代码那样。
    - 自动移除未使用的 `import`，禁止 `import xxx.*;`。

2. **工具类优先级**
    1. 优先使用 **Apache Commons** 工具类（`commons-lang3`, `commons-collections4`, `commons-io`, `commons-codec` 等）；
    2. 若没有合适方法，再考虑 **Guava**, **Hutools**；
    3. 若依旧没有，再使用项目自定义工具类（通常在 `com.juneyaoair.**.utils` 包下）；
    4. 禁止重复造轮子，已有工具方法不要重新实现。

3. **三层架构约束**
    - **Controller 层**：仅负责接收参数、调用 Service 并返回结果。不得写业务逻辑。
    - **Service 层**：承载业务逻辑、事务控制，聚合数据操作。
    - **Repository / Mapper 层**：负责数据库访问和 SQL 定义，不编写业务判断。
    - **工具类**：仅包含通用静态方法，不含任何业务逻辑。
    - 所有生成代码必须符合 Controller → Service → Mapper 的调用方向。

4. **代码风格与格式化**
    - 使用四个空格缩进，UTF-8 编码。
    - 每个公共类、方法、接口必须有 Javadoc 注释。
    - 所有新类应带有作者、创建日期、功能说明。
    - 生成代码前后，AI 应自动优化导入并保持格式整洁。


---

## 1. 工具类与依赖优先级

| 优先级 | 库 | 常用类示例 |
|---------|----|-------------|
| 🥇 Apache Commons | `StringUtils`, `ObjectUtils`, `CollectionUtils`, `MapUtils`, `DateUtils`, `IOUtils`, `DigestUtils` |
| 🥈 Google Guava | `Preconditions`, `ImmutableList`, `Splitter`, `Joiner`, `Strings` |
| 🥉 项目自定义工具类 | 位于 `com.juneyaoair.**.utils` 包下，如 `DateUtil`, `JsonUtil`, `HttpUtil` |

**正确示例 ✅**
```java
import org.apache.commons.lang3.StringUtils;

if (StringUtils.isBlank(username)) {
    throw new IllegalArgumentException("username cannot be blank");
}
```

**错误示例 ❌**
```java
if (username == null || username.trim().isEmpty()) {
    // 手写判断，不规范
}
```

## 2. 包结构与命名规范

com.juneyaoair.jxac
├─ controller      // 表现层：参数接收、校验、调用 Service
├─ service
│   ├─ impl        // 业务逻辑实现类
│   └─ dto         // 请求/响应 DTO
├─ mapper      // MyBatis Mapper 接口或 Repository 类
├─ domain          // 领域模型、实体对象（Entity / BO）
├─ converter       // MapStruct 转换器（DTO ↔ Entity）
├─ util            // 通用工具类（无业务逻辑）
├─ config          // 配置类（Spring 配置、拦截器等）
├─ common          // 公共常量、异常、统一返回体
└─ Application.java


### 命名规范
| 类型 | 示例 | 说明 |
|------|------|------|
| 控制器 | `UserController` | 使用复数资源名或业务语义 |
| 服务接口 | `UserService` | 对应实现类 `UserServiceImpl` |
| Mapper | `UserMapper` | 仅负责数据库操作 |
| DTO | `UserCreateDTO`, `UserQueryDTO` | 数据传输对象 |
| VO | `UserVO` | 前端展示对象 |
| 工具类 | `DateUtil`, `ExcelUtil` | 仅包含静态方法 |

### 规则说明
- Controller 层命名以 `Controller` 结尾，对应业务模块；
- Service 层命名以 `Service` 结尾，实现类以 `ServiceImpl` 结尾；
- Mapper 层命名以 `Mapper` 结尾，禁止写业务逻辑；
- DTO、VO 用于数据传输与展示，命名要清晰表达用途；
- 工具类命名以 `Util` 结尾，禁止出现 `Manager`、`Logic` 等业务含义；
- 包名一律使用小写单词，不使用大写、下划线或连字符；
- 每个包的职责单一，避免在同一包中混放 Controller、Service、Mapper 等文件。

### 2.1 数据库规范（同一厂商，多环境差异）

- 本项目数据库厂商是**固定的**（例如 MySQL 或 Oracle），但不同环境（开发、测试、预发布、生产）之间可能存在**版本或配置差异**。
- **AI 生成 SQL 时的基本要求：**
    1. **尽量少用数据库特定函数**，如 MySQL 的 `DATE_FORMAT()`、Oracle 的 `TO_CHAR()` 等；
    2. **优先使用标准 SQL 语法**（如 `COALESCE()`、`CASE WHEN`），确保跨版本兼容；
    3. **不要在 SQL 中做时间格式化或字符串拼接**，这些操作应放在 Java 代码中（Service 层或 Converter 层）；
    4. **分页、排序逻辑统一在应用层处理**，Mapper XML 中只写基础 SQL；
    5. **时间字段统一使用 Java 8 时间类**（`LocalDateTime`、`LocalDate`、`LocalTime`），由 TypeHandler 负责数据库映射；
    6. 如果确需使用数据库函数（如日期差或字符串函数），请选用**兼容性好的写法**，并在注释中说明原因；
    7. 禁止在 XML 中根据环境版本写 `if test="_databaseId == 'mysql'"` 之类的方言判断；
    8. 所有数据库连接与行为差异（如时区、字符集）均通过配置文件（`application-*.yml`）解决，不通过 SQL 语法规避。

**正确示例 ✅**
```sql
SELECT COALESCE(name, '') AS user_name
FROM user_info
WHERE create_time >= #{startTime} AND create_time < #{endTime}  
```

### 2.2 时间字段与 Oracle 旧版本兼容说明

- **默认规范**  
  项目中时间字段统一使用 **Java 8 时间类**：
    - `LocalDateTime` ↔ 数据库 `TIMESTAMP`
    - `LocalDate` ↔ 数据库 `DATE`
    - `LocalTime` ↔ 数据库 `TIME`（若支持）
    - 映射交由 **MyBatis TypeHandler** 处理，不直接在 SQL 层格式化时间。

- **兼容性考虑（针对旧版 Oracle / 驱动）**
    - 若 Oracle 版本较低（如 11g/早期 12c）或驱动版本过旧（如 ojdbc6.jar），`LocalDateTime` 映射可能失败；
    - 此时应在实体层改用 `java.sql.Timestamp`;
    - 若项目确实受限于旧版 Oracle，可在注释中明确标注：
      > “当前项目因 Oracle 驱动限制，暂使用 java.sql.Timestamp，后续升级 ojdbc8 后再切回 LocalDateTime。”

- **统一规范**
    - 所有时间计算（加减、格式化、区间判断）均在 Java 层完成；
    - **禁止**在 SQL 中使用 `TO_CHAR()`、`TO_DATE()` 等函数；
    - 若历史 SQL 无法立即改造，须在注释中标明原因及迁移计划。



## 3. 代码风格

- 统一编码：UTF-8；
- 每行长度 ≤ 120 字符；
- 类、方法、变量命名遵守驼峰命名法；
- 常量名使用大写下划线（`UPPER_SNAKE_CASE`）；
- 日志使用 `Slf4j`（推荐 Lombok `@Slf4j` 注解），**禁止使用 `System.out.println`**；
- 对象创建优先使用构造器注入，禁止字段注入；
- 空值判断优先使用 `Objects.nonNull()`、`CollectionUtils.isEmpty()`；
- 异常抛出应包含上下文信息，避免仅抛出 `new RuntimeException(e)`；
- 单个方法长度不超过 80 行，职责单一；
- 注释：
    - 复杂逻辑必须添加注释；
    - 公共方法和接口必须写 Javadoc（含入参、返回值、异常说明）；
    - TODO 需说明责任人及计划完成时间；
- 所有新增代码应通过IDEA 格式化；
- 文件末尾需保留一个换行符；
- 统一使用四个空格缩进，不允许 Tab 字符；
- 在返回对象时，建议使用统一封装类（如 `Result<T>`）；
- 新增类需包含作者、日期与功能描述的 Javadoc；
- 在事务方法上使用 `@Transactional(rollbackFor = Exception.class)`；
- 使用 `Optional` 时，仅用于返回值，不作为类成员变量；
- Mapper XML 统一缩进、关键词大写（SELECT / INSERT / UPDATE / DELETE）。
- **日期时间处理规范**：
    - 优先使用 **Java 8+ 时间 API**：`LocalDate`、`LocalDateTime`、`LocalTime`、`Duration`、`Period`；
    - 禁止再使用过时类：`Date`、`Calendar`、`SimpleDateFormat`；
    - 若与数据库交互需转换，请在 `Converter` 层或 `MyBatis TypeHandler` 中统一处理；
    - 格式化请使用 `DateTimeFormatter`，避免线程安全问题。

---

## 4. 禁止行为（AI 生成时不得出现）

以下行为在自动生成或人工提交的代码中 **严禁出现**：

1. **使用全限定类名（FQCN）**
    - ❌ `private java.util.List<User> users;`
    - ✅ `import java.util.List; private List<User> users;`

2. **在 Controller 中编写业务逻辑或数据库操作**
    - Controller 只能调用 Service，不得直接调用 Mapper。

3. **在工具类中包含业务逻辑或数据库访问**
    - 工具类仅能包含无状态的通用静态方法。

4. **捕获异常后不处理**
    - 禁止空 catch 块或仅打印堆栈而不记录日志。
    - ✅ `log.error("Failed to create order", e); throw new BizException("创建订单失败", e);`

5. **使用 `System.out.println` 输出日志**
    - 使用标准日志框架，如 `Slf4j`。

6. **直接 new Service / Mapper / Component 实例**
    - 必须使用 Spring 容器注入（构造器或 `@Autowired`）。

7. **使用通配 import**
    - ❌ `import java.util.*;`
    - ✅ `import java.util.List; import java.util.Map;`

8. **SQL 语句中出现业务逻辑判断**
    - Mapper 层只负责数据访问，不应包含复杂逻辑或流程判断。

9. **未捕获的受检异常直接上抛**
    - 所有受检异常应转换为项目自定义异常（如 `BizException`）。

10. **违反三层架构原则**
    - Controller → Service → Mapper 的调用链不可逆。
    - 禁止跨层调用（如 Service 调用 Controller）。

11. **在 Service 方法中直接拼接 SQL 字符串**
    - 所有 SQL 操作必须在 Mapper 层定义。

12. **未关闭资源（如 IO 流、数据库连接）**
    - 推荐使用 try-with-resources 自动关闭。

13. **在事务方法中执行非幂等的外部调用（如 HTTP 调用）**
    - 应放在事务提交后或异步任务中执行。

14. **硬编码路径、配置、密钥**
    - 所有环境变量必须通过配置文件或环境变量注入。

15. **滥用静态变量存储上下文状态**
    - 禁止在静态变量中保存请求或用户信息。

---

**示例（反例）❌**
```java
@RestController
public class OrderController {
    @Autowired
    private OrderMapper orderMapper;

    @PostMapping("/create")
    public void createOrder(String name) {
        // ❌ Controller 编写业务逻辑
        if (name == null || name.trim().isEmpty()) {
            System.out.println("订单名为空"); // ❌ 不允许使用 System.out
            return;
        }
        orderMapper.insertOrder(name); // ❌ 直接操作数据库
    }
}
```

## 5. 回复语言规范

- **所有回复内容必须使用中文**（简体中文），包括代码说明、注释、日志信息和生成的文档。
- 若输出中包含英文关键字（如类名、方法名、变量名），可保持英文，但解释性内容必须为中文。
- 生成的注释（包括 Javadoc、行内注释、XML 注释）也应以中文撰写。
- 若外部库或依赖名称为英文（如 `Apache Commons`、`MyBatis`、`Spring Boot`），保留英文原文，但说明部分仍需中文解释。
- 对用户的任何问题、总结、建议或指令反馈均应使用中文回答，不得混用英文句式或语气。

## 6. 协作与修改确认规范

- **禁止在未沟通确认前修改任何现有代码。**
    - 在对已有文件、类、函数进行修改前，AI 必须先与我确认修改意图和影响；
    - 修改包括但不限于：调整函数逻辑、删除旧代码、重命名变量、变更接口定义、修改数据库操作、改动依赖配置；
    - 在我明确回复“可以修改”或“确认无影响”之前，AI 只能：
        - 分析代码；
        - 提出修改建议；
        - 给出示例实现方案；
        - 或以伪代码形式展示修改思路。

- **所有修改必须先说明理由**：
    - 说明修改目标（例如“优化性能”“修复空指针”“统一命名规范”）；
    - 说明潜在影响（例如“可能影响接口返回结构”“可能导致数据回写”）；
    - 再等待我确认后再执行实际修改。

- **严禁自动改写未确认的文件**。若需要批量修改多个文件，请在确认范围后再执行。

**示例 ✅**
> 🧠 AI：我建议优化 `AttendanceService` 的 `calculateDailyHours()` 方法，改用 `Stream API` 替换 for 循环，逻辑保持一致。是否可以修改？

**反例 ❌**
> 🔴 AI 直接修改代码并保存到文件，未征得我同意。


## **7. 注释与需求对应规范**

> 适用场景：当代码直接源于产品需求描述、会议沟通、或业务讨论时，需要说明“这段代码为了解决什么需求”。

### **7.1 基本要求**

* 注释必须**用中文**，自然语言表达；
* 注释需明确说明：该段代码对应哪一条“需求描述”；
* 适用于外包交付、产品验收、需求追溯；
* 注释不要引用 GP-xxxx / AI-xxxx 等自动编号；
* 注释描述应简洁明了，避免写长篇产品需求原文。

### **7.2 行级需求映射注释示例**

```java
// 导出功能需支持分页与时间过滤
List<Record> result = recordService.listPaged(start, end, pageSize);
```

### **7.3 方法级需求映射注释**

```java
/**
 * 导出打卡记录（分页模式）
 */
public byte[] exportPaged(ExportDTO dto) { ... }
```

### **7.4 注意事项**

* 无需求关联的代码不需要写需求映射，只需写业务注释；
* 对外交付、外包验收类代码必须有需求映射注释；
* 若需求变更，注释必须同步更新；
* 出现任何自动标签（如 `[GP-01]` / `[AUTO-*]`）视为不合规范。

---

## **8. 通用注释规范（业务导向）**

> 适用场景：提升代码可读性，让开发与非开发人员能理解业务流程，而无需读取所有代码细节。

---

### **8.1 总体原则**

* 注释必须解释“做什么”“为什么”；
* 不解释语法，只解释业务逻辑；
* 必须用中文；
* 注释要帮助开发者和非技术人员快速理解逻辑。

---

### **8.2 模块（类）级注释（可选）**

仅在以下情况添加：

* 模块边界清晰，功能独立；
* 需要对外交付（外包、第三方审计）；
* 业务背景需要说明。

**简洁示例：**

```java
/**
 * 模块：差旅申请审批
 * 说明：封装审批、校验与流程触发逻辑
 */
```

若类职责变动而注释不能同步更新，则宁可删除。

---

### **8.3 方法级注释（必须）**

```java
/**
 * 功能：校验差旅申请是否合规
 * 输入：TravelApp（包含出差时间、预算等）
 * 返回：true 合规；false 不合规
 * 业务逻辑：
 *   - 校验时间是否合法
 *   - 校验预算是否超限
 *   - 超预算触发财务审批
 */
public boolean validateTravelApplication(TravelApp app) { ... }
```

---

### 8.4 行级业务注释（必须，且注释放在代码上一行）

- 所有行级注释必须**写在代码上一行**，不允许写在行尾；
- 注释必须解释“为什么这样写”或“这段逻辑的业务意图”。

#### 示例（正确）
```java
// 若预算超限，必须走财务审批流（不是普通审批）
if (amount > budgetLimit) {
    triggerFinanceApproval();
}

// 校验申请信息（时间、预算、目的地等）
validateTravelApplication(app);
````

#### 反例（错误）

```java
if (amount > budgetLimit) { triggerFinanceApproval(); } // 行尾注释（禁止）
```

---

### 8.5 行级方法调用注释（强制写在方法调用上一行）

- 适用于复杂业务流程中调用多个方法的情况；
- 注释写在调用方法的上一行，用于说明该方法在当前业务流程中的作用；
- 这样代码阅读者无需跳转即可理解逻辑。

#### 示例（正确）
```java
// 校验申请信息（时间、预算、目的地等）
validateTravelApplication(app);

// 触发审批流（超预算 → 财务审批）
triggerApprovalFlow(app);

// 记录审批历史（用于审计）
logApprovalHistory(app);
````

---

### 8.6 Lambda 表达式注释（必须写在 Lambda 的上一行）

- 所有 Lambda 表达式必须在上一行添加业务说明；
- 禁止使用 `[Lambda-xx]`、`[AI-xx]` 等自动标签。

#### 示例（正确）
```java
// 过滤掉不在有效时间范围内的记录
List<Record> validRecords = records.stream()
    .filter(r -> r.getStartTime().isAfter(start) && r.getEndTime().isBefore(end))
    .toList();
````

#### 反例（错误）

```java
records.stream()
    .filter(r -> ...) // 过滤有效记录（禁止行尾注释）
    .toList();
```

#### 📌 附：关于行级注释风格的总结（Codex 将按此生成）

- **所有行级注释 → 统一写在代码上一行**
- **绝不允许写在行尾**
- **描述业务意图，而不是代码语法**
- **方法调用、Lambda、判断语句、重要赋值都必须放注释在上一行**
- **禁止自动编号（GP-xxxx, AI-xxxx）**

---

### **8.7 逻辑块分段注释（推荐）**

```java
// ---------- 校验阶段 ----------

// ---------- 审批阶段 ----------

// ---------- 写日志 ----------
```

---

### **8.8 审查要求（必须执行）**

* 缺少方法注释、关键业务逻辑注释 → 不允许提交；
* 出现自动编号（GP-xxxx / AI-xxxx / AUTO-xxxx）→ 一律退回修改；
* 需求驱动代码需遵循第 7 节“需求对应规范”；
* 所有注释必须与代码同步更新，否则视为错误信息。


## **9. 数据库更新操作注释规范（必选）**

为确保数据变更行为可追溯、可审计、可读性强，所有通过 MyBatis / MyBatis-Plus 执行的数据库更新动作（包括 `update`、`delete`、`insert`、`lambdaUpdate`、`lambdaQuery`、`updateById`、XML 中的 UPDATE 语句）必须添加 **清晰中文注释**，说明本次更新的业务意图与作用范围。

此规范适用于 **所有 Service 层的数据库写操作**。

---

### **9.1 注释原则**

#### ✔ 必须：

1. 注释 **放在更新语句的上一行**，不能写在行尾；
2. 注释必须为 **中文**；
3. 注释需清晰描述本次更新“为什么要更新”“更新了哪些数据”；
4. 注释应明确 **操作对象与数据范围**（例如：按员工编号、日期范围、状态枚举等）；
5. 注释需基于业务语义，而非代码描述。

#### ❌ 禁止：

* 行尾注释；
* 英文注释；
* 模糊注释（如“修改数据”“处理逻辑”之类没有实际意义的注释）；
* 自动编号（GP-xxx / AI-xxx 等）。

---

### **9.2 正确示例（符合规范的写法）**

以下是你提供的示例，转换为符合规范的版本：

```java
// 删除指定员工在 dateList 范围内的排班记录（业务：逻辑删除）
attendClockService.lambdaUpdate()
    .eq(GcsAttendClock::getEmployeeCode, employeeCode)
    .in(GcsAttendClock::getShiftDate, dateList)
    .ne(GcsAttendClock::getShiftCode, AttendClockConst.EMERGENCE_SUPPORT_SHIFT_CODE)
    .set(GcsAttendClock::getDeleteStatus, DeleteStatusEnum.DELETED.getCode())
    .set(GcsAttendClock::getDeleteTime, now)
    .update();
```

#### 注释解释：

* 使用 **上一行注释**；
* 使用 **中文描述业务动作**；
* 指明了 **dateList 范围内的数据被删除（逻辑删除）**；
* 注释内容与业务完全对应。

---

### **9.3 更复杂场景示例**

#### ✔ 示例 1：根据时间范围批量关闭订单

```java
// 关闭超过超时时间未支付的订单（业务：定时任务清理）
orderService.lambdaUpdate()
    .eq(Order::getStatus, OrderStatusEnum.PENDING.getCode())
    .lt(Order::getCreateTime, expiredTime)
    .set(Order::getStatus, OrderStatusEnum.CLOSED.getCode())
    .update();
```

---

#### ✔ 示例 2：XML 中的 UPDATE 必须写块注释

```xml
<!-- 
    业务含义：将员工指定日期的打卡记录标记为无效（逻辑删除）。
    触发场景：后台批量清理操作。
-->
<update id="markInvalid">
    UPDATE attend_clock
    SET delete_status = 1,
        delete_time = SYSDATE
    WHERE employee_code = #{employeeCode}
      AND shift_date IN 
      <foreach collection="dates" item="d" open="(" close=")" separator=",">
          #{d}
      </foreach>
</update>
```

---

#### ✔ 示例 3：updateById 也必须注释

```java
// 更新用户头像（业务：用户修改资料）
userMapper.updateById(user);
```

---

### **9.4 适用于哪些情况？**

| 场景                                 | 是否必须写             |
| ---------------------------------- | ----------------- |
| update / lambdaUpdate / updateById | ✔ 必须写             |
| insert / save                      | ✔ 推荐写（涉及业务意义时必须写） |
| delete / remove / removeById       | ✔ 必须写             |
| 纯查询（仅 select）                      | ✘ 不用写             |
| 批量更新 / 批量删除                        | ✔ 必须写             |
| XML 中的 UPDATE / DELETE             | ✔ 必须写             |

---

### **9.5 业务注释模板（可直接复制使用）**

你也可以添加以下模板供 AI 使用：

```
【数据库更新注释模板】

// 业务意图：XXXX（例如：删除、更新状态、批量关闭等）
// 操作范围：XXXX（例如：按员工编号、日期范围、订单状态等）
// 更新原因（可选）：XXXX（例如：纠错、定时清理、业务流程推进）
```

实际示例：

```java
// 业务意图：逻辑删除排班记录
// 操作范围：指定员工、dateList 日期范围
attendClockService.lambdaUpdate()...
```

## 10. 业务逻辑闭环检查模板（架构师 + 产品经理双视角）

> 当我要求 “按业务逻辑闭环检查模板检查 XX 类” 时，必须严格按照以下指引进行审查。

你需要以 **高级架构师 + 产品经理** 的双重身份，从业务与技术的角度对指定类或模块进行深度逻辑审查。

---

### 【1】职责识别（必须）
- 根据代码推断该类的**核心职责**；
- 用你自己的话概括成 3–6 条；
- 若职责模糊或超出单类应承担范围，应明确指出。

---

### 【2】产品经理视角的业务审核（必须）
从“是否符合需求逻辑与业务流程”的角度检查：

1. 是否覆盖了需求描述中的主流程？
2. 是否覆盖所有关键场景（正常场景 + 异常场景 + 边界场景）？
3. 用户行为路径是否完整？
4. 是否存在业务流程不连贯或遗漏的部分？
5. 状态是否按照业务预期变化？
6. 是否存在可能导致数据错误、重复、覆盖、缺失的场景？
7. 是否有未处理的业务异常场景？

若不符合，请指出**具体方法或具体逻辑位置**。

---

### 【3】高级架构师视角的技术审核（必须）
从“系统流程能否跑通、数据链路是否闭环”的角度检查：

1. 数据流向（入口 → 处理 → 输出/落库）是否闭环？
2. 是否存在漏推进，例如：
    - 消费了消息但未入库
    - 写库失败后未补偿
    - 调度任务可能漏跑
3. 是否存在重复处理（消息重复消费/重复落库/重复触发）？
4. 是否存在状态不同步（部分字段更新、条件错误等）？
5. 异常分支是否完整？是否可能中断流程？
6. 边界条件是否处理完整（null、空列表、0 值、超时、网络异常等）？
7. 是否存在并发/异步导致的竞态风险？
8. 是否存在不对称逻辑（A→B 有逻辑，B→A 却没有）？

若有风险，请列出具体触发场景。

---

### 【4】最终输出格式（必须按此结构输出）
你必须按以下结构输出：

#### ① 类职责总结
（用 3–6 条列出）

#### ② 产品经理视角审查结论
- 覆盖项
- 遗漏项
- 风险项
- 指向具体方法/逻辑行

#### ③ 架构师视角审查结论
- 逻辑闭环情况
- 异常路径情况
- 边界条件
- 事件链路
- 数据一致性情况
- 潜在失败场景

#### ④ 问题清单（如有）
以列表形式输出所有问题与对应代码位置。

#### ⑤ 风险点（如有）
列出可能导致业务事故/生产问题的点。

#### ⑥ 改进建议（必须具体到方法或逻辑块）
例如：
- “在 X 方法的 Y 条件下需要补充 Z 校验”
- “在消息消费完成后应补偿写库”
- “并发情况下需加锁处理 ABC”

---

### 【5】规则要求（必须遵守）
- 不得生成“看起来都没问题”的泛泛总结；
- 必须进行**逐步推理**；
- 必须列出“为什么这样判断”；
- 必须找出真实风险（如果存在）；
- 必须从**两个角色**输出（产品经理 + 架构师）；
- 输出必须结构化、成体系。

---

## 11. AI 辅助开发流程规范（模型选择策略）

为了提升开发质量、减少逻辑缺陷，并确保业务流程闭环，AI 模型（Codex 系列）在不同阶段需要使用不同模式。本流程规范定义了在需求分析、开发、复查及上线审查中的推荐模型与工作方式。

---

### 11.1 新需求分析阶段（使用：codex-high）
当接到新的业务需求时，使用 **codex-high** 进行：
- 需求分解
- 影响范围分析
- 找出需要修改的代码文件与模块
- 识别对 Kafka、Job、DB 的联动影响
- 识别边界条件、异常场景
- 生成初步技术方案（可选）

---

### 11.2 自测与需求闭环检查（使用：codex-high）
开发完成后，切换回 **codex-high**，使用：

- 《业务逻辑闭环检查模板》（第 10 节）

进行以下验证：
- 是否完整实现需求
- 是否遗漏业务场景
- 是否符合产品经理描述的用户路径
- 是否遵守边界条件
- 是否存在单模块内部逻辑漏洞
- 是否存在错误的状态流转


---

### 11.3 上线前系统级审查（使用：codex max-high）
在准备上线前，使用 **codex max-high** 对整个模块做“系统级审查”，重点分析：

- 是否影响其他业务模块
- Kafka → Job → DB → 下游链路是否仍然闭环
- 状态机是否被破坏
- 是否可能出现漏推进、重复处理、数据不一致
- 并发条件、异常分支是否安全
- 多模块组合逻辑是否被动破坏

---


