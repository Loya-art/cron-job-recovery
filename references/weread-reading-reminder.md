# 微信读书（WeRead）每日读书提醒 — 实操细节

触发的 cron：`每日读书提醒`（job id 54600ca6e142，每天 08:00，deliver 到微信）。
weread-skills 技能目录曾被清空导致该任务连败数日，本文件记录实测可靠的恢复与调用知识。

## 网关调用（实测 2026-08-01）

- `POST https://i.weread.qq.com/api/agent/gateway`
- Header: `Authorization: Bearer $WEREAD_API_KEY`（格式 `wrk-xxxxxxxx`，key 在 `hermes-home/.env`）
- Body 参数**必须平铺在顶层**：`{"api_name": "/shelf/sync", "skill_version": "1.0.4"}`，不能包进 `params` 对象
- 每次请求必须带 `skill_version`
- `{"api_name": "/_list"}` 返回全部接口与参数定义（`errcode:0` 正常）
- 匿名调 weread.qq.com 原生接口一律 `{"errCode":-2010,"errMsg":"用户不存在"}`；网关数据接口全部 `need_login:true`

## 已验证的回包结构

### /shelf/sync
- `books[]`（bookId/title/author）、`albums[]`（有声书）、`mp`（文章收藏）；书架总数 = books + albums + (mp 非空?1:0)
- 坑：公众号书（bookId 形如 `MP_WXS_...`）progress 恒为 0 但 readingTime > 0，别据此判"没在读"

### /book/getprogress?bookId=<id>
- 进度百分比在 **`book.progress`**（0-100），不在顶层！还有 `book.chapterIdx`、`book.updateTime`（epoch 秒）、`book.readingTime`（累计秒）、`book.summary`
- "在读"判定：`progress > 0`；`progress == 0` 只是书架收藏

### /book/bestbookmarks?bookId=<id>&chapterUid=0
- 返回 `items[]`：`markText`（划线文本）+ `markCount`（划线人数，125579 → 展示"12.5 万人划线"），按热度排序最多 20 条
- 片段 ≤100 字；**上次推送用过的划线别重复**——换同书第 2/3 条热门

### /readdata/detail
- **必须带 `mode`**（weekly/monthly/annually/overall），缺 mode → `HTTP Error 499`
- `baseTime=0` → 当前周期。⚠️ 月初跑 monthly 会是 0（本月刚开始），报上个月数据就传上个月 1 号的 epoch 秒（如 2026-07-01 → 1782835200）
- 返回：`readDays`、`totalReadTime`（秒）、`dayAverageReadTime`（秒）、`readTimes`（按月桶）、`longest[].book`（读最久的书）
- 展示：秒 → "X 小时 Y 分钟"

## 消息口径（用户模板要求）
- 书单列"在读"：progress > 0 且按 updateTime 排序，主推最近更新的；其他搁置的简单带过
- 主书快读完（86% 之类）给冲刺鼓励；片段优先贴"收官"情绪的名句（如《明朝那些事儿》"成功只有一个——按照自己的方式，去度过人生。"，11.9 万人划线）
- 统计报上个月（月初跑时本月必然为 0）
- 语气亲切 + emoji；不编造数据，只从 API 实际返回提取

## 凭据/技能丢失恢复（详见主 SKILL.md）
1. 技能全文嵌在历史 cron 输出 `cron/output/<jobid>/*.md` 的 Prompt 区段，抽取还原
2. key 在 `sessions/request_dump_*.json` 里 grep `wrk-`（排除占位符 `wrk-xxxxxxxx`）
3. 写回 .env + 同步 D:\hermes-environment\credentials\
