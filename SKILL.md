---
name: cloudflare-pages-deploy
description: >-
  Deploy any static project (HTML, React, Vue, Next.js, etc.) to Cloudflare Pages,
  optionally binding a custom domain or subdomain from the user's own Cloudflare zones,
  fully automated end to end. Use when the user says: "帮我部署到 Cloudflare"、
  "deploy to Cloudflare"、"发布到 Cloudflare"、"上线到 Cloudflare"、"帮我把这个网页部署到网上"、
  "部署这个项目"、"帮我把这个放到网上"、"cloudflare deploy"、"绑定二级域名"、"绑个域名"、
  "用我自己的域名"、"自定义域名"。Trigger on any deployment intent regardless of project type.
---

# Cloudflare Pages Deploy Skill

从零到上线：建项目 → 部署 → 绑定自定义域名/二级域名 → 验证。
全程命令行完成，不需要去 Dashboard 手点（除非用户没有 API Token）。

---

## 0. 认识两种凭证

| 凭证 | 用途 | 从哪来 |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | 部署 + 建域名 + 改 DNS | 用户在 Dashboard 自建 API Token |
| wrangler OAuth 登录 | 只能部署，**不能改 DNS** | `npx wrangler login` 浏览器授权 |

**推荐让用户提供 API Token**，一次搞定全流程。需要的权限：

- `账户` → `Cloudflare Pages` → **编辑**
- `区域` → `DNS` → **编辑**
- `区域` → `区域` → **读取**

> 中文界面路径：https://dash.cloudflare.com/profile/api-tokens → 创建令牌 → 创建自定义令牌 → 开始使用

检查环境：

```bash
echo "${CLOUDFLARE_API_TOKEN:-未设置}"
npx wrangler whoami
```

- 有 `CLOUDFLARE_API_TOKEN`：wrangler 会自动使用它，跳过 `wrangler login`。
- 都没有：跑 `npx wrangler login`（会开浏览器，放在终端面板里跑让用户点授权）。

---

## 1. 确认要部署的目录

用户没说清楚就先问。静态站点直接指到含 `index.html` 的目录；React/Vue/Next 先构建，指到产物目录（`dist` / `build` / `out`）。

```bash
ls <dir>
```

---

## 2. 部署

```bash
# 项目不存在时创建（存在会报错，可忽略后继续 deploy）
npx wrangler pages project create <project-name> --production-branch=main

# 部署
npx wrangler pages deploy <dir> --project-name=<project-name>
```

项目名规则：小写字母、数字、连字符。用户没指定就从目录名推断，并告知一声。

部署成功后先确认 pages.dev 能访问：

```bash
curl -s -o /dev/null -w "%{http_code}\n" https://<project-name>.pages.dev/
```

**到这里已经可交付了。** 如果用户没提域名，输出地址就结束，别自己加戏。

---

## 3. 绑定自定义域名 / 二级域名

### 3.1 列出用户账号里的域名，让用户选

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones?per_page=50" |
python -c "
import sys,json
d=json.load(sys.stdin)
if not d.get('success'):
    print('ERROR:', d.get('errors')); sys.exit(1)
for z in d['result']:
    print(f\"{z['name']}  (status={z['status']}, id={z['id']})\")
"
```

把结果列给用户，问两件事：

1. 用哪个域名（zone）
2. 用哪个二级域名前缀 —— 例如 `hucong` → `hucong.example.com`；根域名传 `@`

> 想加 www 之类的常见前缀可以直接建议，但别替用户决定。

### 3.2 先查重，别覆盖已有记录

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?type=CNAME&name=$FULL_DOMAIN"
```

有结果就把现有的 `content` 告诉用户，确认是改还是换，**不要直接覆盖**。

### 3.3 把域名挂到 Pages 项目

这一步用的是 Pages API，`pages:write` 权限即可：

```bash
curl -s -X POST \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data "{\"name\":\"$FULL_DOMAIN\"}" \
  "https://api.cloudflare.com/client/v4/accounts/$ACCOUNT_ID/pages/projects/$PROJECT/domains"
```

`ACCOUNT_ID` 获取：

```bash
curl -s -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  "https://api.cloudflare.com/client/v4/accounts" |
python -c "import sys,json; [print(a['id'], a['name']) for a in json.load(sys.stdin)['result']]"
```

返回 `status: initializing/pending` 是正常的，等 DNS 记录建好才会变 active。

### 3.4 建 CNAME 记录

```bash
curl -s -X POST \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  --data "{\"type\":\"CNAME\",\"name\":\"$LABEL\",\"content\":\"$PROJECT.pages.dev\",\"proxied\":true,\"ttl\":1}" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records"
```

- `name` 填前缀（如 `hucong`），根域名填 `@`
- `proxied: true` 必须开着，否则 Cloudflare 不会代签证书
- `ttl: 1` = 自动

### 3.5 验证

证书签发通常 30 秒～2 分钟。轮询有上限，不要死等：

```bash
for i in 1 2 3 4 5 6; do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 15 "https://$FULL_DOMAIN/")
  echo "尝试 $i: HTTP $code"
  [ "$code" = "200" ] && break
  sleep 20
done
curl -s --max-time 15 "https://$FULL_DOMAIN/" | head -c 200
```

- `200` 且内容对 → 完成
- 持续 `522/523/526` → 记录或证书还没生效，再等，或让用户检查 DNS
- `401` → Access 应用挡住了，提醒用户

---

## 4. 权限不足怎么办

症状：`Authentication error` / `code 10000`，但 `whoami` 显示已登录。

原因：wrangler 的 OAuth 令牌只有 pages/workers 相关 scope，**没有 DNS 权限**。

处理顺序：

1. 请求用户提供一个 API Token（权限见第 0 节），环境变量传进来。
2. 用户不想建 Token → 给出 DNS 记录的三个字段，让用户在 Dashboard 手加：
   `类型=CNAME，名称=<前缀>，目标=<project>.pages.dev，代理=已代理`
3. 加完再回来跑 3.5 验证。

**永远不要**为了绕过权限去改 wrangler 的配置文件或读取用户的 refresh token。

---

## 5. 安全

- Token 只从环境变量读。允许存放在两个地方：
  - `~/.claude/settings.json` 的 `env` 块（用户私有配置，推荐，配置一次永久生效）
  - 会话里临时传入的环境变量
- **不要**写进项目目录、脚本文件、git 仓库。
- 用户把 Token 贴在对话里时，收尾提醒一句：用完去 Dashboard 点「滚动」或「删除」作废。
- 不要打印完整 Token。

---

## 6. 收尾输出

> ✅ 部署完成！
> - **访问地址**：`https://<project>.pages.dev`
> - **自定义域名**：`https://<domain>`（如已绑定）
> - **管理后台**：https://dash.cloudflare.com/pages

如果改了 DNS，补一句让用户去作废临时 Token。

---

## 常见问题

| 报错 | 处理 |
|---|---|
| `You are not authenticated` | `npx wrangler login`，或设置 `CLOUDFLARE_API_TOKEN` |
| `Project not found` | 先 `wrangler pages project create <name> --production-branch=main` |
| `Must specify a production branch` | 加 `--production-branch=main` |
| `Authentication error` 改 DNS 时 | 令牌缺 `Zone.DNS:Edit`，见第 4 节 |
| 自定义域名一直 pending | CNAME 没建，或 `proxied` 没开 |
| 证书 pending | 等 1~5 分钟；仍不动就检查 DNS 是否指向 `<project>.pages.dev` |
| 本地 `nslookup` 结果奇怪（198.18.x.x） | 代理软件的 fake-ip，别当依据；用 `curl https://cloudflare-dns.com/dns-query` 或直接访问验证 |
