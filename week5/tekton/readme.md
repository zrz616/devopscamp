# tekton
tekton有几个核心概念`task`、`pipeline`、`workspace`、`Trigger`

## task
- 一个task是一段代码，用于完成特定的工作。Task 可以包含多个step
- tekton提供task CRD对象描述task，`taskRun` CRD对象用来执行task

例如: 
``` yaml
apiVersion: tekton.dev/v1
kind: TaskRun
metadata:
  name: run-git-clone
spec:
  # 在安装 jenkins 时已经提前部署了 serviceaccount
  serviceAccountName: build-bot
  taskRef:
    name: git-clone
  params:
    - name: url
      value: "https://github.com/zrz616/vote-app.git"
  workspaces:
    - emptyDir: {}
      name: output
```
- 其中taskRef指向的`git-clone`是由`hub.tekton.dev`提供的`task`, 参考：https://hub.tekton.dev/tekton/task/git-clone
- 在k8s集群中apply上述yaml，tekton将执行git clone的任务

![Alt text](image.png)

## pipeline
- pipeline是一组tasks, 按顺序串联起来
- 每个task都会生成一个artifact(输出), 然后传给下一个task
- 所有task的output artifacts被收集到一个单一目录中

### workspaces
- 存放源文件和构建结果的地方
- 这个目录只能在pod内使用, pod之间不共享
- 需要在pipelineRun中配置workspaces


### example
``` yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: github-pipeline-ex1
spec:
  taskRunSpecs:
    - pipelineTaskName: git-clone
      serviceAccountName: build-bot
  pipelineRef:
    name: github-pipeline-ex1
  params:
    - name: git_url
      value: "https://github.com/devops-advanced-camp/vote.git"
  workspaces:
    - name: output
      persistentVolumeClaim:
        claimName: pipeline-pvc
      subPath: $(context.pipelineRun.uid)
```

- 执行上面的PipelineRun，可以在dashboard上看到：

![Alt text](image-1.png)

- 在TaskRun下也出现了对应的TaskRun

![Alt text](image-2.png)

## ex2 推送镜像
- 通过workspaces传入harbor的凭据,已经提前设置好名为docker-credentials的secret
- Pipeline CRD中定义workspaces
- 通过PipelineRun将secret挂载到workspaces上

``` yaml
# pipeline
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: github-pipeline-ex2
spec:
  workspaces:
    - name: output
    - name: docker-credentials
  params:
    - name: git_url
  tasks:
    # task 1: git-clone
    - name: git-clone
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: output
      params:
        - name: url
          # 指向 pipeline 定义的 git_url 参数
          value: $(params.git_url)
    # task 2: build image with kaniko
    - name: kaniko
      taskRef:
        name: kaniko
      runAfter:
        - git-clone
      workspaces:
        - name: source
          workspace: output
        - name: dockerconfig
          workspace: docker-credentials
      params:
        - name: IMAGE
          value: harbor.blkuo301.devopscamp.us/vote/result:latest
        - name: DOCKERFILE
          value: result/Dockerfile
        - name: CONTEXT
          value: result
        - name: EXTRA_ARGS
          value:
            - "--insecure"
---
# PipelineRun
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: github-pipeline-ex2
spec:
  taskRunSpecs:
    - pipelineTaskName: git-clone
      serviceAccountName: build-bot
  pipelineRef:
    name: github-pipeline-ex2
  params:
    - name: git_url
      value: "https://github.com/zrz616/vote-app.git"
  workspaces:
    - name: output
      persistentVolumeClaim:
        claimName: pipeline-pvc
      subPath: $(context.pipelineRun.uid)
    - name: docker-credentials
      secret:
        secretName: docker-credentials
```

- 可以看见任务进度

![Alt text](image-3.png)

- 在harbor中也可以看见成功推送了

![Alt text](image-4.png)

## ex3 大仓构建
还能进一步改造pipeline，提高复用性

``` yaml
# Pipeline
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: github-pipeline
spec:
  workspaces:
    - name: output
    - name: docker-credentials
  params:
    - name: git_url
    # git repository name
    - name: git_repository
      default: "vote"
    # microservice name(build path)
    - name: microservice
      default: "result"
    - name: registry_url
      default: ""
    # - name: sonar_host_url
    #   default: "http://sonarqube-sonarqube.sonarqube.svc.cluster.local:9000"
    - name: commit_id
      default: "latest"
  tasks:
    - name: git-clone
      taskRef:
        name: git-clone
      workspaces:
        - name: output
          workspace: output
      params:
        - name: url
          value: $(params.git_url)
    - name: kaniko
      taskRef:
        name: kaniko
      runAfter:
        - git-clone
      workspaces:
        - name: source
          workspace: output
        - name: dockerconfig
          workspace: docker-credentials
      params:
        - name: IMAGE
          value: $(params.registry_url)/$(params.git_repository)/$(params.microservice):$(params.commit_id)
        - name: DOCKERFILE
          value: $(params.microservice)/Dockerfile
        - name: CONTEXT
          value: $(params.microservice)
        - name: EXTRA_ARGS
          value:
            - "--insecure"
---
# PipelineRun
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: github-pipeline
spec:
  taskRunSpecs:
    - pipelineTaskName: git-clone
      serviceAccountName: build-bot
  pipelineRef:
    name: github-pipeline
  params:
    - name: git_url
      value: "https://github.com/zrz616/vote-app.git"
    - name: microservice
      value: "result"
    - name: git_repository
      value: "vote"
    - name: commit_id
      value: "latest"
    - name: registry_url
      value: "harbor.blkuo301.devopscamp.us"
  workspaces:
    # - name: output
    #   volumeClaimTemplate:
    #     spec:
    #       accessModes:
    #         - ReadWriteOnce
    #       resources:
    #         requests:
    #           storage: 20Mi
    - name: output
      persistentVolumeClaim:
        claimName: pipeline-pvc
      subPath: $(context.pipelineRun.uid)
    - name: docker-credentials
      secret:
        secretName: docker-credentials

```

- 通过`microservice`可以指定目录构建

![Alt text](image-5.png)

只需要修改pipelineRun中的microservice参数就行了
``` yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  name: github-pipeline
spec:
  taskRunSpecs:
    - pipelineTaskName: git-clone
      serviceAccountName: build-bot
  pipelineRef:
    name: github-pipeline
  params:
    - name: git_url
      value: "https://github.com/zrz616/vote-app.git"
    - name: microservice
      value: "vote"
    - name: git_repository
      value: "vote"
    - name: commit_id
      value: "latest"
    - name: registry_url
      value: "harbor.blkuo301.devopscamp.us"
  workspaces:
    # - name: output
    #   volumeClaimTemplate:
    #     spec:
    #       accessModes:
    #         - ReadWriteOnce
    #       resources:
    #         requests:
    #           storage: 20Mi
    - name: output
      persistentVolumeClaim:
        claimName: pipeline-pvc
      subPath: $(context.pipelineRun.uid)
    - name: docker-credentials
      secret:
        secretName: docker-credentials
```
- 可以看见vote仓库也被构建出来了

![Alt text](image-6.png)

## ex4 github event trigger
- 在ex3的基础上，我们添加一个webhook,这样每次有push事件
- 使用tekton的event trigger需要提前安装好依赖的CRD

``` shell
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml
kubectl apply -f https://storage.googleapis.com/tekton-releases/triggers/latest/interceptors.yaml
```

- 然后会出现一个名为`el-github-listener`的pod，这里用ingress将他的8080端口暴露出来给github webhook使用, 对应路径：https://tekton.blkuo301.devopscamp.us/hooks
- 接下来配置`eventListener`和`triggerTemplate`

``` yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: github-template
spec:
  params:
    - name: git_url
    - name: commit_id
    - name: microservice
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: github-run-
      spec:
        taskRunSpecs:
          - pipelineTaskName: git-clone
            serviceAccountName: build-bot
        pipelineRef:
          name: github-pipeline
        workspaces:
          # - name: output
          #   volumeClaimTemplate:
          #     spec:
          #       accessModes:
          #         - ReadWriteOnce
          #       resources:
          #         requests:
          #           storage: 50Mi
          - name: output
            persistentVolumeClaim:
              claimName: pipeline-pvc
            subPath: $(context.pipelineRun.uid)
          - name: sonar-credentials
            secret:
              secretName: sonar-user-pass
          - name: docker-credentials
            secret:
              secretName: docker-credentials
        params:
          - name: git_url
            value: $(tt.params.git_url) # 注意：$tt 开头，从 EventListener binding params 获取
          - name: microservice
            value: $(tt.params.microservice)
          - name: git_repository
            value: "vote"
          - name: commit_id
            value: $(tt.params.commit_id)
          - name: registry_url
            value: "harbor.wangwei.devopscamp.us"
---
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: github-listener
spec:
  serviceAccountName: tekton-triggers
  triggers:
    - name: github-push-events-trigger-result
      interceptors:
        - ref:
            name: github
          params:
            # - name: secretRef
            #   value:
            #     secretName: github-secret
            #     secretKey: secretToken
            - name: "eventTypes"
              value:
                - pull_request
                - push
            - name: "addChangedFiles"
              value:
                enabled: true
        - ref:
            name: cel
          params:
            - name: filter
              # execute only when a file within the controllers directory has changed
              value: extensions.changed_files.matches('result/')
      bindings:
        - name: git_url
          value: $(body.repository.clone_url)
        - name: commit_id
          value: $(body.after)
        - name: microservice
          value: "result"
      template:
        ref: github-template

    - name: github-push-events-trigger-vote
      interceptors:
        - ref:
            name: github
          params:
            # - name: secretRef
            #   value:
            #     secretName: github-secret
            #     secretKey: secretToken
            - name: "eventTypes"
              value:
                - pull_request
                - push
            - name: "addChangedFiles"
              value:
                enabled: true
        - ref:
            name: cel
          params:
            - name: filter
              # execute only when a file within the controllers directory has changed
              value: extensions.changed_files.matches('vote/')
      bindings:
        - name: git_url
          value: $(body.repository.clone_url)
        - name: commit_id
          value: $(body.after)
        - name: microservice
          value: "vote"
      template:
        ref: github-template

    - name: github-push-events-trigger-worker
      interceptors:
        - ref:
            name: github
          params:
            # - name: secretRef
            #   value:
            #     secretName: github-secret
            #     secretKey: secretToken
            - name: "eventTypes"
              value:
                - pull_request
                - push
            - name: "addChangedFiles"
              value:
                enabled: true
        - ref:
            name: cel
          params:
            - name: filter
              # execute only when a file within the controllers directory has changed
              value: extensions.changed_files.matches('worker/')
      bindings:
        - name: git_url
          value: $(body.repository.clone_url)
        - name: commit_id
          value: $(body.after)
        - name: microservice
          value: "worker"
      template:
        ref: github-template
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tekton-triggers
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: triggers-eventlistener-binding
subjects:
  - kind: ServiceAccount
    name: tekton-triggers
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-roles
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: triggers-eventlistener-clusterbinding
subjects:
  - kind: ServiceAccount
    name: tekton-triggers
    namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-clusterroles
```

- 尝试更新github代码仓库，可以看到流水线被触发

![Alt text](image-7.png)
