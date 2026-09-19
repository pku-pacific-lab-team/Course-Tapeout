# 先进芯片全栈设计与实践

北京大学先进芯片全栈设计与实践课程实验指导网站。

## 本地预览

安装依赖后运行：

```bash
python -m pip install -r requirements.txt
mkdocs serve
```

然后打开终端提示的本地地址。网站内容位于 `2026-fall/`，每年可以复制一个新的年度目录并更新 `mkdocs.yml` 中的 `docs_dir` 和导航。

## 发布

推送到 `main` 或 `master` 分支后，GitHub Actions 会运行 `mkdocs gh-deploy --force`，将网站发布到 GitHub Pages，地址为 <https://pku-pacific-lab-team.github.io/Course-Tapeout/>。

## 目录结构

```text
.
├── 2026-fall/          # 2026 秋季课程内容
│   ├── index.md        # 课程主页
│   ├── lab-0.md ~ lab-6.md
│   ├── final-project.md
│   └── assets/         # 图片与样式
│       ├── brand/      # 校徽、favicon
│       ├── lab0/ ...   # 各次实验的插图，按实验分目录
│       ├── extra.css
│       └── extra.js
├── .github/workflows/  # GitHub Pages 自动部署
├── mkdocs.yml          # MkDocs 配置
└── requirements.txt    # Python 依赖
```

## 发布一次新实验

1. 把实验插图放到 `2026-fall/assets/labN/`，在正文中用 `![说明](assets/labN/xxx.png)` 引用。
2. 编辑 `2026-fall/lab-N.md`，替换掉"尚未发布"的占位内容。
3. 在 `2026-fall/index.md` 的实验安排表里把该行的主题和状态改成「已发布」。
4. 提交并推送到 `main`，Actions 会自动重新部署。
