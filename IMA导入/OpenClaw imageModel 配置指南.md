---
source: IMA
ima_note_id: "7455939762345967"
created: "2026-05-01 19:22:44"
modified: "2026-05-01 19:22:44"
---

# OpenClaw imageModel 配置指南

OpenClaw imageModel 配置指南一、什么是 imageModelimageModel 是 OpenClaw 中专门用于视觉理解的模型配置，独立于主对话模型（model）。当对话涉及图片或视觉内容时，OpenClaw 会自动切换到 imageModel 指定的模型来处理。二、为什么需要 imageModel专业性：主对话模型（如 GPT-4）虽然全能，但在视觉理解或 OCR 方面可能不如专门的视觉模型（如 Claude 3.5 Sonnet 或 Gemini 1.5 Pro）。成本与性能：视觉模型通常成本较高或推理较慢，仅在处理图片时按需调用，可以优化资源分配。三、如何配置在 ~/.config/alma/config.json 或项目配置中添加如下字段：{  "model": "gpt-4o",  "imageModel": "claude-3-5-sonnet-20240620"}四、支持的模型推荐Claude 3.5 Sonnet: 目前公认的视觉理解和代码能力最强的模型之一。GPT-4o: 响应速度快，多模态原生支持。Gemini 1.5 Pro: 支持超长上下文，视觉细节捕捉极佳。五、注意事项确保所选的 imageModel 已经在你的 Provider 配置中启用。如果未配置 imageModel，OpenClaw 默认会尝试使用 model 来处理图片，但若 model 不支持多模态，则会报错。