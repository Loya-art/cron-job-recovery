---
name: cron-job-recovery
description: 诊断并恢复失败的 Hermes 定时任务（cron）——技能缺失、API key 丢失、静默失败。当定时任务报 "skill not found"、需要未配置的凭据、或曾经正常现在开始失败时使用。
---

# Cron Job Recovery（Hermes 定时任务故障恢复）

## 触发场景
- cron 最终报告开头出现 "skill(s) ... could not be found and were skipped"
- 定时任务需要某个凭据（API key / token）但环境里没有
- 一个曾经跑成功的任务开始失败（先看它的历史输出找最后一次成功）
- 任何 cron 失败诊断：先读该任务自己的输出历史

## 文件位置（本机）
> ⚠️ **2026-09-13 更正**：下列 `E:\AI 代理工具\hermes\hermes-home\...` 路径**已于 2026-09-13 整体删除**（9/7 半截 bootstrap 残留；且它的 `bin\uv.exe` 被 PATH 抢先解析，导致 DSH 的 Hindsight 记忆守护进程每次启动都弹一个黑窗）。Loya-art目前**不使用 Hermes**；将来若重装，先用 `hermes config path` 确认 HERMES_HOME，不要照抄旧路径。
- 任务配置：`<HERMES_HOME>\cron\jobs.json`（prompt、schedule、skills、deliver 目标）
- 每次运行的输出：`<HERMES_HOME>\cron\output\<jobid>\<时间戳>.md`——**每个文件都含完整 prompt 和 agent 的最终回复**；失败运行文件名带 `(FAILED)` 标记
- 凭据：`<HERMES_HOME>\.env`（WEREAD_API_KEY、WEIXIN_*、DEEPSEEK_* 等）
- 技能：`<HERMES_HOME>\skills\<name>\SKILL.md`
- 会话转储（可能含历史会话用过的 API key）：`<HERMES_HOME>\sessions\request_dump_*.json`
- D 盘镜像：`D:\hermes-environment\`（skills/、credentials/、tools/）——用户的同步约定：原位置保留，镜像一份过去

## 诊断流程
1. 按时间倒序读该 job 的 cron 输出文件——输出的 Prompt 区段 + agent 回复直接告诉你卡在哪
2. 检查技能目录是否**真的**有非空 SKILL.md（空目录会让 runner 报 "skill not found"）
3. 检查 .env 里有没有任务需要的凭据

## 恢复技术（2026-08-01 实战验证）
1. **技能内容就藏在 cron 输出里**：成功加载过该技能的那次运行，其输出 .md 的 Prompt 区段嵌着完整 SKILL.md（frontmatter `---` 到正文结尾）。抽取出来写回 `skills/<name>/SKILL.md` 即可还原。（当时 weread-skills 目录被清空，v1.0.4 全文就是这样找回的）
2. **API key 可从会话转储找回**：`grep -o -E "wrk-[A-Za-z0-9_-]{6,}" hermes-home/sessions/request_dump_*.json`——注意排除技能文档里的占位符（如 `wrk-xxxxxxxx`），取另一个。先用最简接口验证再信任
3. **写回 .env** 并同步 `D:\hermes-environment\credentials\`
4. **验证凭据**：调服务端最简接口（如网关 `/_list` 返回 errcode:0）

## 坑
- cron 触发的新 shell 可能读不到刚加的 .env 变量——重跑一次任务或重启 hermes 后生效
- 目录列表里看似"已安装"的技能目录可能是空的——永远检查目录内文件数
- 不要指望浏览器 cookie 里有微信读书登录态（Chrome/Edge 均无）
- 凭据缺失时**不要编造任务数据**——如实报告阻塞点和修复方法（用户明确要求过：只从 API 实际返回中提取）

## 支持文件
- `references/weread-reading-reminder.md` — 微信读书网关 API 实测细节 + 每日读书提醒任务的工作流（触发本技能的反复出现的 cron）
