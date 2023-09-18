# vue3

vue模版

## 添加 Husky

git hooks 实现提交代码前自动化校验。

1、安装 husky，添加 .husky 文件夹

```sh
# 安装 husky
npm i husky -D

# 生成 .husky 文件夹（注意：这一步操作之前，一定要执行 git init 初始化当前项目仓库，.husky 文件夹才能创建成功）
npx husky-init install
```

2、修改 `.husky/pre-commit`

```js
#!/usr/bin/env sh
. "$(dirname -- "$0")/_/husky.sh"

npm run lint
```

配置已完成，测试结果

```sh
git add .
git commit -m 'test husky'
```

> 当 git commit 时，它会自动检测到不符合规范的代码，如果无法自主修复就会抛出错误提示。

## 添加 Commitlint

团队以约定commit的规范来提交代码。

1、 安装插件

- @commitlint/cli Commitlint 命令行工具
- @commitlint/config-conventional 基于 Angular 的约定规范

```sh
npm i @commitlint/config-conventional @commitlint/cli -D
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
