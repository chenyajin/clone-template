# vue3

工程化配置

## 一、添加 Husky + lint-staged +

git hooks 实现提交代码前自动化校验，并格式化代码。

1、**利用 git hooks 实现提交前检验**

- 安装 husky，添加 .husky 文件夹

```sh
# 安装 husky
pnpm add husky -D

# 生成 .husky 文件夹（注意：这一步操作之前，一定要执行 git init 初始化当前项目仓库，.husky 文件夹才能创建成功）
npx husky-init install
```

- 修改 `.husky/pre-commit`

```js
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm lint
```

- 配置已完成，测试结果

```sh
git add .
git commit -m 'test husky'
```

> - 当 git commit 时，它会自动检测到不符合规范的代码，如果无法自主修复就会抛出错误提示。
> - 这会对整个项目的工作区（Workspace）和 暂存区（Stage）的变动代码进行检查，但是 `git commit -m` 只对暂存区的代码进行提交，所以工作区的代码没有必要进行校验。
> - lint-staged 会过滤出暂存区的文件，满足上面的需求.

2、**利用 lint-staged 对暂存区代码进行校验并格式化**

- 安装 lint-staged

```sh
pnpm add lint-staged -D
```

- 修改 package.json

```json
  "script": {
    // ...
    "prepare": "lint-staged"
  }
  // ...
  "lint-staged": {
    "*.{vue,js,ts,jsx,tsx}": [
      "pnpm lint",
      "pnpm format"
    ]
  }
```

- 修改 `.husky/pre-commit`

```sh
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

pnpm prepare
```

3、**对样式代码进行校验并格式化**

## 二、添加 Commitlint

团队以约定commit的规范来提交代码。

1、 安装插件

- @commitlint/cli Commitlint 命令行工具
- @commitlint/config-conventional 基于 Angular 的约定规范

```sh
pnpm add @commitlint/config-conventional @commitlint/cli -D
```

2、添加到钩子

```sh
npx husky add .husky/commit-msg 'npx --no-install commitlint --edit "$1"'
```

3、创建 .commitlintrc， 写入配置

```json
{
  "extends": ["@commitlint/config-conventional"]
}
```

4、配置完成，测试
