## 对k8s使用的演进路线

**第一阶段:** 开发项目，将项目打包为docker镜像，发布到docker仓库，编辑yaml文件，部署到k8s集群

**存在的问题**: yaml文件和dockerfile处于没有管理或手动管理的状态，多次部署之后，集群大概就非常混乱了，项目如何打包，打包后如何部署，没有标准化的流程，难以自动化（此处自动化指：通过简单的几条命令搞定，而不是填一大堆参数去定制一个复杂的流程甚至于编写一大堆代码去运行，下同），难以追溯历史状态

**第二阶段:** 在第一阶段的基础上，将dockerfile和代码存放在同一个git仓库中，使用自动化工具，自动打包docker镜像，上传到docker镜像仓库，基本做到ci

**存在的问题**： yaml文件仍然基本没啥管理（既然已经存储dockerfile，为何不直接连yaml也存储了呢？），打包容易自动化了，但是部署仍然难以自动化

**第三阶段:** 在第二阶段的基础上，进一步把yaml文件也存储在git仓库中，并加入一个文件（gitlab ci或github workflow或drone ci等ci工具的描述文件）去描述根据dockerfile打包和部署（应用yaml到目标集群）的整个流程

好处：相比于之前，完全地实现了自动化，并突破性地做到了可以仅通过提交代码这一个动作（也可以加个企业微信通知，然后手动点击触发部署），即完成部署到环境的任务（理想状况）

**存在的问题**：

1. dockerfile文件有点多余，在docker镜像中做的操作一般是复制文件设置环境变量，执行一些命令，这些都可以在deployment的yaml中定义
2. 对基础设施的定义（比如prometheus，grafana，loki等的定义）不应当存储在任何项目仓库中，因为它们不隶属于某个项目（需要基础设施即代码IaC？），而且这些开源组件大多是用helm或者operator这种方式去部署的，而不是直接用yaml文件（虽然最终会被渲染为yaml部署到k8s)
3. 如果多项目都使用这个流程，并部署在同一个集群当中，则可能出现资源冲突，比如都定义了一个prometheus，都定义了一个grafana，(虽然可以通过人员组织架构的规则来命名，但这是不够程序化而是依靠程序员心智的方法，而且对冲突的发现不够直接)
4. 废弃资源的清理，比如上一个版本定义了一个名字叫做the-cm 的configmap，下一个版本没有了，在重新 apply yaml之后，the-cm资源不会删除

针对上述问题的解决：

1. 去掉dockerfile，将dockerfile中的操作放到yaml中的deployment资源中
2. 将基础设施的定义放在一个单独的git仓库中，在该git仓库中添加自动apply的流水线（这也会导致上述的问题5）
3. 将所有yaml文件存储在一个git仓库，其他项目的gitlab ci不再是部署yaml到集群，而是提交到git-ops-repo仓库，再由git-ops-repo中的流水线部署到k8s集群中，于是所有的资源冲突都可以在git上被直接观测到（来自不同项目的提交，修改了同一个资源文件）
4. 我们需要对上一次apply的yaml与这次的yaml对比，生成patch计划，对现有资源进行处理，包括删除，更新，创建。如果要自己完成，这是一个非常复杂而困难的过程，涉及到上次状态的保存，以及两次状态的diff，patch的生成。所幸的是，社区有很多可以用来做这件事的工具：pulumi，terraform，flux cd，（这三个有较多的了解）以及argo cd,ansible jenkinsX（这三个没啥太多的了解）。

综合2，3，4，我认为：应该用一个独立的git仓库（git-ops-repo），去描述集群的desire状态，包括基础设施和工作负载（工作负载也就是各个项目），工作负载的yaml，由各个项目的流水线提交，基础设施的yaml由开发者直接提交，通过某种iac或gitops工具（flux cd），同步到集群，并处理掉废弃资源

### 于是最终流程如下：

 ![Alt text](file:///Users/weicl/Library/Application%20Support/marktext/images/945267411faf7ae40f6b49b68ea97cd8e0b2c8d3.png) 

# 理论部分：

## [什么是基础架构即代码（IaC）](https://www.redhat.com/zh/topics/automation/what-is-infrastructure-as-code-iac)

基础架构即代码 (IaC) 是通过代码（而非手动流程）来管理和置备基础架构的方法。

利用 IaC，我们可以创建包含基础架构规范的配置文件，从而便于编辑和分发配置。此外，它还可确保每次置备的环境都完全相同。

通过对配置规范进行整理和记录，IaC 有助于实现[配置管理](https://www.redhat.com/zh/topics/automation/what-is-configuration-management)，并避免发生未记录的临时配置更改。

版本控制是 IaC 的一个重要组成部分，就像其他任何软件源代码文件一样，配置文件也应该在源代码控制之下。

## [什么是GitOps](https://www.redhat.com/zh/topics/devops/what-is-gitops)

GitOps 使用 Git 拉取请求来自动管理基础架构的置备和部署。Git 存储库包含系统的全部状态，因此系统状态的修改痕迹既可查看也可审计。

GitOps 围绕开发者经验而构建，可帮助团队使用与软件开发相同的工具和流程来管理基础架构。除了 Git 以外，GitOps 还支持您按照自己的需求选择工具。

人们认为是 Weaveworks 创造了 GitOps 这一术语。

## GitOps和IaC的关系

我们可以认为 GitOps 是[基础架构即代码（IaC）](https://www.redhat.com/zh/topics/automation/what-is-infrastructure-as-code-iac) 的一个演变，以 Git 为基础架构配置的版本控制系统。IaC 通常采用声明式方法来管理基础架构，定义系统的期望状态，并跟踪系统的实际状态。

与 IaC 一样，GitOps 也要求您声明式描述系统的期望状态。通过使用声明式工具，您所有的配置文件和源代码都可以在 Git 中进行版本控制。

# 工具

## [terraform](https://www.terraform.io/)

## [pulumi](https://www.pulumi.com/)

## [flux cd](https://fluxcd.io/)
