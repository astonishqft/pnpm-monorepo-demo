# pnpm + workspace 构建你的 monorepo 工程

## 什么是monorepo？

什么是 monorepo？以及和 multirepo 的区别是什么?

关于这些问题，在之前的一篇[介绍 lerna](https://mp.weixin.qq.com/s/4xQTeK0ViMKcCAhSg9P3Vg) 的文章中已经详细介绍过，感兴趣的同学可以再回顾下。

简而言之，`monorepo` 就是把多个工程放到一个 `git` 仓库中进行管理，因此他们可以共享同一套构建流程、代码规范也可以做到统一，特别是如果存在模块间的相互引用的情况，查看代码、修改bug、调试等会更加方便。

## 什么是 pnpm？

`pnpm` 是新一代的包管理工具，号称是**最先进的包管理器**。按照官网说法，可以实现**节约磁盘空间并提升安装速度**和**创建非扁平化的 node_modules 文件夹**两大目标，具体原理可以参考 [pnpm 官网](https://pnpm.io/zh/motivation)。

`pnpm` 提出了 `workspace` 的概念，内置了对 `monorepo` 的支持，那么为什么要用 `pnpm` 取代之前的 `lerna` 呢？

这里我总结了一下几点原因：

- lerna 已经不再维护，后续有任何问题社区无法及时响应
- pnpm装包效率更高，并且可以节约更多磁盘空间
- pnpm本身就预置了对monorepo的支持，不需要再额外第三方包的支持
- **one more thing**，就是好奇心了😜

## 如何使用 pnpm 来搭建 menorepo 工程

### 安装 pnpm

```bash
$ npm install -g pnpm
```

> ⚠️v7版本的pnpm安装使用需要node版本至少大于v14.19.0，所以在安装之前首先需要检查下node版本。

### 工程初始化

为了便于后续的演示，先在工程根目录下新建 `packages` 目录，并且在 `packages` 目录下创建 `pkg1` 和 `pkg2` 两个工程，分别进到 `pkg1` 和 `pkg2` 两个目录下，执行 `npm init` 命令，初始化两个工程，`package.json` 中的 `name` 字段分别叫做 `@qftjs/menorepo1` 和 `@qftjs/monorepo2`(*PS：@qftjs是提前在npm上创建好的组织，没有的话需要提前创建*)。

为了防止根目录被发布出去，需要设置工程工程个目录下 `package.json`配置文件的 `private` 字段为 `true`。

为了实现一个完整的例子，这里我使用了 `father-build` 对模块进行打包，`father-build` 是基于 `rollup` 进行的一层封装，使用起来更加便捷。

在 pkg1 和 pkg2 的src目录下个创建一个 `index.ts` 文件：

```ts
// pkg1/src/index.ts
import pkg2 from '@qftjs/monorepo2';

function fun2() {
  pkg2();
  console.log('I am package 1');
}

export default fun2;
```

```ts
// pkg2/src/index.ts
function fun2() {
  console.log('I am package 2');
}

export default fun2;
```

分别在 pkg1 和 pkg2 下新增 `.fatherrc.ts` 和 `tsconfig.ts` 配置文件。

```ts
// .fatherrc.ts
export default {
  target: 'node',
  cjs: { type: 'babel', lazy: true },
  disableTypeCheck: false,
};
```

```ts
// tsconfig.ts
{
  "include": ["src", "types", "test"],
  "compilerOptions": {
    "target": "es5",
    "module": "esnext",
    "lib": ["dom", "esnext"],
    "importHelpers": true,
    "declaration": true,
    "sourceMap": true,
    "rootDir": "./",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "moduleResolution": "node",
    "baseUrl": "./",
    "paths": {
      "*": ["src/*", "node_modules/*"]
    },
    "jsx": "react",
    "esModuleInterop": true
  }
}
```

全局安装 `father-build`:

```bash
$ pnpm i -Dw father-build
```

最后在 pkg1 和 pkg2 下的 `package.json` 文件中增加一条 `script`:

```json
{
  "scripts": {
    "build": "father-build"
  }
}
```

这样在 pkg1 或者 pkg2 下执行 `build` 命令就会将各子包的ts代码打包成js代码输出至 `lib` 目录下。

要想启动 `pnpm` 的 `workspace` 功能，需要工程根目录下存在 `pnpm-workspace.yaml` 配置文件，并且在 `pnpm-workspace.yaml` 中指定工作空间的目录。比如这里我们所有的子包都是放在 `packages` 目录下，因此修改 `pnpm-workspace.yaml` 内容如下：

```yaml
packages:
  - 'packages/*'
```

初始化完毕后的工程目录结构如下：

```
.
├── README.md
├── package.json
├── packages
│   ├── pkg1
│   │   └── package.json
│   └── pkg2
│       └── package.json
└── pnpm-workspace.yaml
```
### 安装依赖包

使用 `pnpm` 安装依赖包一般分以下几种情况：

- **全局的公共依赖包，比如打包涉及到的 `rollup`、`typescript` 等**

`pnpm` 提供了 [-w, --workspace-root](https://pnpm.io/zh/pnpm-cli#-w---workspace-root) 参数，可以将依赖包安装到工程的根目录下，作为所有package的公共依赖。

比如：

```bash
$ pnpm install react -w
```

如果是一个开发依赖的话，可以加上 `-D` 参数，表示这是一个开发依赖，会装到 `pacakage.json` 中的 `devDependencies` 中，比如：

```bash
$ pnpm install rollup -w -D
```

- **给某个package单独安装指定依赖**

`pnpm` 提供了 [--filter](https://pnpm.io/zh/filtering) 参数，可以用来对特定的package进行某些操作。

因此，如果想给 pkg1 安装一个依赖包，比如 `axios`，可以进行如下操作：

```bash
$ pnpm add axios --filter @qftjs/monorepo1
```

需要注意的是，`--filter` 参数跟着的是package下的 `package.json` 的 `name` 字段，并不是目录名。

关于 `--filter` 操作其实还是很丰富的，比如执行 pkg1 下的 scripts 脚本：

```bash
$ pnpm test --filter @qftjs/monorepo1
```

`filter` 后面除了可以指定具体的包名，还可以跟着匹配规则来指定对匹配上规则的包进行操作，比如：
```bash
$ pnpm build --filter "./packages/**"
```
此命令会执行所有 package 下的 `build` 命令。具体的用法可以参考[filter](https://pnpm.io/zh/filtering)文档。

- **模块之间的相互依赖**

最后一种就是我们在开发时经常遇到的场景，比如 pkg1 中将 pkg2 作为依赖进行安装。

基于 pnpm 提供的 `workspace:协议`，可以方便的在 packages 内部进行互相引用。比如在 pkg1 中引用 pkg2：

```bash
$ pnpm install @qftjs/monorepo2 -r --filter @qftjs/monorepo1
```

此时我们查看 pkg1 的 `package.json`，可以看到 `dependencies` 字段中多了对 `@qftjs/monorepo2` 的引用，以 `workspace:` 开头，后面跟着具体的版本号。

```json
{
  "name": "@qftjs/monorepo1",
  "version": "1.0.0",
  "dependencies": {
    "@qftjs/monorepo2": "workspace:^1.0.0",
    "axios": "^0.27.2"
  }
}
```

在设置依赖版本的时候推荐用 `workspace:*`，这样就可以保持依赖的版本是工作空间里最新版本，不需要每次手动更新依赖版本。

当 `publish` 的时候，会自动将 `package.json` 中的 `workspace` 修正为对应的版本号。

### 只允许pnpm

当在项目中使用 `pnpm` 时，如果不希望用户使用 `yarn` 或者 `npm` 安装依赖，可以将下面的这个 `preinstall` 脚本添加到工程根目录下的 `package.json`中：

```json
{
  "scripts": {
    "preinstall": "npx only-allow pnpm"
  }
}
```

[preinstall](https://docs.npmjs.com/cli/v6/using-npm/scripts#pre--post-scripts) 脚本会在 `install` 之前执行，现在，只要有人运行 `npm install` 或 `yarn install`，就会调用 [only-allow](https://github.com/pnpm/only-allow) 去限制只允许使用 `pnpm` 安装依赖。

## Release工作流

在 `workspace` 中对包版本管理是一个非常复杂的工作，遗憾的是 `pnpm` 没有提供内置的解决方案，一部分开源项目在自己的项目中自己实现了一套包版本的管理机制，比如 [Vue3](https://github.com/vuejs/vue-next)、[Vite](https://github.com/vitejs/vite) 等。

pnpm 推荐了两个开源的版本控制工具：

- [changesets](https://github.com/changesets/changesets)
- [rush](https://rushjs.io/)

这里我采用了 [changesets](https://github.com/changesets/changesets) 来做依赖包的管理。选用 `changesets` 的主要原因还是文档更加清晰一些，个人感觉上手比较容易。

按照 [changesets 文档](https://github.com/changesets/changesets/blob/main/docs/intro-to-using-changesets.md)介绍的，changesets主要是做了两件事：

> Changesets hold two key bits of information: a version type (following semver), and change information to be added to a changelog.

简而言之就是管理包的version和生成changelog。

### 配置changesets

- 安装

```bash
$ pnpm add -DW @changesets/cli
```
- 初始化

```bash
$ pnpm changeset init
```

执行完初始化命令后，会在工程的根目录下生成 `.changeset` 目录，其中的 `config.json` 作为默认的 `changeset` 的配置文件。

修改配置文件如下：

```json
{
  "$schema": "https://unpkg.com/@changesets/config@2.0.0/schema.json",
  "changelog": "@changesets/cli/changelog",
  "commit": false,
  "linked": [["@qftjs/*"]],
  "access": "public",
  "baseBranch": "main",
  "updateInternalDependencies": "patch",
  "ignore": [],
  "___experimentalUnsafeOptions_WILL_CHANGE_IN_PATCH": {
      "onlyUpdatePeerDependentsWhenOutOfRange": true
  }
}
```

说明如下：

- changelog: changelog 生成方式
- commit: 不要让 `changeset` 在 `publish` 的时候帮我们做 `git add`
- linked: 配置哪些包要共享版本
- access: 公私有安全设定，内网建议 restricted ，开源使用 public
- baseBranch: 项目主分支
- updateInternalDependencies: 确保某包依赖的包发生 upgrade，该包也要发生 version upgrade 的衡量单位（量级）
- ignore: 不需要变动 version 的包
- ___experimentalUnsafeOptions_WILL_CHANGE_IN_PATCH: 在每次 version 变动时一定无理由 patch 抬升依赖他的那些包的版本，防止陷入 major 优先的未更新问题

### 如何使用changesets

一个包的发布流程一把分如下几个步骤：

为了便于统一管理所有包的发布过程，在工程根目录下的 `pacakge.json` 的 `scripts` 中增加如下几条脚本：

1. 编译阶段，生成构建产物

```json
{
  "build:packs": "pnpm --filter=@qftjs/* run build"
}
```

2. 清理构建产物和 `node_modules`

```json
{
  "clear": "rimraf 'packages/*/{lib,node_modules}' && rimraf node_modules"
}
```

3. 执行 `changeset`，开始交互式填写变更集，这个命令会将你的包全部列出来，然后选择你要更改发布的包

```json
{
  "changeset": "changeset"
}
```

4. 执行 `changeset version`，修改发布包的版本

```json
{
  "version-packages": "changeset version"
}
```

5. 构建产物后发版
6. 
```json
{
  "release": "pnpm build && pnpm release:only",
  "release:only": "changeset publish --registry=https://registry.npmjs.com/"
}
```

## 参考链接

[用 PNPM Workspaces 替换 Lerna + Yarn](https://juejin.cn/post/7071992448511279141)