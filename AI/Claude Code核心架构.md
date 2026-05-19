### 一、Claude Code是什么
Agent核心就是【感知-决策-行动】的循环，输入一个目标，它会决定读什么文件，调用什么工具，跑什么脚本、修改哪行代码，整体行动会持续十几轮，直到任务执行完成。
Claude Code的循环：
![](AI-Agent-Learning/AI/assets/claude%20code核心架构/file-20260520071736182.jpg)

核心是**大模型自己决定**做什么

### 二、架构设计
自主编程的Agent可以处理的事情非常多：调大模型API、执行几十种工具、管理权限、压缩上下文、维护记忆、支持多Agent协作等，如果全部塞到一个文件中，会非常混乱。
Claude Code如何管理：四层架构
![](AI-Agent-Learning/AI/assets/claude%20code核心架构/file-20260520072105817.jpg)


