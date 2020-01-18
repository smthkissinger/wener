---
id: gitlab-cicd
title: GitaLab持续集成持续交付
---

# Gitlab CI/CD

## Tips
* `.gitlab-ci.yml`
  * 应用构建定义
  * K8S Runner 下使用 Kaniko 构建镜像
  * 使用 [gitlabktl](https://gitlab.com/gitlab-org/gitlabktl) 部署服务和应用到 knative
* `serverless.yml`
  * 无服务函数 - 需要 Knative
  * 可用于部署 _函数_
  * 默认会生成 Dockerfile
  * Gitlab [运行时](https://gitlab.com/gitlab-org/serverless/runtimes) 支持 go ruby nodejs Dockerfile
  * 可以使用 OpenFaaS 的运行时
* `Dockerfile`
  * 无服务应用
  * 定义应用构建方式、应用运行时
  * 默认需要暴露 8080
* 注意
  * 构建的应用镜像需要 PUSH 到仓库 - 因此需要配置仓库信息
* [CI变量说明](https://docs.gitlab.com/ee/ci/variables)

## serverless
* [knative-examplesknative-examples)

```bash
# 查看部署的函数服务
kubectl -n "$KUBE_NAMESPACE" get services.serving.knative.dev

# 请求
curl \
--header "Content-Type: application/json" \
--request POST \
--data '{"GitLab":"FaaS"}' \
http://functions-echo.functions-1.functions.example.com/

```

__OpenFaaS 运行时__
```yaml
hello:
  source: ./hello
  runtime: openfaas/classic/ruby
  description: "Ruby function using OpenFaaS classic runtime"
```

### echo-js example

__.gitlab.yml__
```yaml
include:
  # 模板继承
  template: Serverless.gitlab-ci.yml

functions:build:
  extends: .serverless:build:functions
  # 设置环境信息
  environment: production

functions:deploy:
  extends: .serverless:deploy:functions
  environment: production

```

__serverless.yml__

```yaml
# 服务名字
service: functions
# 服务描述
description: "GitLab Serverless functions using Knative"

provider:
  # 执行 serverless.yml 的 provider
  name: triggermesh
  # 环境变量
  environment:
    FOO: value

functions:
  echo-js:
    # 函数名字
    handler: echo-js
    # 目录
    source: ./echo-js
    # 运行时 - 可选 - 可以在 source 下定义 Dockerfile
    runtime: https://gitlab.com/gitlab-org/serverless/runtimes/nodejs
    # 函数描述
    description: "node.js runtime function"
    # 环境变量
    environment:
      MY_FUNCTION: echo-js
```

## gitlabktl
* [GitLab Knative tool](https://gitlab.com/gitlab-org/gitlabktl)

```bash
# 无法直接 build 可以使用镜像
docker pull registry.gitlab.com/gitlab-org/gitlabktl
# 作为命令使用
gitlabktl(){ docker run --rm -it ${OPT} -v $PWD:/host -w /host registry.gitlab.com/gitlab-org/gitlabktl gitlabktl $*;}
# 构建
# 需要镜像环境 CI_REGISTRY CI_REGISTRY_USER CI_REGISTRY_PASSWORD
gitlabktl serverless build
# 会在目录下生成 Dockerfile
# 会往 registry-internal.wener.me/demo/函数名 推送镜像
OPT="-e CI_REGISTRY=registry.wener.me -e CI_REGISTRY_USER='' -e CI_REGISTRY_PASSWORD='' -e CI_REGISTRY_IMAGE=registry-internal.wener.me/demo " gitlabktl serverless build
# 运行刚才推送的仓库 - 使用的 functions-echo-js
docker run -it --rm -p 8080:8080 registry-internal.incos.dev/demo/functions-echo-js
```