# qqmusic-api

Claude Code 插件，通过 QQ 音乐内部 Web API 自动化管理歌单。如批量导入歌曲、分类歌单等操作。

## 这是什么

逆向工程自 y.qq.com 的内部接口，配合 `chrome-devtools-mcp`，让 Claude 能够直接在已登录的 Chrome 浏览器中操作 QQ 音乐歌单——无需官方 SDK，无需独立登录流程。

## 前置条件

在 y.qq.com 登录 QQ 音乐即可。认证信息（`qm_keyst` cookie）的获取有三种策略，按优先级自动选择：

1. **自动提取**（推荐）：配置了 `chrome-devtools-mcp` 时，自动从浏览器读取 cookie，无需手动操作
2. **文件/环境变量**：从 `.env` 或环境变量 `QM_KEYST` 读取
3. **手动提供**：以上都没有时，引导你从浏览器 DevTools 复制 cookie

## 安装

```bash
claude --plugin-dir /path/to/qqmusic-api
```

## 已实现的操作

| 操作 | 接口 | 状态 |
|------|------|------|
| 列出用户所有歌单 | GET c.y.qq.com | 可用 |
| 获取歌单歌曲列表 | GET c.y.qq.com | 可用 |
| 获取歌单 dirid | GET c.y.qq.com | 可用 |
| 向歌单添加歌曲（批量） | POST musicu.fcg | 可用 |
| 从歌单删除歌曲 | POST musicu.fcg | 可用 |
| 删除歌单 | POST musicu.fcg | 可用 |
| 创建歌单 | POST musicu.fcg | 可用 |
| 搜索歌曲 | POST musicu.fcg | 可用 |
| 红心歌曲（添加/取消收藏） | POST musicu.fcg | 可用 |
| 查询歌曲是否已红心 | POST musicu.fcg | 可用 |
| SmartBox 快速搜索 | GET c.y.qq.com | 可用 |
| 文字导入（搜索 + 批量添加） | 组合流程 | 可用 |

## 使用示例

```
帮我把陈奕迅的歌全部移到"收藏"歌单里
```

```
帮我导入这些歌到"我的收藏"里：
超级帅大叔 - 结婚
超级帅大叔 - 我们的城堡
超级帅大叔 - 我要奔跑
超级帅大叔 - 光年之外
超级帅大叔 - 等下一个人
```

```
桌面上有个 music.txt，每行一首歌，帮我全部导入到"我的收藏"歌单。
```

```
把我所有带有 Live 标识，专辑来自音乐综艺节目的歌曲，都移到"音乐舞台"歌单
```

## 相关项目

- [music163-api](https://github.com/XHXIAIEIN/music163-api) — 网易云音乐版，同样的使用体验，管理网易云音乐歌单

## 免责声明

本插件使用的是 QQ 音乐**非官方内部 API**，这些接口：

- 随时可能因腾讯后端变更而失效
- 不保证长期稳定可用
- 使用时需遵守 QQ 音乐用户协议
- 本项目仅供个人学习和自动化用途，不得用于商业目的

## License

MIT
