# DPM 智能日志数据分析与管理子系统接口文档

本仓库为 Bruno 接口集合，原始 OpenAPI 导入接口保留在各业务目录中。新增的自动化用例统一放在 `日志请查-自动化/`，避免和导入接口混在一起。

## 日志请查自动化

当前已落地 `日志请查-自动化/01 草稿完整提交链路`，覆盖申请人侧 P0 主链路：

1. 初始化日志请查草稿
2. 自动保存完整草稿
3. 提交草稿
4. 查询提交后详情
5. 地市列表回查提交申请

该链路会在运行时保存 `draftId`、`requestId`、`requestNo`，后续请求直接复用前序接口返回值。

当前也已落地 `日志请查-自动化/02 正式申请生命周期`，覆盖一次性创建正式申请后的生命周期：

1. 一次性创建正式申请
2. 查询详情并保存 `lifecycleRequestId`、`lifecycleRequestNo`
3. 省厅待办按申请编号回查
4. 申请人撤回
5. 整改续办重新提交
6. 再次撤回
7. 作废并查询最终详情

该链路需要 `provinceToken`，用于校验省厅待办可见性。

当前还已落地 `日志请查-自动化/03 权限与异常`，覆盖首批高风险权限隔离：

1. 申请人准备一条待审核申请
2. 省厅账号不能新建申请
3. 申请人不能审批
4. 申请人不能查询省厅待办/已办
5. 其他申请人不能查看或撤回当前申请
6. 越权操作后原申请仍保持 `PENDING`

该链路需要 `otherApplicantToken`，并假设该账号不具备当前申请的查看和操作权限。

`日志请查-自动化/04 参数与边界` 覆盖正式创建和分页的高频参数校验：

1. `validDays` 小于 1 或大于 90
2. 申请原因超过 1000 字
3. 调取范围为空或超过 10 条
4. 日志结束时间早于开始时间
5. 日志时间范围超过 180 天
6. 分页缺少页码或每页条数超过 10000

`日志请查-自动化/05 审批规则` 覆盖不依赖真实附件文件的审批状态机负向规则：

1. 非紧急同意缺少领导批示
2. 不同意缺少办理意见
3. 不同意携带领导批示附件
4. 非法审批结论
5. 负向校验后申请仍保持 `PENDING`

`日志请查-自动化/06 附件与导出` 覆盖文件类接口：

1. 使用 `files/fixtures/log-request-application.csv` 上传申请附件
2. 详情回查附件元数据，不允许泄露 `storagePath` 或本地路径
3. 下载附件并按二进制文件流断言
4. 删除附件后再次下载必须被拒绝
5. 使用 `files/fixtures/blocked-attachment.txt` 校验不支持扩展名被拒绝
6. 城市侧导出申请列表并按 Excel 文件流断言

`日志请查-自动化/07 非紧急同意审批闭环` 覆盖审批正向主链路：

1. 申请人创建非紧急待审核申请
2. 省厅待办按申请编号回查
3. 省厅上传 `LEADER_DIRECTIVE` 领导批示附件
4. 省厅同意审批并绑定领导批示
5. 详情确认 `COMPLETED`、`directiveStatus=UPLOADED` 和 `LOG_READ` 授权
6. 流程日志确认同意审批节点关联领导批示附件
7. 省厅已办列表回查已完成申请

`日志请查-自动化/08 紧急同意后补传闭环` 覆盖紧急审批主链路：

1. 申请人创建紧急待审核申请
2. 省厅无领导批示先同意
3. 详情确认 `COMPLETED`、`directiveStatus=PENDING`、授权立即生效且 `pendingDirective=true`
4. 省厅上传并补传 `LEADER_DIRECTIVE`
5. 详情确认 `directiveStatus=UPLOADED` 且授权起止时间不变
6. 流程日志确认补传节点关联领导批示附件
7. 省厅已办列表回查补传完成的紧急申请

`日志请查-自动化/09 驳回后整改续办闭环` 覆盖省厅不同意后的整改重提主链路：

1. 申请人创建非紧急待审核申请
2. 省厅不同意驳回并填写办理意见
3. 详情确认 `REJECTED`、`directiveStatus=NONE` 且无授权
4. 流程日志确认存在 `处置办理：不同意` 和驳回意见
5. 省厅已办列表回查驳回申请
6. 申请人基于同一申请整改续办重新提交
7. 详情确认同一 `id/requestNo` 回到 `PENDING`
8. 流程日志确认保留驳回记录并追加 `申请修改`、`重新提交`
9. 省厅待办列表回查整改重提申请

`日志请查-自动化/10 导出权限与列表过滤` 覆盖导出接口的分支和权限：

1. 创建一条 `PENDING` 待办申请并保存 `requestNo`
2. 城市侧按申请编号导出待办列表
3. 申请人使用 `PROVINCE_TODO` 导出省厅待办必须被拒绝
4. 省厅使用 `PROVINCE_TODO` 按申请编号导出待办列表
5. 创建紧急申请并同意审批，形成 `COMPLETED` 已办数据
6. 省厅使用 `PROVINCE_DONE` 按申请编号导出已办列表
7. 未知 `pageType` 必须被业务拒绝

`日志请查-自动化/11 领导批示绑定规则` 覆盖领导批示附件绑定和补传负向规则：

1. 申请 A 审批时绑定申请 B 的 `LEADER_DIRECTIVE` 必须被拒绝
2. 跨申请绑定失败后，申请 A 仍保持 `PENDING`
3. 上传后删除的 `LEADER_DIRECTIVE` 不能再用于审批
4. 删除后绑定失败后，申请 A 仍保持 `PENDING`
5. 已完成且 `directiveStatus=UPLOADED` 的申请不能再次补传领导批示
6. 非待补传补传失败后，申请仍保持 `COMPLETED/UPLOADED`
7. 申请人账号不能补传领导批示

`日志请查-自动化/12 附件数量限制` 覆盖同类型附件数量边界：

1. 创建待审核申请后连续上传 3 个 `APPLICATION` 附件
2. 第 4 个同类型附件必须被业务拒绝，错误原因包含 `最多3个`
3. 详情回查只存在 3 个未删除 `APPLICATION` 附件
4. 删除其中 1 个附件后，同类型计数释放
5. 再次上传 `APPLICATION` 附件成功
6. 最终详情回查未删除 `APPLICATION` 附件仍为 3 个，且不泄露 `storagePath` 或本地路径

`日志请查-自动化/13 审批兼容与重复审批` 覆盖审批状态机边界：

1. 非紧急申请使用英文 `AGREE` 结论并绑定领导批示，审批成功后状态为 `COMPLETED/UPLOADED`
2. 已完成申请再次审批必须被业务拒绝，且流程日志不新增重复审批意见
3. 办理意见超过 500 字必须被拒绝，失败后申请仍保持 `PENDING`
4. 申请使用英文 `DISAGREE` 结论并填写意见，审批成功后状态为 `REJECTED/NONE`
5. 已驳回申请再次审批必须被业务拒绝，且状态和流程日志保持不变

`日志请查-自动化/14 草稿隔离与删除` 覆盖草稿权限和可见性：

1. 省厅账号不能初始化草稿
2. 申请人本人可初始化并自动保存部分字段草稿
3. 草稿本人详情可见，但不产生用户可见办理流程日志
4. 其他申请人不能查看或删除该草稿
5. 他人删除失败后，草稿仍归本人可见
6. 省厅待办/已办列表不返回 `DRAFT`
7. 本人删除草稿后，再查详情返回业务不存在或不可访问

`日志请查-自动化/15 附件权限与下载越权` 覆盖附件主体权限：

1. 其他申请人不能上传、下载或删除当前申请的 `APPLICATION` 附件
2. 申请人不能上传 `LEADER_DIRECTIVE`
3. 他人越权下载附件时不能返回文件流
4. 他人越权删除失败后，申请人本人仍可下载原申请附件
5. 省厅可上传并下载 `LEADER_DIRECTIVE`
6. 申请人不能下载省厅上传的领导批示附件
7. 所有附件越权失败响应不得泄露 `storagePath` 或本地路径

`日志请查-自动化/16 列表筛选权限隔离` 覆盖列表筛选不能扩大权限：

1. 申请人 A 创建正式申请并保存申请编号
2. A 本人按申请编号查询地市列表可见该申请
3. 申请人 B 按 A 的申请编号查询地市列表不可见该申请
4. B 额外携带伪造 `applicantAccountId/applicantOrgId/applicantName` 筛选条件仍不可见
5. B 不能直接查看 A 的申请详情，失败响应不泄露申请内容或本地路径

`日志请查-自动化/17 状态流转非法操作` 覆盖申请状态机负向流转：

1. `PENDING` 申请不能直接作废
2. `PENDING` 申请不能整改续办重提
3. 待审核非法操作后状态仍为 `PENDING`
4. `WITHDRAWN` 申请不能重复撤回
5. `WITHDRAWN` 可作废为 `VOIDED`
6. `VOIDED` 申请不能撤回或整改续办重提
7. 非法操作不追加错误流程日志，失败响应不泄露本地路径

`日志请查-自动化/18 附件文件边界` 覆盖附件文件输入边界：

1. 抽样验证 `doc/xlsx/pdf/png/jpg/csv` 允许扩展名上传成功
2. 每条申请最多 3 个同类型附件，因此允许扩展名成功路径拆为申请 A/B
3. 详情回查附件元数据，不泄露 `storagePath` 或本地路径
4. 0 字节空文件必须被拒绝
5. 缺少 `file` multipart 参数必须被拒绝

`日志请查-自动化/19 动作权限隔离` 覆盖申请侧动作权限：

1. 省厅账号不能撤回、作废或整改续办申请
2. 其他申请人不能作废或整改续办当前申请
3. 越权动作失败后，原申请仍保持 `PENDING`
4. 越权失败不追加撤回、作废或整改续办流程日志
5. 越权失败响应不泄露 `storagePath` 或本地路径

`日志请查-自动化/20 附件类型独立计数` 覆盖附件计数维度：

1. 同一申请先上传 3 个 `APPLICATION`
2. 已有 3 个 `APPLICATION` 后，仍可上传 3 个 `LEADER_DIRECTIVE`
3. 第 4 个 `LEADER_DIRECTIVE` 必须被业务拒绝，错误原因包含 `最多3个`
4. 详情回查 `APPLICATION` 与 `LEADER_DIRECTIVE` 分别各 3 个未删除附件
5. 详情和失败响应不泄露 `storagePath` 或本地路径

`日志请查-自动化/21 畸形JSON统一错误` 覆盖请求体解析失败的统一兜底：

1. 创建申请接口收到不可解析 JSON 时返回统一 `Result` 错误
2. 详情查询接口收到不可解析 JSON 时返回统一 `Result` 错误
3. 地市分页接口收到不可解析 JSON 时返回统一 `Result` 错误
4. 失败响应不能退化为 HTML/Whitelabel 错误页，且不泄露异常类名、`storagePath` 或本地路径

`日志请查-自动化/22 必填字段校验` 覆盖正式创建申请的必填字段：

1. 缺少 `targetOrgId/targetOrgName/validDays/urgent/requestReason/scopes` 必须被拒绝
2. `scopes` 明细缺少 `systemRange` 必须被拒绝
3. `scopes` 明细缺少日志开始或结束时间必须被拒绝
4. 所有参数错误响应不泄露 `storagePath` 或本地路径

`日志请查-自动化/23 允许扩展名补充` 补齐附件允许扩展名：

1. 创建一条正式申请后上传 `docx/xls/jpeg` 三个 `APPLICATION` 附件
2. 详情回查三个附件均为未删除申请附件
3. 附件元数据不泄露 `storagePath` 或本地路径

`日志请查-自动化/24 附件超大文件限制` 覆盖单文件大小边界：

1. 创建一条正式申请
2. 上传 `files/generated/oversize-attachment.csv`，大小为 10MB + 1 字节，必须被业务拒绝
3. 详情回查不能出现超大附件残留
4. `files/generated/` 已被 `.gitignore` 忽略，运行前用以下命令在集合根目录生成临时大文件：

```bash
python3 -c "from pathlib import Path; p=Path('files/generated/oversize-attachment.csv'); p.parent.mkdir(parents=True, exist_ok=True); p.write_bytes(b'a' * (10 * 1024 * 1024 + 1))"
```

`日志请查-自动化/25 分页参数下界与排序校验` 补充分页参数约束：

1. `pageNum=0` 必须被拒绝
2. 缺少 `pageSize` 必须被拒绝
3. `pageSize=0` 必须被拒绝
4. `sortType=0` 或 `sortType=3` 必须被拒绝
5. 参数错误响应不泄露 `storagePath` 或本地路径

`日志请查-自动化/26 附件请求参数缺失` 覆盖附件接口的 `@RequestParam` 缺失：

1. 上传缺少 `requestId` 必须返回统一错误
2. 上传缺少 `attachmentType` 必须返回统一错误
3. 下载缺少 `attachmentId` 必须返回统一错误且不能返回文件流
4. 失败响应不能退化为 HTML/Whitelabel 错误页，且不泄露 `storagePath` 或本地路径

`日志请查-自动化/27 附件删除参数校验` 覆盖附件删除 JSON 请求体参数：

1. 缺少 `requestId` 必须被拒绝
2. 缺少 `attachmentId` 必须被拒绝
3. 缺少 `attachmentType` 必须被拒绝
4. 空 JSON 请求体必须被拒绝，且不能返回删除成功

`日志请查-自动化/28 导出参数错误不返回文件流` 覆盖导出接口的分页与排序参数校验：

1. 缺少 `pageNum` 必须返回统一错误，且不能返回 Excel 文件流
2. 缺少 `pageSize` 必须返回统一错误，且不能返回 Excel 文件流
3. `pageSize=10001` 必须被拒绝，且不能返回 Excel 文件流
4. `sortType=0` 或 `sortType=3` 必须被拒绝，且不能返回 Excel 文件流
5. 参数错误响应不能退化为 HTML/Whitelabel 错误页，且不泄露 `storagePath` 或本地路径

`日志请查-自动化/29 主体字段防伪造` 覆盖客户端主体字段不能覆盖服务端可信数据：

1. 创建申请时伪造 `applicantAccountId/applicantOrgId/targetOrgName`
2. 详情回查必须使用服务端主体与机构信息，不能透出伪造字段
3. 地市列表按申请编号回查也不能透出伪造字段
4. 用例结束撤回并作废该申请，减少待审核数据残留
5. 作废后详情仍不能泄露伪造主体字段、本地路径或 `storagePath`

`日志请查-自动化/30 草稿提交前校验` 覆盖草稿保存与正式提交校验差异：

1. 草稿阶段允许只保存部分字段
2. 部分字段草稿提交必须被拒绝，且失败后仍保持 `DRAFT`
3. 草稿阶段允许保存空调取范围
4. 无调取范围草稿提交必须被拒绝，且失败后仍保持 `DRAFT`
5. 提交失败不能产生用户可见流程日志，结束时删除测试草稿

`日志请查-自动化/31 动作接口申请ID缺失` 覆盖 JSON 动作接口的申请 ID 必填校验：

1. 详情、删除草稿、提交草稿缺少 `id` 必须被统一拒绝
2. 撤回、作废、整改续办缺少 `id` 必须被统一拒绝
3. 审批、补传领导批示缺少 `id` 必须被统一拒绝
4. 缺参失败不能误返回 `data=true`
5. 失败响应不能退化为 HTML/Whitelabel 错误页，且不泄露 `storagePath` 或本地路径

`日志请查-自动化/32 审批补传必填参数` 覆盖审批与补传的非 ID 必填参数：

1. 审批缺少 `decision` 必须被统一拒绝
2. 审批 `decision` 为空白字符串必须被统一拒绝
3. 补传领导批示缺少 `directiveAttachmentIds` 必须被统一拒绝
4. 补传领导批示 `directiveAttachmentIds=[]` 必须被统一拒绝
5. 参数失败不能误返回 `data=true`，且不泄露 `storagePath` 或本地路径

`日志请查-自动化/33 附件类型枚举错误` 覆盖附件类型非法枚举：

1. 上传附件时 `attachmentType=UNKNOWN` 必须被统一拒绝
2. 删除附件时 `attachmentType=UNKNOWN` 必须被统一拒绝
3. 上传失败不能返回新附件 ID
4. 删除失败不能误返回 `data=true`
5. 失败响应不能退化为 HTML/Whitelabel 错误页，且不泄露 `storagePath` 或本地路径

`日志请查-自动化/34 省厅分页参数校验` 覆盖省厅待办/已办分页入口的参数边界：

1. 省厅待办 `pageNum=0` 必须被拒绝
2. 省厅待办缺少 `pageSize` 必须被拒绝
3. 省厅待办 `sortType=3` 必须被拒绝
4. 省厅已办 `pageNum=0` 必须被拒绝
5. 省厅已办 `pageSize=0` 必须被拒绝
6. 省厅已办 `sortType=0` 必须被拒绝

`日志请查-自动化/35 请求体类型错误` 覆盖语法合法但类型错误的 JSON 请求体：

1. 创建申请接口收到数组请求体必须返回统一错误
2. 详情接口收到数组请求体必须返回统一错误
3. 审批接口收到数组请求体必须返回统一错误
4. 导出接口收到 `null` 请求体必须返回统一错误且不能返回 Excel 文件流
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/36 数字参数类型错误` 覆盖数字字段传入非数字字符串：

1. 详情 `id="not-a-number"` 必须返回统一错误
2. 撤回 `id="not-a-number"` 必须返回统一错误且不能误返回 `data=true`
3. 审批 `id="not-a-number"` 必须返回统一错误且不能误返回 `data=true`
4. 下载附件 `attachmentId=not-a-number` 必须返回统一错误且不能返回文件流
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/37 分页字段类型错误` 覆盖分页字段传入非数字字符串：

1. 地市分页 `pageNum="not-a-number"` 必须返回统一错误
2. 省厅待办分页 `pageSize="not-a-number"` 必须返回统一错误
3. 导出 `sortType="not-a-number"` 必须返回统一错误且不能返回 Excel 文件流
4. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/38 时间字段格式错误` 覆盖时间字段格式错误：

1. 创建申请 `logStartTime` 使用 `2026/05/01 00:00:00` 必须返回统一错误
2. 草稿保存 `logEndTime` 使用 `2026-05-02T00:00:00` 必须返回统一错误
3. 地市分页 `startCreatedTime` 格式错误必须返回统一错误
4. 省厅已办 `endCreatedTime` 格式错误必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/39 布尔字段类型错误` 覆盖 `urgent` 非布尔类型：

1. 创建申请 `urgent="not-a-boolean"` 必须返回统一错误
2. 地市分页 `urgent="not-a-boolean"` 必须返回统一错误
3. 省厅待办 `urgent="not-a-boolean"` 必须返回统一错误
4. 导出 `urgent="not-a-boolean"` 必须返回统一错误且不能返回 Excel 文件流
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/40 调取范围字段类型错误` 覆盖 `scopes` 非数组或数组元素非对象：

1. 创建申请 `scopes` 为对象必须返回统一错误
2. 创建申请 `scopes` 为字符串必须返回统一错误
3. 创建申请 `scopes=[1]` 必须返回统一错误
4. 草稿保存 `scopes` 为字符串必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/41 领导批示附件ID类型错误` 覆盖 `directiveAttachmentIds` 类型错误：

1. 审批 `directiveAttachmentIds=["not-a-number"]` 必须返回统一错误
2. 审批 `directiveAttachmentIds=[{"id":1}]` 必须返回统一错误
3. 补传领导批示 `directiveAttachmentIds=["not-a-number"]` 必须返回统一错误
4. 补传领导批示 `directiveAttachmentIds="not-an-array"` 必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/42 字符串字段空白校验` 覆盖字符串字段存在但为空白：

1. 创建申请 `targetOrgId="   "` 必须返回统一错误
2. 创建申请 `targetOrgName="   "` 必须返回统一错误
3. 创建申请 `requestReason="   "` 必须返回统一错误
4. 创建申请 `scopes[0].systemRange="   "` 必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/43 附件删除字段类型错误` 覆盖附件删除请求体字段类型错误：

1. 删除附件 `requestId="not-a-number"` 必须返回统一错误
2. 删除附件 `attachmentId="not-a-number"` 必须返回统一错误
3. 删除附件 `attachmentType={"name":"APPLICATION"}` 必须返回统一错误
4. 删除附件 `attachmentType=1` 必须返回统一错误
5. 删除失败不能误返回 `data=true`，且不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/44 整改续办字段类型错误` 覆盖整改续办请求字段类型错误：

1. 整改续办 `id="not-a-number"` 必须返回统一错误
2. 整改续办 `validDays="not-a-number"` 必须返回统一错误
3. 整改续办 `urgent="not-a-boolean"` 必须返回统一错误
4. 整改续办 `scopes="not-an-array"` 必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/45 排序字段类型错误` 覆盖 `sort` 非字符串数组：

1. 地市分页 `sort={"field":"createdTime"}` 必须返回统一错误
2. 省厅待办 `sort="createdTime"` 必须返回统一错误
3. 导出 `sort={"field":"createdTime"}` 必须返回统一错误且不能返回 Excel 文件流
4. 导出 `sort=[{"field":"createdTime"}]` 必须返回统一错误且不能返回 Excel 文件流
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/46 附件上传参数类型错误` 覆盖 multipart 上传请求参数类型错误：

1. 上传附件 `requestId=not-a-number` 必须返回统一错误
2. 上传附件 `requestId=` 空值必须返回统一错误
3. 上传附件 `attachmentType` 为空白必须返回统一错误
4. 上传附件 `attachmentType=application` 小写枚举必须返回统一错误
5. 上传失败不能返回附件 ID，且不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/47 导出页面类型字段类型错误` 覆盖导出 `pageType` 字段类型错误：

1. 导出 `pageType={"value":"CITY"}` 必须返回统一错误且不能返回 Excel 文件流
2. 导出 `pageType=["CITY"]` 必须返回统一错误且不能返回 Excel 文件流
3. 导出 `pageType=1` 必须返回统一错误且不能返回 Excel 文件流
4. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/48 创建字段显式null校验` 覆盖创建申请必填字段显式为 `null`：

1. 创建申请 `validDays=null` 必须返回统一错误
2. 创建申请 `urgent=null` 必须返回统一错误
3. 创建申请 `scopes=null` 必须返回统一错误
4. 创建申请 `scopes[0].logStartTime=null` 必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/49 分页字段显式null校验` 覆盖分页字段显式为 `null`：

1. 地市分页 `pageNum=null` 必须返回统一错误
2. 地市分页 `pageSize=null` 必须返回统一错误
3. 省厅待办分页 `pageNum=null` 必须返回统一错误
4. 导出 `pageSize=null` 必须返回统一错误且不能返回 Excel 文件流
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/50 审批补传字段显式null校验` 覆盖审批和补传关键字段显式为 `null`：

1. 审批 `id=null` 必须返回统一错误
2. 审批 `decision=null` 必须返回统一错误
3. 补传领导批示 `id=null` 必须返回统一错误
4. 补传领导批示 `directiveAttachmentIds=null` 必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/51 创建字符串字段显式null校验` 覆盖创建申请字符串必填字段显式为 `null`：

1. 创建申请 `targetOrgId=null` 必须返回统一错误
2. 创建申请 `targetOrgName=null` 必须返回统一错误
3. 创建申请 `requestReason=null` 必须返回统一错误
4. 创建申请 `scopes[0].systemRange=null` 必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/52 动作接口申请ID显式null校验` 覆盖动作接口 `id=null`：

1. 详情 `id=null` 必须返回统一错误
2. 删除草稿 `id=null` 必须返回统一错误
3. 提交草稿 `id=null` 必须返回统一错误
4. 撤回 `id=null` 必须返回统一错误
5. 作废 `id=null` 必须返回统一错误
6. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/53 附件删除字段显式null校验` 覆盖附件删除字段显式为 `null`：

1. 删除附件 `requestId=null` 必须返回统一错误
2. 删除附件 `attachmentId=null` 必须返回统一错误
3. 删除附件 `attachmentType=null` 必须返回统一错误
4. 删除失败不能误返回 `data=true`，且不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/54 草稿保存字段校验` 覆盖草稿保存阶段仍需校验的字段：

1. 草稿保存 `id=null` 必须返回统一错误
2. 草稿保存 `id="not-a-number"` 必须返回统一错误
3. 草稿保存 `scopes` 超过 10 条必须返回统一错误
4. 草稿保存 `scopes[0].logStartTime=null` 必须返回统一错误
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/55 附件下载参数边界` 覆盖附件下载 `attachmentId` 边界：

1. 下载附件 `attachmentId=` 空值必须返回统一错误且不能返回文件流
2. 下载附件 `attachmentId` 为空白必须返回统一错误且不能返回文件流
3. 下载附件 `attachmentId=0` 必须被拒绝且不能返回文件流
4. 下载附件 `attachmentId=-1` 必须被拒绝且不能返回文件流
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/56 列表筛选组合校验` 覆盖地市列表筛选语义：

1. 创建一条紧急待审核申请并保存申请编号、目标机构名称
2. 使用 `requestNo/status/urgent/targetOrgId` 正向组合筛选必须命中该申请
3. 使用相反 `urgent=false` 筛选不能命中该申请
4. 使用相反 `status=COMPLETED` 筛选不能命中该申请
5. 使用错误 `targetOrgId` 筛选不能命中该申请

`日志请查-自动化/57 省厅待办已办列表边界` 覆盖省厅待办/已办状态边界：

1. 创建一条待审核申请并保存申请编号
2. `PENDING` 申请必须出现在省厅待办
3. `PENDING` 申请不得出现在省厅已办
4. 省厅驳回后，`REJECTED` 申请必须出现在省厅已办
5. 省厅驳回后，`REJECTED` 申请不得出现在省厅待办

`日志请查-自动化/58 授权有效期精确校验` 覆盖审批授权有效期：

1. 创建 `validDays=7` 的非紧急申请
2. 上传领导批示并同意审批
3. 详情确认状态为 `COMPLETED/UPLOADED`
4. 授权结束时间与开始时间相差 7 天
5. 授权范围继承申请范围，且不处于待补传状态

`日志请查-自动化/59 请查单位组织范围校验` 覆盖请查单位组织范围：

1. `targetOrgId` 不存在时必须被业务拒绝
2. `targetOrgId` 不属于当前账号省级组织范围时必须被业务拒绝
3. `targetOrgId` 对应机构状态异常时必须被业务拒绝
4. 真实环境运行前需配置 `missingTargetOrgId/outOfScopeTargetOrgId/inactiveTargetOrgId`
5. 失败响应不能泄露异常类名、HTML/Whitelabel、本地路径或 `storagePath`

`日志请查-自动化/60 整改续办字段替换校验` 覆盖撤回后整改续办重提：

1. 创建待审核申请并保存原申请编号
2. 申请人撤回后基于同一 `id` 整改续办重新提交
3. 重提后仍沿用原申请编号，状态回到 `PENDING`
4. `validDays/urgent/requestReason/scopes` 必须替换为重提内容
5. 重提后无授权残留，且流程日志包含撤回和重新提交

`日志请查-自动化/61 领导批示附件重复ID校验` 覆盖领导批示附件重复绑定：

1. 创建非紧急待审核申请
2. 省厅上传一份 `LEADER_DIRECTIVE`
3. 审批时传入重复的 `directiveAttachmentIds`
4. 接口必须以“领导批示附件无效”拒绝
5. 失败后申请仍为 `PENDING/NONE`，无授权、无同意审批流程日志

`日志请查-自动化/62 领导批示附件类型错误业务校验` 覆盖附件类型错绑：

1. 创建非紧急待审核申请
2. 申请人上传 `APPLICATION` 申请附件
3. 省厅同意审批时将该申请附件作为领导批示绑定
4. 接口必须以“领导批示附件类型不正确”拒绝
5. 失败后申请仍为 `PENDING/NONE`，原申请附件仍是 `APPLICATION`

`日志请查-自动化/63 附件状态流转边界` 覆盖已完成状态下附件变更边界：

1. 创建非紧急待审核申请
2. 上传领导批示并同意审批，形成 `COMPLETED/UPLOADED`
3. 已完成申请继续上传 `APPLICATION` 必须被拒绝
4. 已完成且非待补传申请继续上传 `LEADER_DIRECTIVE` 必须被拒绝
5. 失败后完成态、授权和领导批示附件数量保持不变

`日志请查-自动化/64 导出过滤文件流一致性` 覆盖导出过滤和文件流差异：

1. 创建一条紧急待审核申请并保存申请编号
2. 使用 `requestNo/status/urgent/targetOrgId` 命中筛选导出城市侧列表
3. 使用相反 `status/urgent` 不命中筛选导出城市侧列表
4. 两次均应返回 Excel 文件流
5. 命中筛选导出的文件体积应大于不命中筛选导出，侧面验证过滤条件参与导出数据集

`日志请查-自动化/65 紧急待补充列表展示校验` 覆盖紧急待补充展示和列表分区：

1. 创建紧急待审核申请并校验 `待审核·紧急`
2. 省厅无领导批示同意后，详情展示 `COMPLETED/PENDING`
3. 详情状态展示为 `已完成·紧急·待补充`
4. 授权状态展示为 `有效，待补充领导批示`
5. 该申请不进入省厅待办，但必须出现在省厅已办

`日志请查-自动化/66 草稿附件流程日志校验` 覆盖草稿附件维护日志语义：

1. 初始化 `DRAFT` 草稿
2. 草稿状态上传 `APPLICATION` 附件成功且详情可见
3. 上传草稿附件不产生用户可见流程日志
4. 草稿状态删除该附件成功
5. 删除草稿附件后当前附件不展示，仍不产生用户可见流程日志

`日志请查-自动化/67 领导批示删除历史引用校验` 覆盖领导批示历史追溯：

1. 创建非紧急申请，上传领导批示并同意审批
2. 省厅删除已被审批流程引用的领导批示附件
3. 当前附件列表不再展示该领导批示
4. 审批流程日志仍保留领导批示附件 ID
5. 历史附件元信息展示 `deleted=true/downloadable=false`，删除后下载被拒绝

`日志请查-自动化/68 草稿导出排除校验` 覆盖草稿导出排除：

1. 初始化并保存有意义的 `DRAFT` 草稿
2. 草稿详情和本人地市列表可见，状态展示为 `草稿`
3. 按草稿申请编号导出城市侧列表返回 Excel 文件流
4. 草稿编号导出文件体积必须与空结果基线一致
5. 用于验证所有导出均排除 `DRAFT`

`日志请查-自动化/69 JSON接口拒绝multipart契约` 覆盖 JSON-only 接口契约：

1. `/create` 收到 multipart/form-data 必须统一拒绝，不得创建申请或附件
2. `/approve` 收到 multipart/form-data 必须统一拒绝
3. 审批 multipart 失败后原申请仍为 `PENDING/NONE`，无授权、无领导批示附件绑定
4. `/directive/supplement` 收到 multipart/form-data 必须统一拒绝
5. 失败响应不能退化为 HTML/Whitelabel、框架异常类名、本地路径或文件流

`日志请查-自动化/70 申请编号递增唯一校验` 覆盖申请编号生成：

1. 连续创建两条正式申请
2. 详情校验申请编号符合 `RZQCyyyyMMddNNNN`
3. 两条申请编号必须唯一
4. 同日期前缀下第二条流水必须大于第一条；跨日期时编号前缀必须前进

`日志请查-自动化/71 列表创建时间范围筛选校验` 覆盖地市列表创建时间过滤：

1. 创建申请并读取申请编号和创建时间
2. 宽时间范围叠加申请编号筛选必须命中
3. 未来开始时间筛选必须不命中
4. 过早结束时间筛选必须不命中

`日志请查-自动化/72 申请人姓名筛选校验` 覆盖申请人姓名过滤：

1. 创建申请时提交伪造申请人姓名字段
2. 详情回查必须展示服务端可信申请人姓名，不透出伪造值
3. 使用真实申请人姓名筛选地市列表必须命中
4. 使用不存在姓名筛选必须不命中

`日志请查-自动化/73 授权主体隔离校验` 覆盖授权主体与应用可见性隔离：

1. 创建非紧急申请并上传领导批示
2. 同意审批后申请人详情显示授权主体为原申请人
3. 省厅账号查看详情时授权主体仍为原申请人
4. 其他申请人不能查看该申请授权详情，失败响应不泄露编号、姓名、主体 ID 或本地路径

`日志请查-自动化/74 多领导批示绑定校验` 覆盖多个领导批示附件绑定：

1. 非紧急申请上传两个 `LEADER_DIRECTIVE` 附件
2. 同意审批时一次性绑定两个领导批示
3. 详情当前附件列表展示两个未删除领导批示
4. 审批流程日志的 `directiveAttachmentIds` 和历史附件元信息均保留两个引用

`日志请查-自动化/75 正式附件流程日志校验` 覆盖正式附件流程日志：

1. 正式 `PENDING` 申请上传 `APPLICATION` 附件后当前附件列表可见
2. 上传附件必须写入 `上传日志请查附件` 用户可见流程日志
3. 删除附件后当前附件列表移除该附件
4. 删除附件必须写入 `删除日志请查附件` 用户可见流程日志

`日志请查-自动化/76 附件扩展名大小写兼容` 覆盖允许扩展名大小写不敏感：

1. 创建正式申请
2. 上传大写 `.CSV` 申请附件必须成功
3. 详情展示原始文件名、附件类型和可下载状态
4. 附件元数据不泄露 `storagePath` 或本地路径

`日志请查-自动化/77 紧急携带批示直接同意校验` 覆盖紧急同意的有批示分支：

1. 创建紧急申请并在审批前上传领导批示
2. 同意审批时绑定该领导批示
3. 详情状态为 `COMPLETED/UPLOADED`，不显示待补充
4. 授权状态为 `有效` 且 `pendingDirective` 不为 true
5. 审批流程日志绑定领导批示，且申请不再进入省厅待办

`日志请查-自动化/78 空草稿与列表展示校验` 覆盖空草稿列表边界：

1. 初始化无意义空草稿，详情本人可访问但无业务字段、无附件、无流程日志
2. 空草稿按申请编号和 `DRAFT` 状态查询时不进入地市列表
3. 保存有意义字段后，同一草稿进入本人地市列表并展示 `草稿`
4. 用例结束删除该草稿，避免真实环境残留

`日志请查-自动化/79 列表有效期展示校验` 覆盖列表有效期展示：

1. 创建 `PENDING` 申请后，地市列表 `validPeriodDisplay` 展示申请有效天数
2. 同意审批后，地市列表展示实际授权开始和结束时间
3. 已完成申请的 `validPeriodDisplay` 不再退化为 `N天`

`日志请查-自动化/80 省厅已办终态排除校验` 覆盖省厅已办状态边界：

1. `WITHDRAWN` 已撤回申请不进入省厅已办
2. `VOIDED` 已作废申请不进入省厅已办
3. 与既有 `COMPLETED/REJECTED` 已办正向用例互补

`日志请查-自动化/81 省厅已办导出权限补充` 覆盖省厅已办导出权限：

1. 准备一条省厅已办可见的紧急已完成申请
2. 申请人使用 `PROVINCE_DONE` 页面类型导出必须被拒绝
3. 拒绝响应不能返回 Excel 文件流，且不能泄露申请编号、业务内容或本地路径
4. 省厅账号使用同样条件导出仍返回 Excel 文件流

`日志请查-自动化/82 非草稿删除草稿拒绝校验` 覆盖删除草稿状态边界：

1. 准备一条 `PENDING` 待审核申请
2. 使用 `/draft/delete` 删除该非草稿申请必须被拒绝
3. 删除失败后详情仍为 `PENDING`，流程日志不追加删除草稿动作
4. 删除失败后省厅待办仍可见该申请

`日志请查-自动化/83 撤回驳回附件维护校验` 覆盖撤回/驳回状态附件维护：

1. `WITHDRAWN` 已撤回申请仍允许申请人本人上传和删除 `APPLICATION` 附件
2. `REJECTED` 已驳回申请仍允许申请人本人上传和删除 `APPLICATION` 附件
3. 删除后当前详情附件列表不再展示已删除附件，且不泄露本地存储路径

`日志请查-自动化/84 导出默认页面类型校验` 覆盖导出默认语义：

1. 创建一条可导出的地市侧待审核申请
2. `/export` 不传 `pageType` 时默认按 `CITY` 地市列表权限导出
3. 同一筛选条件下，不传 `pageType` 与显式 `CITY` 返回同类 Excel 文件流

`日志请查-自动化/85 申请人账号单位筛选校验` 覆盖申请人账号/单位筛选：

1. 创建一条本人申请并保存申请编号
2. 使用 `applicantAccountId` 正向筛选命中本人申请
3. 使用 `applicantOrgId` 正向筛选命中本人申请
4. 筛选条件只收窄本人可见范围，不依赖客户端伪造主体字段

`日志请查-自动化/86 省厅申请附件维护拒绝校验` 覆盖申请附件主体权限：

1. 申请人创建待审核申请并上传 `APPLICATION` 附件
2. 省厅账号上传 `APPLICATION` 附件必须被拒绝
3. 省厅账号删除 `APPLICATION` 附件必须被拒绝
4. 拒绝后原申请附件仍未删除、仍可下载，且响应不泄露本地存储路径

`日志请查-自动化/87 作废态申请附件维护拒绝校验` 覆盖作废态附件维护边界：

1. 申请人创建待审核申请并上传 `APPLICATION` 附件
2. 申请撤回后作废为 `VOIDED`
3. `VOIDED` 作废态继续上传 `APPLICATION` 附件必须被拒绝
4. `VOIDED` 作废态继续删除既有 `APPLICATION` 附件必须被拒绝
5. 拒绝后申请仍为 `VOIDED`，既有附件仍未删除

`日志请查-自动化/88 附件删除错配与重复删除校验` 覆盖附件删除三元组一致性：

1. 创建申请 A 并上传 `APPLICATION` 附件
2. 使用申请 B 的 `requestId` 删除申请 A 的附件必须被拒绝
3. 使用错误 `attachmentType=LEADER_DIRECTIVE` 删除申请 A 的 `APPLICATION` 附件必须被拒绝
4. 错配删除失败后原附件仍未删除、仍可下载
5. 正确删除后重复删除必须被拒绝，且当前附件列表不再展示已删除附件

`日志请查-自动化/89 领导批示上传状态边界` 覆盖领导批示附件上传状态约束：

1. `WITHDRAWN` 已撤回申请不能继续上传 `LEADER_DIRECTIVE`
2. `REJECTED` 已驳回申请不能继续上传 `LEADER_DIRECTIVE`
3. `VOIDED` 已作废申请不能继续上传 `LEADER_DIRECTIVE`
4. 失败后申请状态、`directiveStatus` 和当前领导批示附件列表保持不变
5. 响应不泄露本地存储路径

`日志请查-自动化/90 补传领导批示附件业务校验` 覆盖紧急待补传附件绑定规则：

1. 先形成 `COMPLETED + directiveStatus=PENDING` 的紧急待补传申请
2. 补传其他申请的 `LEADER_DIRECTIVE` 必须被拒绝
3. 补传当前申请的 `APPLICATION` 申请附件必须被拒绝
4. 补传当前申请已删除的 `LEADER_DIRECTIVE` 必须被拒绝
5. 每次失败后申请仍保持待补传，授权起止时间不变，且不产生成功补传流程日志

`日志请查-自动化/91 不存在领导批示附件业务校验` 覆盖领导批示附件存在性规则：

1. 非紧急待审批申请使用不存在的 `directiveAttachmentIds` 同意审批必须被拒绝
2. 审批失败后申请仍为 `PENDING + directiveStatus=NONE`，无授权、无成功同意流程日志
3. 紧急待补传申请使用不存在的 `directiveAttachmentIds` 补传必须被拒绝
4. 补传失败后申请仍为 `COMPLETED + directiveStatus=PENDING`，授权起止时间不变，且不产生成功补传流程日志

## 环境变量

模板文件位于 `environments/Log Request Automation Template.yml`。本机可直接复制或使用 `environments/Log Request Automation Local.yml`，该文件已加入 `.gitignore`，用于填写真实 token 和环境专属组织 ID。

运行真实环境前至少需要提供：

- `baseUrl`：服务地址，例如 `https://example-host:18080/ilda`
- `applicantToken`：申请人登录 token
- `targetOrgId`：被调取日志的目标机构 ID
- `targetOrgName`：兼容字段，后端应按 `targetOrgId` 反查覆盖
- `missingTargetOrgId`：不存在的机构 ID，用于组织范围负向校验
- `outOfScopeTargetOrgId`：真实存在但不属于当前账号省级组织范围的机构 ID
- `inactiveTargetOrgId`：真实存在但状态异常的机构 ID
- `missingDirectiveAttachmentId`：不存在的领导批示附件 ID，用于附件存在性负向校验；默认值可按环境覆盖
- `provinceToken`：省厅账号 token，用于省厅待办/已办、审批和权限隔离用例
- `otherApplicantToken`：其他申请人 token，用于越权访问和越权操作用例
- `applicantAccountId`：`applicantToken` 对应的真实账号 ID，用于申请人账号筛选校验
- `applicantOrgId`：`applicantToken` 对应的真实组织 ID，用于申请人单位筛选校验
- `validDays`：有效天数
- `requestReason`：申请原因
- `systemRange`：调取系统范围
- `logStartTime`、`logEndTime`：日志时间范围

真实 token、账号密码、Cookie 不要写入可提交的集合文件；请在 Bruno 客户端本地环境中填写，或写入已忽略的 `environments/Log Request Automation Local.yml`。

## 命令行运行

建议先用本地环境文件跑单组冒烟。在集合目录执行：

```bash
bru run '日志请查-自动化/01 草稿完整提交链路' \
  -r \
  --env-file 'environments/Log Request Automation Local.yml' \
  --insecure \
  --bail
```

确认账号、组织和附件存储环境可用后，再运行完整自动化目录：

```bash
bru run '日志请查-自动化' \
  -r \
  --env-file 'environments/Log Request Automation Local.yml' \
  --insecure \
  --bail
```

如果不想改本地环境文件，也可以继续使用模板文件并通过命令行覆盖变量：

```bash
bru run '日志请查-自动化' \
  -r \
  --env-file 'environments/Log Request Automation Template.yml' \
  --env-var baseUrl='https://example-host:18080/ilda' \
  --env-var applicantToken='<真实申请人token>' \
  --env-var provinceToken='<真实省厅token>' \
  --env-var otherApplicantToken='<真实其他申请人token>' \
  --env-var applicantAccountId='<真实申请人账号ID>' \
  --env-var applicantOrgId='<真实申请人组织ID>' \
  --env-var targetOrgId='<真实请查单位ID>' \
  --env-var missingTargetOrgId='<不存在机构ID>' \
  --env-var outOfScopeTargetOrgId='<越界机构ID>' \
  --env-var inactiveTargetOrgId='<状态异常机构ID>' \
  --env-var missingDirectiveAttachmentId='999999999' \
  --insecure \
  --bail
```

## 断言口径

- HTTP 状态码必须为 `200`
- 统一响应体 `code` 必须为 `200`
- 成功响应体 `reason` 兼容 `success` 或当前 Java 服务返回的 `成功`
- 权限和参数负向用例必须返回非 `200` 业务码，并且响应中不能泄露本地文件路径、`storagePath` 或伪造主体字段
- 附件下载和列表导出是二进制响应，只断言文件头、文件名和响应体大小，不按 JSON `Result<T>` 断言
- 草稿 ID、申请编号、状态、调取范围、流程日志和地市列表可见性必须和业务预期一致

下载、导出等二进制接口后续应按文件响应断言，不按 JSON `Result<T>` 断言。
