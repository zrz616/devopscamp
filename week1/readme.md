## 从公有云搭建CI/CD流水线

### terraform
> terraform是一种IaC工具, 可以将基础设置的整个构建过程都体系在代码里
#### 1. 安装terraform
```bash
# macos
brew install terraform
```
#### 2. 配置环境变量
```bash
# aliyun
export TF_VAR_access_key="xxx" # access key id
export TF_VAR_secret_key="<KEY>" # secret key
```
#### 3. init && apply
- 准备好IaC代码后，进入目录执行`terraform init`安装依赖模块
- 可以执行`terraform plan`预先查看将要开通的资源
- 只需要`terraform apply -auto-approve`，就可以开通资源，并进行相应的初始化工作了

### grafana
> Grafana是一个开源的数据可视化和分析软件。它可以帮助您通过图表、仪表板和警报来监

之前的terrraform代码中已经通过Helm Charts安装了grafana，并且内置了各种dashborad，如kubernetes集群监控、coreDNS、ArgoCD等
![Alt text](assets/image.png)
![Alt text](assets/image-1.png)
![Alt text](assets/image-2.png)

### gitlab
> Gitlab是一个基于MIT协议发布的版本控制系统，使用Ruby on Rails开发，自带一个Web界面，支持Git作为唯一的代码
在前面的初始化代码中给gitlab的代码仓库提交了两套代码`camp-go-example`和`camp-go-example-helm`
``` shell
project_response=$(curl -s -I --request GET --header "PRIVATE-TOKEN: $personal_access_token" "https://gitlab.${domain}/api/v4/projects/1" --insecure)
echo "Project response: "$project_response

if echo "$project_response" | grep -q "HTTP/2 404"; then
	echo "Project not found, create."
	# Create GitLab Project
	curl --request POST --header "PRIVATE-TOKEN: $personal_access_token" \
	--header "Content-Type: application/json" --data '{"name": "${example_project_name}", "description": "GeekTime Cloud Native DevOps Camp", "path": "${example_project_name}", "initialize_with_readme": "false", "only_allow_merge_if_pipeline_succeeds": "true"}' \
	--url "https://gitlab.${domain}/api/v4/projects/" --insecure
	
	git clone https://github.com/lyzhang1999/${example_project_name}.git
	cd ${example_project_name}
	git remote set-url origin https://root:$personal_access_token@gitlab.${domain}/root/${example_project_name}.git
	git add .
	git commit -a -m 'Initial'
	# push main and dev branch
	GIT_SSL_NO_VERIFY=true git push -uf origin main && GIT_SSL_NO_VERIFY=true git push -uf origin main:dev
	# Delete branch protection
	curl --insecure --request DELETE --header "PRIVATE-TOKEN: $personal_access_token" "https://gitlab.${domain}/api/v4/projects/1/protected_branches/main"
fi

############################ create camp-go-exmaple-helm project and push source code from github ################################
project_response=$(curl -s -I --request GET --header "PRIVATE-TOKEN: $personal_access_token" "https://gitlab.${domain}/api/v4/projects/2" --insecure)
echo "Project response: "$project_response

if echo "$project_response" | grep -q "HTTP/2 404"; then
	echo "Project not found, create."
	# Create GitLab Project
	curl --request POST --header "PRIVATE-TOKEN: $personal_access_token" \
	--header "Content-Type: application/json" --data '{"name": "${example_project_name}-helm", "description": "GeekTime Cloud Native DevOps Camp", "path": "${example_project_name}-helm", "initialize_with_readme": "false"}' \
	--url "https://gitlab.${domain}/api/v4/projects/" --insecure
	
	git clone https://github.com/lyzhang1999/${example_project_name}-helm.git
	cd ${example_project_name}-helm

	yq -i '.assets/image.repository = "harbor.${domain}/${harbor_registry}/${example_project_name}"' charts/env/dev/values.yaml
	yq -i '.assets/image.repository = "harbor.${domain}/${harbor_registry}/${example_project_name}"' charts/env/main/values.yaml

	# generate regcred.yaml in charts/templates/assets/image-pull-secret.yaml for pull assets/image from harbor
	kubectl create secret docker-registry regcred --docker-server=harbor.${domain} --docker-username=admin --docker-password=${harbor_password} -o yaml --dry-run=client > charts/templates/assets/image-pull-secret.yaml

	git remote set-url origin https://root:$personal_access_token@gitlab.${domain}/root/${example_project_name}-helm.git
	git add .
	git commit -a -m 'Initial'
	GIT_SSL_NO_VERIFY=true git push -uf origin main
	# Delete branch protection
	curl --insecure --request DELETE --header "PRIVATE-TOKEN: $personal_access_token" "https://gitlab.${domain}/api/v4/projects/2/protected_branches/main"
fi
```
我在实际使用时遇到一些问题，因为我的初始环境没有安装`git`工具，在git clone时会出错导致gitlab出现两个空项目，在添加`apt install -y git`后解决，第二个问题是`git add`和`git commit`没有执行成功，是因为没有设置git的username和email，添加`git config --global user.name <name>`和`git config --global user.email <email>`后解决
#### 自动构建
提交代码之后会触发jenkins自动构建流水线

### jenkins
本套CI流水线中包含代码测试、sonar-scanner代码扫描、kaniko构建镜像并导出成tarball、grype镜像安全扫描、crane推送镜像、cosign镜像签名
![Alt text](assets/image-4.png)
![Alt text](assets/image-3.png)

### Harbor
Harbor是一个用于存储和分发Docker容器内容的企业级Registry服务器。harbor中可以设置只信任通过cosign签名的镜像，这样可以确保只有通过流水线的镜像才会被自动部署
![Alt text](assets/image-26.png)

### ArgoCD
>Argo CD是一款开源的GitOps工具，可以实现Git仓库中的应用部署。
- 当镜像更新完成后，argoCD会从相应的helm仓库中拉取描述文件进行部署
- 可以通过不同的Namespaces区分不同的环境

argocd-assets/image-updater会更新helm仓库中的assets/image.tag信息
![Alt text](assets/image-21.png)
当harbor上dev、main镜像签名通过后，ArgoCD可以看到两个应用
![Alt text](assets/image-5.png)
ArgoCD开始将两个应用部署到k8s上
![Alt text](assets/image-6.png)
部署完成后两个应用可以正常访问了
![Alt text](assets/image-7.png)

## 提交代码更新
1. 首先提交dev分支代码到gitlab，jenkins流水线通过后ArgoCD部署dev环境
2. 通过merge_request将dev代码合并到main分支，触发流水线
3. jenkins流水线通过后ArgoCD部署main环境

### 更新dev环境
- 先在dev分支下修改代码
![Alt text](assets/image-8.png)
![Alt text](assets/image-9.png)

- 然后会触发新的CI流水线
![Alt text](assets/image-10.png)

- 通过后ArgoCD开始更新dev应用
![Alt text](assets/image-11.png)

- 可以看到变化了
![Alt text](assets/image-12.png)

### 更新main环境
- 再次提交代码到main分支,先创建一个merge request
![Alt text](assets/image-13.png)
- 要取消勾选删除分支的选项
![Alt text](assets/image-14.png)
- 提交mr后会再次触发CI流水线
![Alt text](assets/image-15.png)
- 可以选则通过pipeline后自动提交合并，在合并后再次触发main分支的CI流水线
![Alt text](assets/image-16.png)
- main分支也完成了更新
![Alt text](assets/image-17.png)
![Alt text](assets/image-18.png)


## 创建新的测试环境
1. 创建一个新的testing分支
2. 将helm仓库中env目录里的main复制一份命名为testing
3. ArgoCD部署testing环境
4. 可以看到testing Namespace下有相应的deployment

先创建一个新的testing分支
![Alt text](assets/image-19.png)

jenkins底下已经出testing分支
![Alt text](assets/image-20.png)

在helm仓库中env目录里的main复制为testing，并修改颜色以做区分
![Alt text](assets/image-22.png)

ArgoCD出现了第三个应用
![Alt text](assets/image-23.png)

可以看到应用创建成功了
![Alt text](assets/image-24.png)
![Alt text](assets/image-25.png)

## 敏捷开发中的迭代、用户故事、任务有哪些区别

### 迭代

- 迭代是指在一个时间段内，完成一组功能、任务或用户故事的过程。通常持续1到4周。
- 迭代的主要目标是将软件系统分成小块，并在每个迭代结束时交付可用的增量版本。
- 迭代通常以一个可工作的软件版本作为结果，可以进行测试和验证。

### 用户故事

- 用户故事是用户需求的一种表达方式，通常以简洁的形式来描述一个功能或特性。
- 用户故事通常包括角色（谁需要这个功能）、功能（做什么）和价值（为什么需要）。
- 用户故事有助于团队更好地理解用户需求，并将其分解成小任务，以便在迭代中实现

### 任务

- 任务是用户故事的更小的、可执行的部分。
- 任务通常以更具体的行动步骤或工作单元的形式存在，以实现用户故事所描述的功能。
- 任务可以分配给团队成员，以确保每个任务都有人负责。
- 任务的完成状态可以用于跟踪迭代进度，并作为验收标准的一部分。

### 总结
总结来说，迭代是时间框架，用于将开发工作划分为小块，用户故事是帮助团队对焦用户需求的手段，而任务是用户故事的具体执行步骤和验收标准。任务有助于团队管理和跟踪工作，确保用户故事按计划完成。

## DevOps 工作流中如何避免“配置漂移”(基础设施和应用配置)
### 配置漂移
> 配置漂移是指在应用程序或基础设施的配置发生未经授权或不受监控的变化的现象，这可能会导致问题和不一致性。

以下是一些可能导致配置漂移的行为：
1. **手动更改配置**：手动更改服务器、应用程序或基础设施配置，而不经过适当的审查和记录，可能会导致配置漂移
2. **无计划的紧急修复**：在紧急情况下，为了解决问题，可能会进行临时修复，而不考虑如何影响配置的一致性
3. **不受控制的环境变化**：如果环境中的组件（如库、操作系统、依赖项）发生不受控制的更改，可能会导致配置漂移，因为应用程序配置可能需要相应调整
4. **缺乏自动化**：在没有自动化流程的情况下进行部署和配置更改，可能会导致人为错误和配置漂移
5. **缺乏文档和记录**：没有详细的文档和记录来跟踪配置更改，无法及时发现和解决配置漂移问题
6. **缺乏权限控制**：如果没有适当的权限控制，任何人都可以更改配置，这可能导致未经授权的配置更改
7. **忽视持续监控**：如果不定期监控配置，就无法及时发现配置漂移，从而使问题得以累积
8. **不合规的更新和补丁管理**：未经适当测试和验证的更新或补丁可能会导致配置漂移和不稳定性
9. **不恰当的版本控制**：如果没有适当管理和版本控制配置文件，就可能导致混乱和配置漂移
10. **人为错误**：操作失误、误删除或误操作配置文件等人为错误可能会导致配置漂移
11. **缺乏自动化测试**：如果没有自动化测试来验证配置更改的影响，就很难确保配置的一致性。

### 避免配置漂移
为了避免配置漂移，团队应采取一系列最佳实践，如自动化、版本控制、权限控制、文档化、自动化测试和持续监控，以确保配置的一致性和稳定性。这些措施可以减少配置漂移的风险，提高系统的可靠性。
1. **使用GitOps**：通过 GitOps 方法，您可以使用 Git 来存储所有基础设置和应用程序配置。然后，您可以在 CI/CD 流水线中自动部署这些配置。
2. **基础设施即代码（Infrastructure as Code，IaC）**：使用IaC工具（如Terraform）来定义和管理基础设施和配置。这样可以确保基础设施和配置的一致性，并且可以轻松地重现环境。
3. **持续监控和审查**：定期检查和监控生产环境的配置，以确保没有意外的漂移。使用工具来自动检测配置变化并生成报告。
4. **权限和访问控制**：限制对生产环境的访问权限，并确保只有授权的人员才能修改配置。实施权限控制，记录每个人的操作，以便跟踪配置更改。
5. **自动化测试**：在部署之前执行自动化测试，以确保配置变更不会破坏应用程序的功能。包括单元测试、集成测试和回归测试等。
6. **文档化**：详细记录配置信息，包括版本、依赖关系和更改历史。这可以帮助团队理解和管理配置。
7. **灾备和备份**：定期创建系统备份和快照，以防止配置漂移导致数据丢失或系统不稳定。确保可以快速还原到稳定状态。
