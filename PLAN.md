# Scheduled Tasks Enhancement Plan

## Overview

改进定时任务功能：支持编辑、新增工作目录选择、复用 sessions 组件。

参考官方 Claude Code 桌面端 APP 的 "New scheduled task" 对话框实现。

---

## Phase 1: Data Model Extension

### 1.1 扩展 CronTask 类型 (`src/utils/cronTasks.ts`)

在现有 `CronTask` 类型中新增以下字段：

```typescript
type CronTask = {
  // ... existing fields ...
  
  /** Human-readable task name (e.g. "daily-code-review") */
  name?: string
  /** Task description (e.g. "Review yesterday's commits") */
  description?: string
  /** Working directory for the task execution */
  folder?: string
  /** Model to use (e.g. "claude-opus-4-6", "claude-sonnet-4-6") */
  model?: string
  /** Permission mode: "ask" | "auto-accept" | "plan" | "bypass" */
  permissionMode?: string
  /** Whether to use git worktree for execution */
  worktree?: boolean
  /** Schedule frequency for UI display: "manual" | "hourly" | "daily" | "weekdays" | "weekly" */
  frequency?: string
  /** Time string for scheduled execution (e.g. "09:00") - UI helper for daily/weekdays/weekly */
  scheduledTime?: string
}
```

**向后兼容**：所有新字段均为 optional，旧数据无需迁移。

### 1.2 更新存储 read/write (`src/utils/cronTasks.ts`)

- `readCronTasks()`: 在读取时保留新字段（name, description, folder, model, permissionMode, worktree, frequency, scheduledTime）
- `writeCronTasks()`: 在写入时包含新字段（仍然 strip `durable` 和 `agentId`）
- `addCronTask()`: 扩展函数签名接收新字段

### 1.3 新增 `updateCronTask()` 函数 (`src/utils/cronTasks.ts`)

```typescript
export async function updateCronTask(
  id: string,
  updates: Partial<Omit<CronTask, 'id' | 'createdAt'>>,
  dir?: string,
): Promise<boolean>
```

- 查找并更新内存中（session store）或磁盘上的任务
- 如果 cron 表达式变化，重新验证
- 返回是否找到并更新成功

---

## Phase 2: CronUpdateTool (`src/tools/ScheduleCronTool/CronUpdateTool.ts`)

### 2.1 创建新工具文件

参照 `CronCreateTool.ts` 和 `CronDeleteTool.ts` 的模式：

```typescript
const inputSchema = z.strictObject({
  id: z.string().describe('Job ID returned by CronCreate.'),
  cron: z.string().optional().describe('New cron expression'),
  prompt: z.string().optional().describe('New prompt'),
  name: z.string().optional().describe('New name'),
  description: z.string().optional().describe('New description'),
  folder: z.string().optional().describe('New working directory'),
  model: z.string().optional().describe('New model'),
  permissionMode: z.string().optional().describe('New permission mode'),
  worktree: z.boolean().optional().describe('New worktree setting'),
  recurring: z.boolean().optional().describe('New recurring setting'),
  frequency: z.string().optional().describe('New frequency'),
  scheduledTime: z.string().optional().describe('New time'),
})
```

### 2.2 更新 prompt.ts

- 新增 `CRON_UPDATE_TOOL_NAME = 'CronUpdate'`
- 添加 description 和 prompt 构建函数

### 2.3 更新 UI.tsx

- 新增 `renderUpdateToolUseMessage()` 和 `renderUpdateResultMessage()`

---

## Phase 3: UI Components (复用 sessions 组件)

**核心原则**：所有可复用的组件直接 import 使用，不复制代码。sessions 那边改了，这边自动生效。

### 3.1 频率到 Cron 表达式映射工具 (`src/utils/cronFrequency.ts`)

新建工具函数，在 UI 友好的频率设置和 cron 表达式之间互转：

```typescript
// Frequency → Cron
function frequencyToCron(frequency: string, time?: string): string
// "daily" + "09:00" → "0 9 * * *"
// "hourly" → "0 * * * *"
// "weekdays" + "09:00" → "0 9 * * 1-5"
// "weekly" + "09:00" → "0 9 * * 1"

// Cron → Frequency (best effort)
function cronToFrequency(cron: string): { frequency: string; time?: string }
```

### 3.2 定时任务向导 (`src/components/scheduled-tasks/ScheduledTaskWizard.tsx`)

使用现有 **Wizard 框架** (`src/components/wizard/`)，创建多步骤向导：

```typescript
type ScheduledTaskWizardData = {
  name: string
  description: string
  prompt: string
  model?: string
  permissionMode?: string
  folder?: string
  worktree?: boolean
  frequency: string       // "manual" | "hourly" | "daily" | "weekdays" | "weekly"
  scheduledTime?: string  // "09:00"
  cron?: string           // 最终生成的 cron 表达式
}

// 支持 create 和 edit 两种模式
type Props = {
  mode: 'create' | 'edit'
  initialData?: Partial<ScheduledTaskWizardData>  // edit 模式下预填充
  taskId?: string                                  // edit 模式下的任务 ID
  onComplete: (data: ScheduledTaskWizardData) => void
  onCancel: () => void
}
```

**向导步骤**（每步复用 sessions 组件）：

| Step | 组件 | 复用来源 |
|------|------|---------|
| 1. NameStep | `TextInput` | `src/components/TextInput.tsx` |
| 2. DescriptionStep | `TextInput` + external editor | `src/components/agents/new-agent-creation/wizard-steps/DescriptionStep.tsx` 的模式 |
| 3. PromptStep | `TextInput` + external editor | `src/components/agents/new-agent-creation/wizard-steps/PromptStep.tsx` 的模式 |
| 4. ModelStep | `ModelSelector` | **直接复用** `src/components/agents/ModelSelector.tsx` |
| 5. PermissionStep | `Select` | **直接复用** `src/components/CustomSelect/select.tsx` |
| 6. FolderStep | `Select` / `FuzzyPicker` | **复用** `src/components/CustomSelect/select.tsx` + `sessionStorage.ts` 的项目发现 |
| 7. ScheduleStep | `Select` + `TextInput` | **直接复用** `src/components/CustomSelect/select.tsx` |
| 8. ConfirmStep | Summary display | 使用 `Dialog` + `Text` 显示摘要 |

### 3.3 各步骤详细设计

#### Step 1: NameStep

```tsx
// 直接使用 TextInput，参照 DescriptionStep 模式
function NameStep(): ReactNode {
  const { goNext, goBack, updateWizardData, wizardData } = useWizard<ScheduledTaskWizardData>()
  // TextInput with validation: name is required
}
```

#### Step 4: ModelStep（复用 ModelSelector）

```tsx
import { ModelSelector } from '../../agents/ModelSelector.js'

function ModelStep(): ReactNode {
  const { goNext, updateWizardData, wizardData } = useWizard<ScheduledTaskWizardData>()
  return (
    <WizardDialogLayout subtitle="Select Model">
      <ModelSelector
        initialModel={wizardData.model}
        onComplete={(model) => {
          updateWizardData({ model })
          goNext()
        }}
        onCancel={goBack}
      />
    </WizardDialogLayout>
  )
}
```

#### Step 5: PermissionStep（复用 Select）

```tsx
import { Select } from '../../CustomSelect/select.js'

const permissionOptions = [
  { label: 'Ask permissions', value: 'ask', description: 'Always ask before making changes' },
  { label: 'Auto accept edits', value: 'auto-accept', description: 'Automatically accept all file edits' },
  { label: 'Plan mode', value: 'plan', description: 'Create a plan before making changes' },
  { label: 'Bypass permissions', value: 'bypass', description: 'Accepts all permissions', disabled: false },
]
```

#### Step 6: FolderStep（复用 sessionStorage 项目发现）

```tsx
import { loadAllProjectsMessageLogs } from '../../utils/sessionStorage.js'
// 或者直接读取 GlobalConfig.projects 获取最近项目列表

function FolderStep(): ReactNode {
  // 1. 从 GlobalConfig.projects 获取已知项目路径
  // 2. 使用 Select 展示 "Recent" 项目列表
  // 3. 最后一项 "Choose a different folder" 允许手动输入路径
}
```

#### Step 7: ScheduleStep（频率 + 时间）

```tsx
const frequencyOptions = [
  { label: 'Manual', value: 'manual' },
  { label: 'Hourly', value: 'hourly' },
  { label: 'Daily', value: 'daily' },
  { label: 'Weekdays', value: 'weekdays' },
  { label: 'Weekly', value: 'weekly' },
]

function ScheduleStep(): ReactNode {
  // 1. 选择频率 (Select 组件)
  // 2. 如果是 daily/weekdays/weekly，显示时间输入 (TextInput，格式 HH:MM)
  // 3. 使用 frequencyToCron() 转换为 cron 表达式
}
```

#### Step 8: ConfirmStep

```tsx
function ConfirmStep(): ReactNode {
  // 显示所有配置的摘要
  // Enter 确认创建/更新
  // Esc 返回上一步
}
```

---

## Phase 4: Integration

### 4.1 新增 /schedule-local Skill (`src/skills/bundled/scheduleLocal.ts`)

或修改现有 `/schedule` skill，添加本地定时任务的向导流程：

```typescript
// 用户输入 /schedule 时:
// 1. 如果没参数 → 显示 ScheduledTaskWizard (create 模式)
// 2. 如果参数是 task ID → 显示 ScheduledTaskWizard (edit 模式，预填充)
// 3. 如果参数是 "list" → 调用 CronList
```

### 4.2 注册 CronUpdateTool

在工具注册表中添加 CronUpdateTool：

- 更新 `src/tools/ScheduleCronTool/` 导出
- 确保工具在 `isKairosCronEnabled()` 条件下启用

### 4.3 更新 CronListTool 输出

在列表输出中包含新字段（name, description, folder, model 等），方便用户识别任务。

---

## Phase 5: Testing

### 5.1 单元测试

- `cronTasks.ts`: 测试 updateCronTask() 的各种场景（内存任务、磁盘任务、不存在的 ID）
- `cronFrequency.ts`: 测试频率到 cron 的双向转换
- `CronUpdateTool.ts`: 测试验证逻辑（无效 ID、权限检查）

### 5.2 集成测试

- 验证 wizard 创建的任务能被 scheduler 正确执行
- 验证编辑后的任务能正确更新 nextFireAt

---

## File Changes Summary

| File | Change Type | Description |
|------|------------|-------------|
| `src/utils/cronTasks.ts` | **Modified** | 扩展 CronTask 类型，新增 updateCronTask()，更新 read/write/add |
| `src/utils/cronFrequency.ts` | **New** | 频率 ↔ Cron 表达式转换工具 |
| `src/tools/ScheduleCronTool/CronUpdateTool.ts` | **New** | 编辑定时任务工具 |
| `src/tools/ScheduleCronTool/prompt.ts` | **Modified** | 新增 CronUpdate 的 name/description/prompt |
| `src/tools/ScheduleCronTool/UI.tsx` | **Modified** | 新增 update 的 render 函数 |
| `src/components/scheduled-tasks/ScheduledTaskWizard.tsx` | **New** | 定时任务向导（复用 sessions 组件） |
| `src/components/scheduled-tasks/steps/NameStep.tsx` | **New** | 名称输入步骤 |
| `src/components/scheduled-tasks/steps/DescriptionStep.tsx` | **New** | 描述输入步骤 |
| `src/components/scheduled-tasks/steps/PromptStep.tsx` | **New** | Prompt 输入步骤 |
| `src/components/scheduled-tasks/steps/ModelStep.tsx` | **New** | 模型选择步骤（复用 ModelSelector） |
| `src/components/scheduled-tasks/steps/PermissionStep.tsx` | **New** | 权限模式步骤（复用 Select） |
| `src/components/scheduled-tasks/steps/FolderStep.tsx` | **New** | 工作目录步骤（复用 Select + 项目发现） |
| `src/components/scheduled-tasks/steps/ScheduleStep.tsx` | **New** | 频率+时间步骤（复用 Select） |
| `src/components/scheduled-tasks/steps/ConfirmStep.tsx` | **New** | 确认步骤 |
| `src/skills/bundled/scheduleLocal.ts` | **New or Modified** | /schedule skill 集成向导 |
| `src/hooks/useScheduledTasks.ts` | **Modified** | 支持 folder/model/permissionMode/worktree 在任务执行时生效 |

## Key Reuse Points (复用列表)

确保以下组件是直接 import 复用，**不是复制代码**：

1. **`WizardProvider`** - `src/components/wizard/WizardProvider.tsx`
2. **`WizardDialogLayout`** - `src/components/wizard/WizardDialogLayout.tsx`
3. **`useWizard`** - `src/components/wizard/useWizard.ts`
4. **`WizardNavigationFooter`** - `src/components/wizard/WizardNavigationFooter.tsx`
5. **`ModelSelector`** - `src/components/agents/ModelSelector.tsx`
6. **`Select`** - `src/components/CustomSelect/select.tsx`
7. **`TextInput`** - `src/components/TextInput.tsx`
8. **`Dialog`** - `src/components/design-system/Dialog.tsx`
9. **`FuzzyPicker`** - `src/components/design-system/FuzzyPicker.tsx`（如 FolderStep 需要搜索）
10. **`useKeybinding`** - `src/hooks/useKeybinding.ts`
11. **Session Project Discovery** - `src/utils/sessionStorage.ts` (loadAllProjectsMessageLogs)
12. **GlobalConfig.projects** - `src/utils/config.ts`（最近项目列表）

## Implementation Order

1. Phase 1 (Data Model) → 2. Phase 2 (CronUpdateTool) → 3. Phase 3 (UI) → 4. Phase 4 (Integration) → 5. Phase 5 (Testing)

每个 Phase 完成后做一次代码审查。
