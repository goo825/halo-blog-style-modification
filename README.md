# halo博客样式修改

这个仓库用于管理 Halo 博客的主题样式资源，并配合 Jenkins 展示从 GitHub 到 Kubernetes 中 Halo 博客的自动化发布流程。

## 演示目标

提交样式文件到 GitHub 后，由 Jenkins 拉取仓库内容，将静态资源同步到 Halo 工作目录，并让博客页面呈现可见的样式变化。

## 仓库结构

```text
.
├── Jenkinsfile
└── assets
    ├── README.md
    └── css
        └── custom.css
```

## Jenkins 流水线

新建 Jenkins Pipeline 时可以使用这个仓库：

```text
https://github.com/goo825/halo-blog-style-modification.git
```

流水线会执行三步：

1. 在 `blog-system` 命名空间中查找 `app=halo` 的 Pod。
2. 将 `assets/` 目录同步到 Halo Pod 的 `/root/.halo2/assets`。
3. 重启 `deployment/halo`，等待滚动更新完成。

## 当前样式效果

`assets/css/custom.css` 会调整博客的基础配色、卡片、标题、引用块和按钮样式，并在页面右下角显示：

```text
GitHub + Jenkins 自动发布样式已生效
```

看到这个标记，就说明 Jenkins 已经把 GitHub 中的样式资源发布到了 Halo。
