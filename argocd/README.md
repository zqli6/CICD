# Argo CD 应用部署指南：Git 仓库、认证、Application 与 ApplicationSet

本指南适用于 **Argo CD v2.x** 稳定版本，覆盖从 Git 仓库规划到 Webhook 触发的完整部署流程。


# 1. Git 仓库目录结构

Argo CD 以 **Application** 为最小部署单元，每个 Application 对应 Git 仓库中**一个确定的路径**。该路径下的清单被整体渲染和同步。

```text
gitops-repo/
├── apps/                         # 可部署单元的根目录；每个子目录对应一个 Application 的 path
│   ├── web-frontend/             # 一个部署单元：可以是 Helm Chart、Kustomize overlay 或纯清单目录
│   │   ├── Chart.yaml            # 若存在，Argo CD 倾向按 Helm 处理；纯清单目录不应包含此文件
│   │   ├── values.yaml           # Helm Chart 的默认值文件
│   │   ├── templates/            # Helm 模板目录
│   │   │   ├── deployment.yaml   # 工作负载清单（由 Helm 渲染）
│   │   │   └── service.yaml      # Service 清单
│   │   └── kustomization.yaml    # 若存在，Argo CD 倾向按 Kustomize 处理；与 Chart.yaml 互斥
│   ├── api-service/
│   │   ├── deployment.yaml       # 纯清单：无 Chart.yaml / kustomization.yaml，直接作为 Kubernetes 资源应用
│   │   └── service.yaml
│   └── worker/
│       └── deployment.yaml
├── environments/                 # 可选：按环境组织的 Kustomize overlays 或 Helm values
│   ├── dev/                      # dev 环境的 overlay / values
│   ├── staging/
│   └── production/
└── argocd/                       # 可选：Application / ApplicationSet 定义本身（“管理应用的清单”）
    ├── web-frontend-app.yaml     # 一个 Application 资源
    └── apps-applicationset.yaml  # 一个 ApplicationSet 资源，用于自动生成多个 Application
```

**核心规则**：`apps/` 下每个子目录是一个独立路径。纯清单目录**不应**包含 `Chart.yaml` 或 `kustomization.yaml`，否则 Argo CD 会按对应工具处理，而不是当作原生 YAML 直接应用。


# 2. Git 仓库连接认证

凭据以 **Kubernetes Secret** 形式存储在 `argocd` 命名空间中，通过标签 `argocd.argoproj.io/secret-type` 区分用途。SSH 与 HTTPS 二选一即可。

## 2.1 SSH 私钥方式

**步骤 1：生成 SSH 密钥对**（本地执行）：

```bash
ssh-keygen -t ed25519 -C "argocd@myorg.com" -f argocd-key -N ""
```

**步骤 2：将公钥添加到 Git 平台**。以 GitLab 为例：仓库 **Settings → Repository → Deploy keys**，粘贴 `argocd-key.pub` 内容。Argo CD 仅需读权限。

**步骤 3：创建 repository Secret**，填入**私钥**：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitops-repo-ssh
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository   # 固定标签：声明这是一个“仓库连接”Secret
stringData:
  type: git
  url: git@gitlab.com:myorg/gitops-repo.git
  sshPrivateKey: |                                # 填入 SSH 私钥内容（不是公钥）
    -----BEGIN OPENSSH PRIVATE KEY-----
    ...
    -----END OPENSSH PRIVATE KEY-----
```

或使用 CLI：

```bash
argocd repo add git@gitlab.com:myorg/gitops-repo.git \
  --ssh-private-key-path ./argocd-key
```

> Argo CD 需要信任 Git 主机的 SSH host key。自建 GitLab 可能需要将主机 key 加入 `argocd-ssh-known-hosts-cm` ConfigMap；`insecure: "true"` 可临时跳过校验，生产环境不建议。

## 2.2 HTTPS 用户名 + Token 方式

**步骤 1：在 GitLab 生成 Token**（`read_repository` 权限的 Personal Access Token 或 Project Access Token）。

**步骤 2：创建 repository Secret**：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitops-repo-https
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository   # 固定标签：仓库连接 Secret
stringData:
  type: git
  url: https://gitlab.com/myorg/gitops-repo.git   # GitLab 要求 .git 后缀，否则 301 重定向不被跟随
  username: oauth2                                # GitLab 推荐使用 oauth2 作为用户名
  password: glpat-xxxxxxxxxxxx                    # 填入生成的 Token
```

CLI：

```bash
argocd repo add https://gitlab.com/myorg/gitops-repo.git \
  --username oauth2 \
  --password glpat-xxxxxxxxxxxx
```

**可选：按 URL 前缀复用凭据**（多个仓库同一凭据源时使用 `repo-creds`）：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-group-creds
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repo-creds   # 与 repository 不同：按 URL 前缀匹配，而非精确仓库
stringData:
  type: git
  url: https://gitlab.com/myorg                   # 此前缀下的所有仓库自动匹配此凭据
  username: oauth2
  password: glpat-xxxxxxxxxxxx
```


# 3. 情况一：使用 Application 手动管理

适用于应用数量少、需要逐个精细控制参数的场景。每个应用对应一个 `Application` 资源。

## 3.1 Helm 应用

`path` 指向包含 `Chart.yaml` 的目录，`source.helm` 配置 value 文件。

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: web-frontend
  namespace: argocd                              # Application 资源本身放在 argocd 命名空间
spec:
  project: default
  source:
    repoURL: https://gitlab.com/myorg/gitops-repo.git
    targetRevision: main
    path: apps/web-frontend                      # 指向包含 Chart.yaml 的目录
    helm:
      valueFiles:
        - values-production.yaml                 # 相对于 path 的 values 文件
  destination:
    server: https://kubernetes.default.svc       # 目标集群 API Server（集群内可直接用此地址）
    namespace: web-frontend                      # 工作负载部署到的命名空间
  syncPolicy:
    automated:
      prune: true                                # Git 中移除的资源，集群中自动删除
      selfHeal: true                             # 集群中手动修改的漂移，自动回退
    syncOptions:
      - CreateNamespace=true                     # 目标命名空间不存在时自动创建
```

## 3.2 Kustomize 应用

`path` 指向包含 `kustomization.yaml` 的 overlay 目录：

```yaml
spec:
  source:
    repoURL: https://gitlab.com/myorg/gitops-repo.git
    targetRevision: main
    path: environments/production/web-frontend   # 指向 Kustomize overlay 目录
    kustomize:
      images:
        - web-frontend=registry.example.com/web-frontend:v1.2.3   # 覆盖镜像 tag
```

## 3.3 纯 YAML 清单资源

目录中**仅**放置 `deployment.yaml`、`service.yaml` 等原生清单，**不写** `helm` / `kustomize` 段：

```yaml
spec:
  source:
    repoURL: https://gitlab.com/myorg/gitops-repo.git
    targetRevision: main
    path: apps/api-service                       # 该目录内仅含原生 YAML/JSON，无 Chart.yaml / kustomization.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: api-service
```

Argo CD 会将 `path` 下的 YAML/JSON 作为清单直接应用。如果目录中意外出现 `Chart.yaml` 或 `kustomization.yaml`，需在 `source` 中显式指定 `directory` 类型，否则可能被按对应工具处理。


# 4. 情况二：使用 ApplicationSet 自动生成

ApplicationSet 通过 **Git directory generator** 扫描仓库目录结构，按 **template** 批量生成 Application。新增目录时无需手写新的 Application 资源。

## 4.1 基本 ApplicationSet（覆盖纯清单 / 混合目录）

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monorepo-apps
  namespace: argocd
spec:
  goTemplate: true
  goTemplateOptions: ["missingkey=error"]        # 模板中引用了不存在的参数时直接报错，避免静默生成错误 Application
  generators:
    - git:
        repoURL: https://gitlab.com/myorg/gitops-repo.git
        revision: main
        directories:
          - path: apps/*                         # 匹配 apps/ 下所有一级子目录作为生成单元
  template:
    metadata:
      name: '{{.path.basename}}'                 # 用目录名作为 Application 名（如 web-frontend、api-service）
    spec:
      project: default
      source:
        repoURL: https://gitlab.com/myorg/gitops-repo.git
        targetRevision: main
        path: '{{.path.path}}'                   # 由 generator 注入的完整目录路径（如 apps/web-frontend）
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{.path.basename}}'          # 默认以目录名作为目标命名空间
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

`{{.path.path}}` 是匹配到的完整目录路径，`{{.path.basename}}` 是最右侧目录名。只要 `apps/` 下新增子目录，ApplicationSet 控制器会自动检测并创建对应 Application。

## 4.2 排除特定目录

如果某些目录不应被自动生成 Application，使用 `exclude: true`：

```yaml
directories:
  - path: apps/*                                # 先包含全部
  - path: apps/do-not-deploy                    # 再排除特定目录
    exclude: true                               # exclude 优先级高于 include
```

## 4.3 Web 应用（Helm / Kustomize）的 template 调整

Directory generator 本身不判断工具类型，差异体现在 `template.spec.source` 中。若 `apps/web-frontend` 是 Helm Chart，在 template 的 `source` 下追加：

```yaml
source:
  repoURL: https://gitlab.com/myorg/gitops-repo.git
  targetRevision: main
  path: '{{.path.path}}'
  helm:
    valueFiles:
      - values-production.yaml                  # 相对于 path 的 Helm values 文件
```

若为 Kustomize overlay，替换为 `kustomize:` 段即可。若同一 ApplicationSet 下混合了纯清单目录和 Helm Chart 目录，单纯 directory generator 无法一次性适配，可考虑 matrix generator 结合 list generator 按路径模式分别配置 `source`。

## 4.4 删除 ApplicationSet 时保留工作负载

默认情况下，删除 ApplicationSet 会触发其生成的 Application 被级联删除，进而删除集群中的工作负载。若需要**仅移除 ApplicationSet / Application 定义，保留运行中的 Kubernetes 资源**，在 `spec.syncPolicy` 中设置：

```yaml
spec:
  syncPolicy:
    preserveResourcesOnDeletion: true             # 删除 ApplicationSet 时，不删除其生成的 Application 所管理的工作负载
```

该配置使 ApplicationSet controller **不添加** Argo CD 的资源 Finalizer，工作负载作为孤儿资源继续运行。

> **与单 Application 的区别**：针对单个 Application 执行孤儿删除，使用 `kubectl delete application <name> -n argocd --cascade=orphan`。`preserveResourcesOnDeletion` 是 ApplicationSet 层面的声明式配置，从源头长期阻止级联删除。


# 5. GitLab Webhook 配置

Argo CD 默认每 **3 分钟**轮询 Git 仓库。Webhook 将检测延迟降至近实时。

## 5.1 在 Argo CD 中配置 Webhook Secret

Argo CD 使用 `argocd-secret` 中的 **`webhook.gitlab.secret`** 字段验证 GitLab 请求。

**步骤 1：生成 Secret**：

```bash
openssl rand -hex 32
```

**步骤 2：写入 `argocd-secret`**：

```bash
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {"webhook.gitlab.secret": "<上一步生成的值>"}}'
```

或直接在 `argocd-secret` 的 `stringData` 中加入：

```yaml
stringData:
  webhook.gitlab.secret: "<生成的值>"             # GitLab webhook 的共享 Secret，必须与 GitLab 侧填写的值完全一致
```

**步骤 3：重启 `argocd-server`**：

```bash
kubectl -n argocd rollout restart deployment argocd-server
```

Webhook Secret 在 Argo CD 侧是**可选的**，但不配置时任何知道 endpoint 的人都可以发送伪造事件；如果 Argo CD 可公网访问，**强烈建议配置**。

## 5.2 在 GitLab 中配置 Webhook

进入 GitLab 项目 → **Settings → Webhooks → Add new webhook**：

| 字段 | 值 |
|---|---|
| **URL** | `https://<argocd-server>/api/webhook`（固定端点，不要遗漏 `/api/webhook`） |
| **Secret token** | 与 `webhook.gitlab.secret` **完全一致** |
| **Trigger** | 至少勾选 **Push events** |
| **SSL verification** | 生产环境保持开启；自签名证书测试环境可临时关闭 |

保存后 GitLab 会发送测试请求，Argo CD `/api/webhook` 返回 **200** 即表示连通。

## 5.3 验证

1. **GitLab 侧**：Webhook 详情页 **Recent events** 查看请求响应状态。
2. **Argo CD 侧**：`kubectl -n argocd logs deploy/argocd-server` 中应有接收 webhook 并触发 reconciliation 的日志。
3. **行为验证**：修改 Git 中某个 Application 路径下的清单并 push，配置了 `automated` 同步的 Application 应在秒级触发同步，而非等待 3 分钟轮询。


# 6. 选用建议

| 维度 | Application 手动管理 | ApplicationSet 自动生成 |
|---|---|---|
| **适用规模** | 少量应用，需逐参数精细控制 | 多应用，目录结构规范统一 |
| **新增应用** | 新增 Application YAML 并 apply | 只需在 Git 中添加目录，自动发现并生成 Application |
| **Git 目录要求** | 每个 Application 指向一个确定路径 | 依赖 `directories.path` 通配符匹配，目录结构必须规范 |
| **Web 应用 / 纯清单** | `source` 中显式写 `helm:` / `kustomize:`，纯清单不写工具段 | 同理，在 `template.spec.source` 中按类型配置 |
| **删除时保留资源** | `--cascade=orphan` | `preserveResourcesOnDeletion: true` |
| **Webhook 生效范围** | 清单变更即时同步 | 清单变更即时同步；**新增目录**的发现仍依赖 ApplicationSet controller 自身的仓库刷新周期 |

两种方式在**认证**和**Webhook**层面的配置位置与原理完全相同；核心差异在于 Application 资源是由人静态编写，还是由 ApplicationSet 根据 Git 目录结构动态生成。