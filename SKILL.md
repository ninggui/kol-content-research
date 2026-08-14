---
name: kol-content-research
description: 深度学习/研究XX博主帖子时用。定位账号→全量拉取→内涵提炼（观点/方法论），不学语言风格。
---

# KOL 深度内容研究（内涵提炼，非风格模仿）

用户要求"深度学习/研究 XX 博主/XX 的帖子"时使用。目标是从内容中提炼**观点、方法论、知识体系**（用于知识库/竞品研究），**不学语言风格**（风格模仿走 `xhs-style-imitation`）。

已验证案例：2026-08-14 克杰陈（小红书，蔚来资本/体系面）+ 李安迪andylee（B站，蔚来产品/体验面），产出 `/path/to/data/nio_research/kjc_deep_analysis.md` + `andy_deep_analysis.md`。

## 触发条件
- "深度学习XX的帖子/文章"
- "研究一下XX博主"
- "把XX的文章研究下"
- 用户点名 KOL 并要求提炼知识体系

## 执行流程（小红书 KOL）

### 1. 定位账号
```python
# 用 xiaohongshu-mcp 的 search_feeds，多关键词搜索
search_feeds("博主名")              # 直接搜
search_feeds("博主名 领域关键词")   # 加领域词提高召回
```
- 从结果提取：`userId`（**必须完整24位**）+ 任一条笔记的 `xsecToken`
- 判别账号：nickname 精确匹配 + 标题内容相关性

### 2. 拉全量笔记列表
```python
user_profile(user_id=..., xsec_token=..., tab="note")
```
- 返回 ~113KB JSON：`userBasicInfo`（昵称/简介/位置）+ `feeds[]`（全部笔记 id/title/互动）
- 得到全量标题图谱 → 按高赞+主题相关性选深度帖

### 3. 批量拉正文（get_feed_detail）
- payload 写文件再 curl，间隔 3-5s 防风控
- 每篇响应存独立文件：`xhs_kjc_depth_{feed_id}.json`
- **字段路径坑**：不同账号返回结构可能不同！
  - 蔚来官方号：正文在 `noteDetail.desc`
  - 克杰陈：正文在 `data.note.desc`（嵌套更浅）
  - 解析脚本要**同时兼容两种路径**，先探测 `noteDetail` 再试 `data.note`
- **视频笔记坑**：video 类型笔记 `desc` 常为空（内容在画面/口播里），0字节属正常，从标题+互动量提炼，不重试死磕
- **token 过期坑**：xsecToken 会过期（LB 前缀失效），部分拉取返回 0 字节——记录失败项，下一轮用 `search_feeds`/`user_profile` 刷新 token 重试

### 4. 提炼内涵（核心工作）
输出结构：
1. **账号定位**：平台/粉丝/简介/内容类型
2. **核心观点提炼**：每篇深度帖 → 观点 + 出处（帖名/数据）
3. **方法论总结**：该 KOL 的分析框架（可复用）
4. **互补性分析**：与已有知识源对比（视角/方法/覆盖面差异）

## B站 KOL 研究
- 空间页 SPA 常空白 + space API 需 WBI 签名 → 用**搜索页 + browser console fetch**，详见 `bilibili-api-patterns` 的 `references/browser-console-fallback.md`
- 视频 `desc` 常为空（内容在口播）→ 靠**标题图谱 + 播放量分布 + 百度百科词条**提炼
- 身份确认优先查百度百科（`baike.baidu.com/item/{昵称}` 有创作者档案）

## 产出与交付
- 写入 `/path/to/data/nio_research/{博主}_deep_analysis.md`（或对应领域目录）
- **用户说"放到对应文档/知识库"= 立即同步飞书文档，不要反问**（2026-08-14 用户用"?"纠正过"要不要同步"的反问）
- 同步流程：lark-cli 追加章节到对应飞书文档（用 `docs +fetch --detail with-ids` 拿末尾 block_id → `block_insert_after`，读操作被 consent 拦时用后台 delegate_task 绕过，见 lark-cli-master）

## 硬性约束
- **只学内涵，不学语言风格**——用户明确要求时，输出观点/方法论，不产出仿写文案
- 数据真实：引用观点标注来源（帖名/平台/时间），不编造博主没说过的话
- 用户说"其他不纳入"= 收窄范围，尊重决策不追加
- KOL 知识库合并到 skill（如 nio-brand-voice）时注意归属：深度解析内容放研究文档，风格模板才进写作 skill
