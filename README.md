# cloudflare-pages-deploy

一个 Claude Code Skill：把任意静态项目部署到 Cloudflare Pages，并且可以自动绑定你自己域名下的二级域名——全程命令行，不用去 Dashboard 手点。

## 它能做什么

- 部署静态站点（HTML / React / Vue / Next.js / 任意静态文件）
- 自动创建 Pages 项目
- 列出你 Cloudflare 账号下的所有域名，让你选一个
- 绑定二级域名：自动建 CNAME、等证书签发、验证可访问
- 权限不足时给出明确的排障指引

图片、视频也能传（单文件上限 25 MiB，单站点 20,000 个文件）。

## 安装

把 `SKILL.md` 放到 Claude Code 的 skills 目录：

```bash
# macOS / Linux
mkdir -p ~/.claude/skills/cloudflare-pages-deploy
cp SKILL.md ~/.claude/skills/cloudflare-pages-deploy/

# Windows (Git Bash)
mkdir -p "$USERPROFILE/.claude/skills/cloudflare-pages-deploy"
cp SKILL.md "$USERPROFILE/.claude/skills/cloudflare-pages-deploy/"
```

或者克隆整个仓库后软链接过去。

## 准备

- Node.js ≥ 16
- 一个 Cloudflare 账号
- 一个 API Token（推荐），权限三项：

| 类别 | 权限 | 级别 |
|---|---|---|
| 账户 | Cloudflare Pages | 编辑 |
| 区域 | DNS | 编辑 |
| 区域 | 区域 | 读取 |

绑定自定义域名时还需要：

| 类别 | 权限 | 级别 |
|---|---|---|
| 区域 | 电子邮件路由 | 编辑 |
| 账户 | 电子邮件路由地址 | 编辑 |

（后两项只有用邮件路由功能时才需要）

把 Token 放进环境变量即可，推荐写进 `~/.claude/settings.json` 的 `env` 块：

```json
{
  "env": {
    "CLOUDFLARE_API_TOKEN": "你的token"
  }
}
```

没有 Token 也能用，跑 `npx wrangler login` 浏览器授权即可——但那种方式**没有 DNS 权限**，绑不了自定义域名。

## 用法

在 Claude Code 里直接说人话：

> 把这个文件夹部署到 Cloudflare

> 把这个部署到我的域名下

也可以显式调用：

```
/cloudflare-pages-deploy
```

触发词包括：`部署到 Cloudflare`、`上线`、`发布`、`帮我把这个网页放到网上`、`绑定二级域名`、`用我自己的域名`。

## 流程

```
确认目录 → 检查凭证 → 建项目 → 部署 → [可选] 列域名 → 选二级域名
→ 建 Pages 域名绑定 → 建 CNAME → 轮询验证 → 输出地址
```

## 常见问题

| 现象 | 原因 |
|---|---|
| `Authentication error` 改 DNS 时 | Token 缺 `Zone.DNS:Edit` |
| 自定义域名一直 `pending` | CNAME 没建，或 `proxied` 没开 |
| 证书迟迟不生效 | 等 1~5 分钟；确认目标指向 `<project>.pages.dev` |
| `nslookup` 结果怪异（198.18.x.x） | 代理软件的 fake-ip，不能用它判断 |

## 说明

- 部署上去的内容是**完全公开**的，没有访问控制。要私密访问请用 Cloudflare Access。
- 免费版：单文件 25 MiB、单站点 20,000 文件、每月 500 次部署。

## License

MIT
