# Website-deployment-is-too-slow

解决网站部署速度过慢问题

1 我家这个网站是如何部署的？要解决部署问题，首先得知道它是如何部署的
页面资源(html)是否与静态资源(js/css)分离？
静态资源是否上了 CDN？
CDN 提供商是哪一家，它家的 CDN 怎么配置？
静态资源上了 CDN 要注意什么问题？为什么有一次用户反馈只有他们的页面打不开？为什么有一次新版本部署上去但未生效？
静态资源上传 CDN 时，如何提高上传速度，以此来提高网站的部署速度
CDN 回源过多，命中率过低怎么解决？
打包体积终于被我弄小了，依赖安装及打包的时间又过长，那我应该如何优化呢 (又延伸出 CI，webpack 及 npm 若干问题)
安装依赖是否有更有效更快地方法呢？
打包是在 CI 下打包的？CI 又是什么
CI 下如何利用缓存，避免每次都要安装依赖？
如何优化 webpack 打包的时间，原理都是什么
当 webpack 利用多进程多线程打包优化时，我的 CI 有几个 CPU 核心，这一点是否有所帮助
即使 CI 的核心数很多，如果有并行的多个流水线，CI 的 cpu 会不会爆掉
这其中涉及到了大部分前端都不会接触的技术与能力，这对自己的项目实战能力以及工程化能力都有大幅提高。比如

操作系统，特别是其中涉及到的进程与线程，文件系统甚至是 CPU 在部署优化时扮演重要角色
DevOps，如 CI 中的部署优化，甚至可能有最近大火的容器与 docker
CDN，了解网站部署到 CDN 的过程及可能出现的问题
webpack

解决方法1:在生产环境中使你的 npm i 速度提升 50%

对于一个前端应用，或者说是一个 Node 应用，在 CICD pipeline 中，无论是构建，测试，部署，其中必不可少的环节就是依赖安装: npm i。
npm i 不仅是必不可少的环节，而且很可能也是耗时最长的一个环节。

打蛇打七寸，优化应该从瓶颈处开始，如果能从依赖安装下手，将能极大地缩短部署时间，提高产品交付效率，改善 DevOps 流程，从而促进敏捷开发。

CI 环境中的优化不同于本地开发环境，其中最大的不同在于: 在本地环境中安装依赖是有状态的，如 node_modules，~/.npmrc，~/.npm 一系列硬盘目录及文件，无不构成状态。而在生产环境中，往往结合 CICD 工具，每次分配的 Runner 不一定是一台服务器(容器)，往往被视为无状态，因而无法有效利用缓存而导致 CI 中部署用时过长。

但也正因为 CICD Runner 的无状态化，这意味着你只要参考构建脚本，如 .gitlab-ci.yaml，.travis.yaml 或者 .github/workflows/deploy.yaml，就可以从零把项目跑起来，而避免过多在熟悉新项目时求助他人。

不同的部署方式，不同的持续集成工具有不同的实践方法，但优化的原理大同小异。


只安装有必要的库

npm install 的速度显然与它依赖的包的多少是正相关的，那减少包的数量就可以减少装包所花费的时间。这个道题如同每天都少吃一顿饭就会变瘦一样简单。

使用 --production 可以只安装 package.json 中 dependencies 中的依赖项，只把要安装在 CI 中需要使用到的依赖。在生产环境部署时可以指定 --production。

npm install --production
另外，通过设置环境变量 NODE_ENV=production 相当于指定 --production，测试如下:

$ NODE_ENV=production npm config get production
true
这唯一的不足就是需要你谨小慎微战战兢兢，如临深渊如履薄冰地把 packages 分拣到 devDependencies 与 dependencies 中。

使用私有镜像仓库

从 npm 的官方仓库可以发布与拉取 package，比如 react，vue 和 express 都发布在官方仓库中。官方 registory 地址是 https://registry.npmjs.org/，这对国内的前端应用来说，由于远在国外，远程仓库的服务器时延过大，此时的瓶颈在于网络。

国内的公有镜像库是一个不错的选择，如淘宝镜像，它每隔十分钟与官方镜像仓库同步一次，而它位于国内，网络不至于太慢。

对于企业级项目的生产环境来说，在集群内部搭建私有仓库是一个更好的选择，也更能缩短时延。 流量能在局域网完成的，就不需要去互联网中溜圈了。

使用 npm config 来设置 npm 仓库，它将把仓库信息写入 ~/.npmrc 中，这足以适用于 CICD 中。

npm config set registry https://registry-npm.shanyue.tech/
缓存：人不能两次踏进同一条河流

人不能踏进同一条河流两次，一个接口不能连续慢两次，npm install 同样不能慢两次。

正如 http cache 一样，npm install 也可以充分利用缓存，npm 的缓存在 linux 系统中存在于以下路径:

~/.npm
命令 npm cache verify 可以校验缓存，如下所示:

$ npm cache verify
Cache verified and compressed (~/.npm/_cacache):
Content verified: 4415 (156612117 bytes)
Index entries: 6239
Finished in 14.017s
结合 npm install --prefer-offline 缓存优先策略充分利用缓存

但是有一个悖论，在 CI 中是无状态的: 你本次装的 node_modules 和生成的 ~/.npm 将会在下次装包时丢失。

这时候就需要利用 CI 的缓存功能，如以下是 travis ci 的配置

install:
-npmci
# keep the npm cache around to speed up installs
cache:
  directories:
  -"$HOME/.npm"
progress: 屏蔽非必要打印信息

当设置 progress 为 true 时，npm install 时将会显示进度条，把它设置为 false 可小幅提高 npm install 的速度。

修改设置:

npm set progress=false
在 官方文档 中对 progress 有以下介绍：

Default: true, unless TRAVIS or CI env vars set. Type: Boolean
因此，只需要在构建服务器中简单设置一个 CI=true 的环境变量即可以很轻便地解决问题

使用 npm ci 替代 npm i

单看名字，里边有个 CI，它就像适用在 CI 中，事实也如此。它在 CI 环境中做了一系列优化，如去除掉一些面向用户的特性来加强速度。除了性能，它也有一些在 CI 上基于完整性与安全性的检查，如 package.json 与 package-lock.json 版本不一致的问题。

为了更好地提高速度，npm ci 基于一个独立的库 libcipm 安装依赖，而它拥有和 npm install 兼容的 API。并且当它安装依赖时，默认是缓存优先的，它会充分利用缓存，从而加速装包。

经实验，npm ci 可以减少将近一半的的依赖安装时间。

$ npm install
added 1154 packages in 60s

$ npm ci
added 1154 packages in 35s

最后总结一下如何减少 npm install 的时间，都是很简单的设置，一分钟搞定

选择时延低的 registry，需要企业技术基础建设支持
NODE_ENV=production，只安装生产环境必要的包
CI=true，npm 会在此环境变量下自动优化
结合 CI 的缓存功能，充分利用 npm cache
使用 npm ci 代替 npm i，既提升速度又保障应用安全性

（问题1:加了--production不下载开发依赖怎么打包？合理的前端项目应该将打好包的文件也上传 git，而不是在生产环境再打包。问题2:package-lock.json？yarn.lock会把私有地址寄下来造成信息泄露，记得别传到github上啊）

扩展：团队开发代码约束

1:ESLint 与约束
$ npm install eslint --save-dev 

规则集需要统一集中配置，ESLint 会默认读取配置文件 .eslintrc 来解析，而规则集在 rules 中进行配置：

{
  "rules": {
    "semi": ["error", "always"],
    "quotes": ["error", "double"]
  }
}

2:airbnb

airbnb javascript style我们仅仅需要使用 extend 配置项去继承一些优秀的开源的代码规范，并使用 rules 做一些自己团队的规则补充。

{
  "extend": ["airbnb-base"],
  "rules": {
    "semi": ["error", "never"]
  }
}

3:开发环境，生产环境与警告

开发环境对于开发而言重要的是什么？
是开发体验。
一个良好的编码规范会带来解放强迫症的舒适感，但过于严格的代码风格有时也会使人烦躁。试举两个小例子，有可能是在你写代码时出现过的场景：
禁止掉 console.log，避免在生产环境输出多余的东西。但偏偏在测试环境经常需要调试，但是如果仅仅设为警告的话，警告又会被忽视，失去意义。
特别是当设置了规则 no-unused-vars 时。如果仅仅是为了在开发时调试，却因为无法通过 ESlint 规则校验无法方便调试。
这是一个约束与自由的权衡，ESLint 在提供强有力约束时自然会牺牲一些开发上的便利性。中庸，儒家思想讲究中庸，此时可以在权衡下选择一个中庸的方案：
把 ESLint 的所有影响调试的规则校验都设置为 Warn，那你又问了警告往往不是会被忽略吗？是这样子的，所以需要在 CI 中设置环境变量 CI=true，如此在 CI 中即使有警告也无法交付。CI 指持续集成，在本篇文章后边也会接着提到，另外，在本系列文章中也会重点讲解 CI，欢迎持续关注。
如在 create-react-app 中的大部分规则都是设置为 Warn

但是，如果你使用了 webpack，并且结合 eslint-loader，那解决方案就更加简单了：使用 emitWarning: true，在测试环境把所有 Error 都当做 Warn，这样避免了修改 ESLint 规则，webpack 的配置如下
{
  test: /\.(js|mjs|jsx|ts|tsx)$/,
  enforce: 'pre',
  use: [
    {
      options: {
        cache: true,
        emitWarning: true,
      },
      loader: require.resolve('eslint-loader'),
    },
  ]
}
所以有两种权衡开发体验与编程规范的方式：
把 ESLint 的 rule 设置为 Warn，并在持续集成中配置环境变量 CI=true。
结合 webpack 与 eslint-loader，根据当前环境的环境变量配置 emitWarning。

第一层约束: IDE

当不符合代码规范的第一时间，我们就要感知到它，及时反馈，快速纠正，比直到最后积攒了一大堆错误要高效很多。
这里以 VS Code 作为示例，它只需要安装一个插件：eslint，便可以做到智能提示

第二层约束: Git Hooks

git 自身包含许多 hooks，在 commit，push 等 git 事件前后触发执行。与 pre-commit hook 结合可以帮助校验 Lint，如果非通过代码规范则不允许提交。
husky 是一个使 git hooks 变得更简单的工具，只需要配置几行 package.json 就可以愉快的开始工作。
husky 的原理是什么？
// package.json
{
  "scripts": {
    "lint": "eslint . --cache"
  },
  "husky": {
    "hooks": {
      "pre-commit": "npm lint",
    }
  }
}
或者结合 lint-staged[4] 调用校验规则
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged"
    }
  },
  "lint-staged": {
    "*.js|{lib,setup,bin,hot,tooling,schemas}/**/*.js|test/*.js|{test,examples}/**/webpack.config.js}": [
      "eslint --cache"
    ],
    "*.{ts,json,yml,yaml,md}|examples/*.md": [
      "prettier --check"
    ],
    "*.md|{.github,benchmark,bin,examples,hot,lib,schemas,setup,tooling}/**/*.{md,yml,yaml,js,json}": [
      "cspell"
    ]
  }
}
不过做前端的都明白，客户端校验是不可信的，通过一条命令即可绕过 git hooks。
$ git commit -n

 第三层约束: CI
 
 git hooks 可以绕过，但 CI(持续集成) 是绝对绕不过的，因为它在服务端校验。使用 gitlab CI 做持续集成，配置文件 .gitlab-ci.yaml 如下所示
lint:
  stage:lint
  only:
    -/^feature\/.*$/
  script:
    -npmlint
（总结）
团队中代码规范统一是极有必要的
使用成熟的 eslint config，并做细节修改
设置部分 eslint rule 为警告，保障开发体验，并且在 pre-commit 与 CI 中把警告视为不通过，保证严格的代码规范
可以在 IDE (vscode)，git hooks，CI 中添加规范校验拦截
可以使用 husky 与 lint-staged 很方便地做关于 lint 的 git hooks
git hooks 的规范校验可以通过 git commit -n 跳过，需要在 CI 层继续加强校验
