---
name: save-interview
description: 将面试题及答案追加保存到面试题.md
user_invocable: true
---

# 保存面试题

将对话中涉及的面试题内容追加到 `C:\Users\76741\Desktop\面试\面试题.md`。

## 触发时机

- 用户使用 `/save-interview` 命令
- 用户明确要求保存某道面试题

## 保存格式

按以下格式追加到文件末尾：

```
## {题目}
{答案内容，保留关键点和结构}
```

## 操作步骤

1. 读取 `C:\Users\76741\Desktop\面试\面试题.md` 当前内容
2. 将新题目按格式追加到文件末尾
3. 告知用户已保存
