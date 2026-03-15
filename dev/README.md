# /dev

Development folder for in-progress features.

Each feature gets its own subfolder:
```
/dev/{feature-name}/index.html
```

**Workflow:**
1. Copy current prod HTML into `/dev/{feature}/index.html`
2. Develop and test on the feature branch
3. When ready, backup prod → `/backups/` and promote dev file → `/prod/index.html`
