# PPTAgent 技术分析报告
本报告详细介绍了 PPTAgent 如何整合多种 LLM 提供商，以及如何将大模型的推理结果（tokens）逐步转化为高保算的 PPT 文件。
## 1. LLM Providers 的集成方式
PPTAgent 在 `pptagent/llms.py` 中实现了一个统一的封装类 `LLM` 和 `AsyncLLM`。
-   **OpenAI 兼容性**：核心实现基于 `openai` 官方 SDK，通过设置 `base_url` 和 `api_key` 来兼容所有支持 OpenAI 协议的服务商。
-   **支持的服务商**：
    -   **OpenAI**: 默认后端。
    -   **DeepSeek**: 在 `format_message` 中有特殊逻辑，会将 `system_message` 合并到 `user_message` 中以适应其推理偏好。
    -   **Anthropic / Gemini / Ollama**: 通过各自的 OpenAI 兼容网关（如 One API, LiteLLM 或官方提供的兼容接口）接入。
-   **多模态能力 (VLM)**：支持将图片以 Base64 编码的形式嵌入到提示词中，用于演示文稿的视觉识别和图片重写。
---
## 2. 从 Tokens 到 PPT 的转化流程
PPTAgent 采用“两阶段、基于编辑”的生成范式，将生成过程分解为多个 Agent 的协同工作。
### 第一阶段：内容规划 (Prompting & Induction)
1.  **Tokens 变为 Markdown (JSON Outline)**:
    -   `planner` Agent 接收文档摘要，输出符合 `Outline` schema 的 JSON 数据。这描述了 PPT 的章节结构和每页的主题。
    -   `content_organizer` Agent 从原始文档中提取出与特定 Slide 主题相关的关键点（Key Points）。
### 第二阶段：页面生成 (Action Generation & Execution)
1.  **布局选择 (Layout Selection)**:
    -   `layout_selector` 根据 Slide 内容从参考模板中挑选最合适的 Layout（如“标题+文本”、“标题+两图”等）。
2.  **内容填充 (Content Generation)**:
    -   `editor` Agent 根据选定 Layout 的 **Content Schema**，将提取的关键点重写为符合幻灯片字数限制和格式要求的文本/图片路径。
3.  **动作生成 (Markdown 变为 PPT 动作)**:
    -   `coder` Agent 接收 **Template Slide 的 HTML 表示** 以及 `editor` 输出的 **Command List**（新旧内容的差异）。
    -   `coder` 输出 Python API 调用序列（如 `replace_paragraph`, `clone_paragraph`, `replace_image`）。
4.  **物理执行**:
    -   这些 API 调用最终作用于 `python-pptx` 封装的幻灯片对象，完成最终的渲染。
---
## 3. 模板文件的交互逻辑
模板在 Stage I (Induction) 被分析，生成以下关键文件：
-   **`source.pptx`**: 作为视觉和结构参考的底稿。其内容被转化为 HTML 以便编码器（Coder）理解。
-   **`slide_induction.json`**: 存储了每个 Layout 的元数据：
    -   **Functional Keys**: 标识哪些页是标题页、目录页、过场页或结束页。
    -   **Content Schema**: 定义了 Layout 中包含哪些元素（Title, Body_Text, Image_Placeholder）及其字符上限、默认数量。`editor` Agent 必须严格遵守此 Schema 才能确保内容能填入。
-   **`image_stats.json`**: 包含对模板中占位图片的 **Vision Caption**。当 LLM 需要生成或选择新图片时，它通过这些描述来理解占位符原本代表的视觉语义（例如“公司 Logo 占位符”或“全屏背景图”）。
-   **`description.txt`**: 提供关于模板整体风格或特定页面的文本描述，帮助 LLM 在 `layout_selector` 阶段做出更具上下文感知的决策。
---
## 4. PPT 修改示例与 Prompt 演示
以下是一个模拟对话，展示了如何通过 LLM 交互来修改 PPT 中的特定页面。
### 场景：将“技术架构”页修改为具有“商业价值”导向的内容。
#### Agent 1: Editor (内容重写)
**System Prompt**: 你是一位资深的商业文案专家，擅长将技术细节转化为商业价值。
**User Prompt**:
```text
===Current Slide Context===
Title: 云原生架构技术细节
Content: 
- 容器化部署 (Docker/K8s)
- 微服务拆分 (gRPC/Rest)
- 自动扩缩容机制
===Schema===
{
  "Title": {"type": "text", "limit": 20},
  "Benefits": {"type": "list", "limit": 50, "default_quantity": 3}
}
Task: 请将上述技术内容重写为针对 CEO 的商业价值描述，语言需简洁有力。
```
**LLM Response**:
```json
{
  "elements": [
    {"name": "Title", "data": ["云原生驱动业务敏捷"]},
    {"name": "Benefits", "data": [
      "极速发布：产品迭代周期缩短 40%",
      "弹性成本：资源利用率提升 2 倍",
      "高可用：业务连续性达 99.99%"
    ]}
  ]
}
```
#### Agent 2: Coder (动作翻译)
**User Prompt**:
```text
===Edit Target (HTML)===
<div class="Title">云原生架构技术细节</div>
<ul class="Benefits">
  <li>容器化部署</li>
  <li>微服务拆分</li>
</ul>
===Commands===
("Title", "text", "quantity_change: 0", ["云原生架构技术细节"], ["云原生驱动业务敏捷"])
("Benefits", "text", "quantity_change: 1", ["容器化部署", "微服务拆分"], ["极速发布...", "弹性成本...", "高可用..."])
Task: 请生成 API 调用。
```
**LLM Response**:
```python
# 修改标题
replace_paragraph("Title", 0, "云原生驱动业务敏捷")
# 增加并更新列表项
clone_paragraph("Benefits", 1) # 基于第二个 li 克隆
replace_paragraph("Benefits", 0, "极速发布：产品迭代周期缩短 40%")
replace_paragraph("Benefits", 1, "弹性成本：资源利用率提升 2 倍")
replace_paragraph("Benefits", 2, "高可用：业务连续性达 99.99%")
```
