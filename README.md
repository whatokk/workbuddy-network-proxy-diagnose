# 网络代理诊断与清理

> WorkBuddy Skill · 屋里涛说

诊断并清理 WorkBuddy / 本机的代理阻塞问题。关键是**分清三类「代理」，别删错**。

## 技能清单

| 技能 | 说明 |
|---|---|
| **workbuddy-network-proxy-diagnose** · 代理诊断 | 端口归属识别 → 多路由对比实测 → 用户级代理变量清理与验证。 |


## 安装

把 `skills/` 下的技能目录拷贝到 WorkBuddy 的技能目录：

```bash
cp -r skills/* ~/.workbuddy/skills/
```

Windows PowerShell：

```powershell
Copy-Item .\skills\* "$env:USERPROFILE\.workbuddy\skills\" -Recurse -Force
```

重启 WorkBuddy 后，技能列表即可看到。

## 使用要点

- 触发词：联网异常、代理挡路、联网慢、curl exit 7、删除代理、子 Agent 502。

## 环境依赖

- PowerShell / curl

## 目录规范

```
workbuddy-network-proxy-diagnose/
└── skills/
    ├── workbuddy-network-proxy-diagnose/
```

每个技能遵循统一结构：`SKILL.md`（必需，含 name/description frontmatter）+ `scripts/`（可选）+ `references/`（可选）。

---

## License

MIT — 随意取用、修改、二次分发。
