# GitHub 开源发布指南

## 1. 这些文件分别起什么作用

| 仓库路径 | 作用 | 当前状态 |
| --- | --- | --- |
| `README.md` | 项目入口，介绍用途、安装、输入、运行和限制 | 已添加 |
| `LICENSE` | 给出代码与随附文档的使用、修改和分发许可 | 已添加 MIT |
| `requirements.txt` | 安装运行依赖 | 已添加 |
| `.gitignore` | 避免新加入虚拟环境、本地配置、生产数据和结果文件 | 已添加 |
| `docs/GITHUB_PUBLISHING.md` | 保存发布与维护流程 | 本文 |
| `CONTRIBUTING.md` | 更完整的贡献约定 | 可在贡献者增加后添加，当前 README 已有简要说明 |
| `.github/ISSUE_TEMPLATE/` | 提交问题时的表单模板 | 后续可选 |
| `.github/workflows/` | GitHub Actions 自动验证流程 | 后续有可公开的示例与测试后再添加 |

公开仓库让别人看到代码，开源许可证则说明别人如何使用代码。MIT 允许商业使用、修改和再分发，需要保留版权及许可声明，并包含不提供担保的条款。参见 [GitHub 许可证说明](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/licensing-a-repository) 和 [MIT 标准文本](https://opensource.org/license/mit)。

`LICENSE` 目前以 `Hot-Rolled Steel Preprocessing contributors` 作为项目贡献者署名。发布前可将其改为实际著作权人的姓名、GitHub 名称或组织名称；不要把他人的代码权利归到自己名下。该许可不授予原始生产数据的使用权。

## 2. 本项目的发布起点

本指南编写时，项目已有本地 Git 仓库，分支为 `main`，尚无首次提交，也未配置远程仓库。Notebook 之前已暂存过，因此**需要重新 `git add` 清理后的版本**；直接提交旧的暂存内容可能带上此前的运行输出。

已添加的忽略规则覆盖 CSV、Excel、Parquet、虚拟环境、编辑器配置和 `semi_supervised_outputs/`。本地数据与导出结果仍保留在磁盘。`.gitignore` 不会删除文件，也不会自动撤销已经暂存或跟踪的文件。

Notebook 的公开版本已清除执行输出，默认数据路径改为 `data/input.csv`。以后每次运行后，在 JupyterLab 中使用清除所有输出的操作，再保存 Notebook，最后暂存。

## 3. 在 GitHub 创建空的公开仓库

登录 GitHub 后创建新仓库：

1. 仓库名可使用 `hot-rolled-steel-preprocessing`，也可自行命名。
2. 描述可填：`热轧板带数据预处理与随机森林回归自训练实验`。
3. 选择 **Public**。
4. **不要勾选自动添加 README、.gitignore 或 License**，因为本地已有这些文件。
5. 创建后复制仓库的 HTTPS 地址。

这个流程与 [GitHub 官方本地代码上传指南](https://docs.github.com/en/migrations/importing-source-code/using-the-command-line-to-import-source-code/adding-locally-hosted-code-to-github?platform=windows) 一致。以下命令中的 `YOUR_USERNAME` 必须替换为你的 GitHub 用户名，仓库名也需对应实际创建的名称。

## 4. 本地暂存与首次提交

在项目根目录打开 PowerShell。先检查 Git 提交身份：

```powershell
git config user.name
git config user.email
```

若为空或需要更改，只在此仓库设置（先替换示例文本）：

```powershell
git config user.name "你的 GitHub 名称"
git config user.email "你的 GitHub 提交邮箱"
```

可以使用 GitHub 账户设置中显示的 noreply 提交邮箱。然后明确暂存要公开的文件：

```powershell
git add README.md LICENSE requirements.txt .gitignore docs/GITHUB_PUBLISHING.md "数据预处理/数据预处理.ipynb"
git status --short
git diff --cached --stat
git diff --cached -- "数据预处理/数据预处理.ipynb"
```

检查列表应主要包含上述六个文件，Notebook 的代码单元格应为 `outputs: []` 且执行计数为 `null`。如果看到了本地数据或其他不准备公开的文件，先从暂存区移除；在本项目尚无首次提交的状态下，可对明确的文件使用 `git rm --cached -- "具体文件路径"`，这会保留磁盘文件。确认暂存内容后提交：

```powershell
git commit -m "Prepare project for open source release"
```

`git add` 将文件当前版本放入暂存区；`git commit` 创建本地历史记录；只有后面的 `git push` 才会上传这些提交。

## 5. 关联远程仓库并推送

```powershell
git remote add origin https://github.com/YOUR_USERNAME/hot-rolled-steel-preprocessing.git
git remote -v
git push -u origin main
```

按 Git 凭据管理器提示完成 GitHub 登录。若远程名称 `origin` 已存在，先用 `git remote -v` 检查其地址，不要重复添加。命令行身份验证可参考 [GitHub 身份验证文档](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/about-authentication-to-github)。

若误在远程仓库初始化了 README，导致推送被拒绝，先检查远程历史并整合双方文件；不要直接强制推送覆盖远程内容。

## 6. 上传后检查

- 仓库首页能正常展示 README，安装步骤与 Notebook 链接可用。
- `LICENSE` 内容完整，GitHub 能识别 MIT 许可证。
- Notebook 页面可以浏览代码，且没有历史生产数据输出。
- 仓库文件列表中没有生产 CSV、增强训练集或 `.venv`。
- 在仓库 About 中补充简介与主题，例如 `python`、`jupyter-notebook`、`semi-supervised-learning`、`regression`。

当前仓库需要使用者自行准备数据。下一步如果希望别人能下载后立即体验，可以增加一份明确标为合成数据的 `examples/demo.csv`；忽略规则已为该位置预留例外，发布前仍需核对内容。

## 7. 后续维护

每次修改后，先运行必要验证、清除 Notebook 输出，再提交和推送：

```powershell
git add README.md "数据预处理/数据预处理.ipynb"
git diff --cached --stat
git commit -m "Describe the change"
git push
```

将实际修改的其他文件也加入 `git add`。较大改动可以先创建分支、提交后通过 Pull Request 合并；稳定版本可以建立 tag 并在 GitHub 创建 Release，记录变化和已知限制。

本次准备只修改本地项目文件，未创建远程仓库、创建 Git 提交或执行推送。
