# Vidu S1：ModelScope 仓库无法用于本地 ComfyUI 部署

> 结论：这是一次有效的 SOP 负向测试，不是部署成功记录。目标仓库没有本地模型权重、推理实现、许可证或 Vidu S1 的 ComfyUI 接入路径，因此在 N0 正确标记为 `BLOCKED`，没有进入安装阶段。

## 1. 记录信息

| 字段 | 内容 |
|---|---|
| 执行日期 | 2026-07-13 |
| deployment_record_id | `VIDU-S1-20260713-001` |
| candidate_id | `VIDU-S1-MODELSCOPE-LOCAL-001` |
| deployment_id | 未分配；N1 未通过 |
| 请求目标 | 将 Vidu S1 作为本地模型接入 ComfyUI，并用实际部署测试 SOP |
| 当前节点 | N0 需求定义与接入路径确认 |
| 状态 | `BLOCKED` |

结构化记录见 [deployment_record.yaml](deployment_record.yaml)。

## 2. 候选仓库身份

- 来源：`https://www.modelscope.cn/ViduAI/Vidu-S1.git`
- 分支：`master`
- commit：`c8e5248ac8b3b1f3794d463b09f8d1b4d389f7cc`
- 执行时工作区：干净
- ModelScope 模型页：<https://modelscope.cn/models/ViduAI/Vidu-S1>
- ModelScope 官方元数据：<https://modelscope.cn/api/v1/models/ViduAI/Vidu-S1>
- 执行时选定字段快照：[modelscope_metadata_snapshot.json](modelscope_metadata_snapshot.json)

官方元数据显示 `SupportDeployment=0`、`SupportApiInference=false`、框架为 `other`，且没有填写模型许可证。它没有声明可由 ModelScope 本地部署。

## 3. 仓库实际文件

| 文件 | 字节数 | SHA-256 | 判定 |
|---|---:|---|---|
| `.gitattributes` | 2,166 | `93AA9372B6C9E5E64D89BD9E46CB5902F9C9769066F0F69A110EE76CA1ED73A9` | 通用 LFS 规则，不是模型 |
| `configuration.json` | 50 | `AC947913BCF6D31AA1FDDC88D6A001C01370642ABE6DEBA0523C084C14D1CB1E` | 仅声明 `framework=other`、`task=multimodal-dialogue` |
| `README.md` | 3,827 | `11F04234043D93A0D4B03DB393A5179CA5A141A8F680FD1D0505505885C3B622` | 产品说明与外部 API 链接 |
| `飞书20260708-093415.mp4` | 156,036,246 | `B93D8AD01CF10FD303EA9C13358E3AC6059226D95E5B23E3A186A0E1D2ECBFB5` | 教程视频，不是 checkpoint |

仓库共 4 个跟踪文件。不存在 checkpoint、Safetensors、PyTorch/ONNX 模型、推理脚本、依赖锁、Dockerfile、启动入口或本地 smoke test。

`.gitattributes` 中出现 `.safetensors`、`.pt`、`.onnx` 等匹配规则，只说明这些扩展名若存在时应由 LFS 管理，不能证明仓库实际包含模型文件。

## 4. N0 接入路径核验

| 必需项 | 实际结果 | 状态 |
|---|---|---|
| 不可变模型身份 | 只有仓库 commit，没有模型 revision/权重身份 | FAIL |
| 主模型及全部组件 | 未公开 | FAIL |
| 本地推理代码与依赖 | 未公开 | FAIL |
| 模型/权重许可证 | 未提供；论文引用不等于许可证 | FAIL |
| 输入输出 schema 与最小样例 | 只有概念性能力描述 | FAIL |
| ComfyUI 原生 loader/workflow | 未发现 Vidu S1 支持 | FAIL |
| 官方 custom node | 未发现 | FAIL |
| dtype、量化、算子和显存要求 | 未披露 | FAIL |
| 可测本地验收标准 | 官方没有给出可复现本地测试配置 | FAIL |

因此无法形成 N0 要求的组件、节点、算子和验收契约，也无法进入 N1 求硬件—软件版本交集。

## 5. 官方可用路径不是本地模型部署

Vidu 官方公开的是托管服务集成：

- 官方 API 仓库：<https://github.com/shengshu-ai/vidu-s1-api>
- 中文说明：<https://github.com/shengshu-ai/vidu-s1-api/blob/main/README.zh-CN.md>
- 使用 `POST /live/v1/lives` 创建会话；
- 使用阿里云 RTC 传输实时音视频；
- 使用持续 WebSocket 承载控制指令；
- 请求需要 Vidu API Key。

该仓库的 MIT License 只覆盖 API 示例/Skill 代码，不是 Vidu S1 权重许可证。调用托管 API 也不能证明模型在目标设备上运行。

## 6. ComfyUI “Vidu 节点”不能外推到 S1

ComfyUI 官方存在 Vidu Partner API 节点，但当前源码支持的是 `viduq1`、`viduq2-*`、`viduq3-*` 等普通视频任务接口：

- 官方源码：<https://github.com/Comfy-Org/ComfyUI/blob/master/comfy_api_nodes/nodes_vidu.py>
- 官方节点文档示例：<https://docs.comfy.org/built-in-nodes/Vidu3TextToVideoNode>

源码中没有 Vidu S1、`/live/v1/lives`、RTC 或实时控制 WebSocket。品牌相同不能作为模型系列兼容证据；目前也未发现 Vidu 官方的 S1 custom node。

## 7. 节点结论

```text
N0 = BLOCKED
N1-N5 = NOT_STARTED
deployment_id = 未分配
未下载 ComfyUI
未安装 Python/框架/节点依赖
未把 MP4 放入任何模型目录
```

虽然执行期间做过只读设备基线采集，但它没有用于 N1 放行，也不构成 Vidu S1 兼容证据。没有模型组件和推理要求时，设备再强也无法求出有效版本交集。

## 8. 解除阻断所需材料

若目标仍是本地 ComfyUI 部署，必须先由官方提供：

1. 权重下载地址、全部组件、不可变 revision、字节数和发布方 hash；
2. 本地推理代码、依赖锁、启动方式和最小样例；
3. 模型/权重许可证和使用范围；
4. 支持的 OS、设备后端、驱动、框架、精度、算子和内存要求；
5. 输入输出 schema 与可复现验收基线；
6. 官方 ComfyUI workflow/custom node，或足以开发 wrapper 的本地推理接口。

材料齐全后应创建新的 `candidate_id`，从 N0 重新执行。

## 9. 可选路线纠正

如果业务接受云服务，可另建“Vidu S1 远程 API/RTC 接入”部署记录，验收 API 鉴权、数据与隐私、网络、计费、限流、会话时长、RTC/WebSocket 稳定性、服务可用性和供应商回退。

把 S1 API 封装成 ComfyUI remote-service wrapper 需要独立设计；持续 RTC/WebSocket 会话与常规“一次节点执行、返回一个文件”的 workflow 模型并不等价。即使 wrapper 成功，也只能称为远程服务接入，不能称为 Vidu S1 本地模型部署。

## 10. 本次 SOP 测试发现

SOP 成功阻止了以下错误动作：

- 把 `git clone` 成功当成模型已下载；
- 把 149 MiB 的 MP4/LFS 对象当成模型权重；
- 因 ComfyUI 支持其他 Vidu 系列而推断支持 Vidu S1；
- 在模型组件、许可证和推理实现未知时提前安装运行栈；
- 把云 API 调用结果归因于本机模型兼容。

根据本案例，SOP 的 N0 已补充：仓库类型和实际文件检查、本地推理与远程 API 边界、LFS/演示文件识别，以及云 API 接入的额外验收要求。
