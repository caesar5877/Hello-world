
好的，我将根据您提供的资料，用中文详细解释关于将 DPM 和 CSR Admin Tool 引入 CaaS 的信息。

这份设计文档概述了将 Auth Secure Transaction 的一次性密码 (OTP) 生成流程迁移到挑战即服务 (CaaS) 平台的设计，旨在增强安全性和操作效率 [1]。当前，客户服务代表 (CSR) 仍然通过 Call Us 页面生成 OTP [1]。

以下是具体的解决方案步骤、设计考虑、分析和试点策略：

**一、建议的解决方案步骤**
为了将 OTP 生成流程迁移到 CaaS，设计了以下步骤：
1.  **用户身份验证启动** [1]：
    *   当用户需要身份验证时，他们将在 Call Us 页面上触发 CaaS 平台的服务 [1]。
    *   用户将通过风险 CPM API 与 DPM 团队交互，以验证风险密钥 [1]。
2.  **客户服务代表 (CSR) 交互** [1]：
    *   客户将联系 CSR [1]。
    *   CSR 将在 CaaS 会话中调用一个新的 CaaS 会话，将用户的交易与风险评估链接起来 [1]。
    *   此步骤还将涉及使用 CSR 的 CFM API 来验证风险密钥 [1]。
3.  **短信挑战调用** [1]：
    *   CSR 将为用户触发短信挑战 [1]。
    *   OTP 将生成并存储在核心数据库和 CaaS 数据库中 [1]。
    *   OTP 随后将通过通信团队发送短信 [1]。
    *   进度将传达给挑战即服务团队以供进一步处理 [1]。
4.  **用户 OTP 交互** [1]：
    *   收到短信后，用户将在用户界面上点击“我有一个代码” [1]。
    *   此操作将触发 API 调用以检索 OTP 前缀 [1]。
5.  **最终身份验证** [1]：
    *   用户输入 OTP 以验证其身份 [1]。
    *   此步骤涉及调用 RFCM API 以获取风险团队的最终裁决 [1]。
    *   根据此裁决，状态将返回给用户，完成验证流程 [1]。

**二、设计考虑**
此集成过程包含几个关键的设计考虑：
1.  **CaaS 调用** [1]：
    *   CaaS 将被调用两次，第一次调用来自 DPM/CaaS 工具，第二次调用来自 CaaS 本身 [1]。
    *   CaaS 将拥有其独特的挑战令牌标识符和风险密钥标识符，例如 Wire、Unrecognized DVC、FQPW 等 [1]。
    *   CaaS API 调用的顺序将是：ChallengeOptions (挑战选项)，接着是 ListOTPPrefix (列出 OTP 前缀)，最后是 ChallengeVerification (挑战验证) [1]。
2.  **CSR CaaS 调用** [1]：
    *   CSR 将拥有其独特的挑战令牌标识符和风险密钥标识符，这些标识符将与用户挑战 CaaS 调用中的不同 [1]。
3.  **CaaS 和 CSR - 渠道 ID 入职** [2]：
    *   需要创建 CaaS 挑战标识/挑战选项/原因 [2]。
    *   需要提供未屏蔽电话号码列表和额外的屏蔽电话信息 [2]。
    *   DPM/Admin Tool 到 CaaS 需要 API-ID [2]。
4.  **挑战令牌集成** [2]：
    *   挑战令牌标识符将包括联系人参考 ID、原因代码、通信模式和挑战密钥 [2]。这将是基于交易的 [2]。
    *   `/cs/v3/challenge/verify/challenge-options` 将映射所有原因代码 [2]。
    *   `/cs/v3/challenge/verify/generate/otp` 和 `/cs/v3/challenge/verify/verify` 将所有重写映射到 CaaS，然后禁用旧有表 [2]。
    *   **用户状态更新** [2]：
        *   **方案 1 - 原因代码 1 (注册/密码重置)**：如果用户处于 LOCKed 状态创建 OTP，其他 OTP 生成将失败 [2]。当从 CSR/DPM 生成 OTP 时，这将调用 CaaS Challenge/Invoker 来生成 OTP [2]。OTP 将返回到 LDAP，这意味着 OTP 在 LDAP 中的停留时间更长 [2]。访问 API 将被调用，将挑战流映射到 CaaS Challenge/Invoker 来生成 OTP [2]。所有其他 OTP 生成将调用 CaaS Challenge/Invoker 来生成 OTP，无需访问 LDAP [2]。
        *   **方案 2 - CS/RDPM 调用的 CaaS API Challenge/Invoker**：当来自 CS/RDPM 的挑战流调用 CaaS API Challenge/Invoker 时，OTP 将作为事件与 OTP 加密一起通过 [2]。事件处理程序将更新用户状态和 LDAP 密码 [2]。
        *   **方案 3 - 原因代码 2 (注册) - 针对新用户或用户状态更新**：访问 API 将监听 Challenge/Invoker 事件，以通知用户已通过注册/密码重置流程 [2]。访问 API 将构建 Kafka 消费者来监听 Challenge/Invoker 事件并更新用户状态以获得 OTP_PA_SSWORD_EXPIRED，当注册/密码重置发生时 [2]。这将导致 CSR OTP 值被 CaaS 启动 [2]。当 CaaS OTP 不再停留时，新的 CaaS OTP 生成将发生 [2]。
        *   **当用户调用 OTP 时**：Challenge/Verify API 将验证 OTP 字段值 [2]。需要支持 OTP 验证，但没有预设 [2]。预设验证将不支持此流程 [2]。所有 OTP 预设都被忽略，以用于注册/密码重置流程 [2]。
    *   IRR 集成时间表：待与 Preadmin 和控制团队确定 [2]。
5.  **风险集成** [2]：
    *   CaaS 需要与挑战令牌标识符或风险标识符的 CFM/Risk 链接 [2]。
    *   在挑战调用中，系统已经知道挑战令牌标识符和风险标识符 (原因代码) 是在生成 OTP 后发生的 [2]。
    *   在 Risk Progress Event Kafka 中，通过挑战令牌标识符和风险标识符提供原因代码是可行的 [2]。
    *   RFCM API 中不存在此数据，因此不需要从 RFCM API 中删除数据 [2]。
    *   用户无法访问的数据可能需要删除 (如果 API 调用需要 RFCM) [2]。
    *   对于 RFCM 中没有 Reason Code 的所有拒绝，CaaS 将提供一个默认值 [2]。
    *   当 DPM/Admin Tool 在 RFCM 中没有 Reason Code 时，挑战令牌标识符/风险标识符的确定可以通过在挑战令牌标识符/风险标识符中使用 DPM/Admin Tool 的理由来完成 [2]。
6.  **Zelle Tag 集成** [2]：
    *   DPM Admin Tool 将调用带有 Zelle Tag 密钥对的 Zelle Tag API [2]。
    *   Zelle Tag API 将仅用于 OTP 密钥 [2]。
    *   电话号码或电子邮件将来自 CaaS ChallengeOptions API 调用 [2]。
    *   CaaS ChallengeOptions API 调用将有一个新的字段来存储以前的挑战选项 [2]。
    *   遗留流程将使用未从 CaaS ChallengeOptions API 调用中检索到的电话号码 [2]。
7.  **Chip_disp 集成** [2]：
    *   将使用此表将挑战令牌标识符与 CSR 提供的原始 CaaS 关联起来 [2]。
    *   当进行新挑战令牌标识符调用时，系统会将 person-id、reason code 和 challenge token identifier 插入到 `chip_disp` 表中 [2]。
    *   当挑战令牌标识符/风险标识符更新时，它将更新 `chip_disp` 表中的人 ID、原因代码和挑战令牌标识符 [2]。
    *   当 CSR 调用 ChallengeInvoker 时，它会将人 ID、原因代码和挑战令牌标识符插入到 `chip_disp` 表中 [2]。

**三、分析**
在用户流的挑战/验证过程中，将从 `chip_disp` 表中检索数据，从缓存数据中审查信息（在 Invoker 步骤中完成），并将挑战令牌标识符和风险 ID 作为审查的一部分发送到 CSR 流中，以便 CSR 访问，并用于 RBAID 进度事件以处理失败 [3]。
*   **子 (CSR) ChallengeOptions** [3]：
    *   需要分析 ChallengeOptions Slot (13098) 中挑战值的原因 [3]。
    *   “理由”字段的预期值为“未确定” [3]。
*   **ChallengeInvoker** [3]：
    *   可以将原始（父）挑战令牌标识符映射到 ChallengeInvoker 的 CSR 流事件 [3]。
    *   原因代码值将作为请求的一部分从缓存/板载数据中传递 [3]。
    *   CSR 挑战令牌标识符将被传递到 ChallengeInvoker 的用户流事件 [3]。
*   **ChallengeVerify** [3]：
    *   CSR 挑战令牌标识符将被传递到 ChallengeVerify slot event 的用户流 [3]。
    *   预期值为“CHALLENGETOKENIDENTIFIER: EDOBAGCJAG | CHILD_CHALLENGETOKENIDENTIFIER: ABDOBAGCJAG” [3]。

**四、CaaS 和 CSR 原因代码** [3]
可以在两种可能的解决方案中支持此功能：
*   **方案 1 (CSR Admin Tool 作为此解决方案)**：创建新的 ListReasonCodes API [3]。但这可能不是最佳选择，因为它耦合度高 [3]。然而，需要创建新的 API 来支持简单的 API [3]。
    *   RDI - 参考数据干扰：跟进未来的集成 [3]。
*   **方案 2 (审计团队首选解决方案)**：修改 ChallengeOptions API，以便 CSR 流返回 Reason Codes 列表 [3]。此选项允许所有 Reason Codes 均来自 CaaS [3]。
    *   创建一个新表来存储 Reason Codes 列表及其描述 [3]。
    *   在 DPM/Admin Tool UI 上显示 Reason Codes 列表 [3]。
    *   可能需要在此处添加一些与这些类别对应的支持流 ID [3]。
    *   Mazid, Payam 确认这超出了本项目范围 [3]。
*   **渠道 ID** [3]：
    *   DCS - DPM [3]。
    *   CSR - Admin Tool [3]。

**五、试点策略** [3]
*   **原因代码**：入驻新的原因代码（应该已经完成入驻） [3]。
*   **CSR Admin Tool**：将调用 Zelle Tag API 并检索电子邮件和电话号码 [3]。
*   **Challenge Verify**：更新 API 以在请求中接受电子邮件和电话号码 [3]。

总体而言，这项工作是将 Auth Secure Transaction 的关键流程，特别是 OTP 生成，迁移到更现代化、安全且高效的 CaaS 平台 [1]。

好问题 👍。这些缓存策略的名字，其实都是根据数据写入（write）时的处理方式来命名的。理解这些名字，可以从 “数据写到哪里” 和 “什么时候写” 这两个维度来看。

⸻

1. Write Through（直写）
	•	名字含义：数据写操作直接“穿透（through）”缓存，立即写入到主存。
	•	机制：
	•	数据写入缓存的同时，也立刻写入主存。
	•	保证主存和缓存的数据始终一致。
	•	特点：
	•	简单、可靠，一致性好。
	•	但写操作延迟高，因为每次都要写主存。

⸻

2. Write Back（回写 / 延迟写）
	•	名字含义：写入先在缓存里“积累”，等到需要替换或特定时机再“写回（back）”主存。
	•	机制：
	•	数据先写缓存，并标记为“脏数据”（dirty）。
	•	只有当缓存行被替换时，才把脏数据写回主存。
	•	特点：
	•	写操作很快（只写缓存）。
	•	减少主存访问次数，提高性能。
	•	但需要额外机制保证一致性（比如缓存一致性协议）。

⸻

3. Write Around（绕写）
	•	名字含义：写操作绕过（around）缓存，直接写主存，不写缓存。
	•	机制：
	•	写数据时直接写到主存。
	•	只有在读时才把数据加载进缓存。
	•	特点：
	•	避免因为大量写入带来的缓存污染（比如写入一次性数据）。
	•	但写完立刻读，可能会 cache miss。

⸻

4. 对比一下

策略	写缓存	写主存	主存一致性	适用场景
Write Through	✅	✅	始终一致	数据一致性要求高
Write Back	✅	延迟	延迟一致	写频繁、性能优先
Write Around	❌	✅	始终一致	避免缓存污染


⸻

所以，这些名字都是直观的：
	•	Through = 穿透 → 每次写都穿透到主存。
	•	Back = 回写 → 先放缓存，之后再写回主存。
	•	Around = 绕过 → 写的时候绕开缓存。

⸻

要不要我帮你画一张示意图，把三个策略的写入路径直观对比一下？