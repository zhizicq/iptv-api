# Gitee 与 GitHub 两套配置的维护说明

仓库同时服务两条互不干扰的发布链路：GitHub Actions（海外网络）与 Gitee Go（国内网络）。
两者的运行参数、白名单与发布目标都不同，**改一侧时不要覆盖另一侧**。

## 一览

| 维度 | GitHub 侧 | Gitee 侧 |
| --- | --- | --- |
| 配置文件 | `config/config.ini` | `config/config.gitee.ini` |
| 白名单 | `config/whitelist.txt`（378 条，永久置顶） | `config/whitelist.gitee.txt`（空） |
| 测速 | `open_speed_test = False`、`open_filter_speed = False` | `open_speed_test = True`、`open_filter_speed = True` |
| 结果策略 | 全量展示（约 830 条），不剔除慢/失效源 | 只展示通过测速与速率阈值的源 |
| IPv6 | `ipv_type = all` | `ipv_type = ipv4`（runner 无 IPv6） |
| 每频道条数 | `output_urls_limit = 5` | `output_urls_limit = 3` |
| 速率阈值 | `min_speed = 0.5` | `min_speed = 0.3` |
| 分辨率过滤 | 关闭 | 关闭（Gitee 镜像没有 ffmpeg） |
| 截图 / RTMP / 兜底 | 见 config.ini | 全部关闭 |
| 发布目标 | GitHub Pages | Gitee `output` 分支 |
| 流水线 | `.github/workflows/*` | `.workflow/refresh-generate.yml` |

## Gitee 侧配置要点

`config/config.gitee.ini` 由 `config/config.ini` 复制而来，只改动以下键：

| 键 | 值 | 说明 |
| --- | --- | --- |
| `open_speed_test` | `True` | 国内网络下必须真实测速 |
| `open_filter_speed` | `True` | 按速率过滤，慢源不进结果 |
| `open_filter_resolution` | `False` | Gitee Go 镜像没有 ffmpeg，探测必失败 |
| `open_stream_screenshot` | `False` | 同上，依赖 ffmpeg |
| `open_rtmp` | `False` | 不做 RTMP 拉流 |
| `open_supply` | `False` | 关闭兜底源，避免把未测速源塞回结果 |
| `speed_test_mode` | `quick` | 快速测速，控制流水线时长 |
| `ipv_type` | `ipv4` | runner 无 IPv6，避免无效测速 |
| `output_urls_limit` | `3` | 每频道最多 3 条，减少重复 |
| `min_speed` | `0.3` | 与国内网络实测带宽匹配的阈值 |

`config/whitelist.gitee.txt` 只保留 `[WHITELIST]` 与 `[KEYWORDS]` 两个段头、内容为空：
白名单里的接口不参与测速且强制置顶，在国内真实测速链路里会架空测速，因此 Gitee 侧不启用。

## 流水线如何生效

`.workflow/refresh-generate.yml` 在安装依赖之后、运行 `main.py` 之前执行两条覆盖：

```bash
cp config/config.gitee.ini config/config.ini
cp config/whitelist.gitee.txt config/whitelist.txt
```

覆盖只发生在流水线的工作区，**不会提交回 master**；随后脚本会 echo 关键项
（open_speed_test / open_filter_speed / open_filter_resolution / ipv_type / whitelist 条目数）便于核对。

## 维护规则

1. **不要互相覆盖**：`config/config.ini`、`config/whitelist.txt` 只服务 GitHub；
   `config/config.gitee.ini`、`config/whitelist.gitee.txt` 只服务 Gitee。
2. **新增频道模板两边共用**：`config/demo.txt` 是频道模板，两侧共享同一份，改它要同时评估两条链路。
3. **改 Gitee 侧参数**：直接编辑 `config/config.gitee.ini`（或临时改流水线覆盖后的值做验证），不要改 `config/config.ini`。
4. **改 GitHub 侧参数**：直接编辑 `config/config.ini`，不要动 `config.gitee.ini`。
5. **想让 Gitee 也用白名单**：只往 `config/whitelist.gitee.txt` 里加，且要接受「这些源不参与测速」的后果。
6. **Gitee 侧依赖**：`requirements-py39.txt`（Python 3.9 可用的最高版本），不要沿用 Pipfile 的 3.14 时代版本。

## 验证

```bash
# Gitee 侧配置可解析且关键项正确
python -c "import configparser as c; p=c.ConfigParser(); p.read('config/config.gitee.ini', encoding='utf-8'); s=p['Settings']; print({k: s[k] for k in ('open_speed_test','open_filter_speed','open_filter_resolution','open_rtmp','open_supply','speed_test_mode','ipv_type','output_urls_limit','min_speed')})"

# 白名单为空
grep -vE '^\s*(#|\[|$)' config/whitelist.gitee.txt | wc -l   # 期望 0
```
