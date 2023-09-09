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
![Alt text](image.png)
![Alt text](image-1.png)
![Alt text](image-2.png)

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

	yq -i '.image.repository = "harbor.${domain}/${harbor_registry}/${example_project_name}"' charts/env/dev/values.yaml
	yq -i '.image.repository = "harbor.${domain}/${harbor_registry}/${example_project_name}"' charts/env/main/values.yaml

	# generate regcred.yaml in charts/templates/image-pull-secret.yaml for pull image from harbor
	kubectl create secret docker-registry regcred --docker-server=harbor.${domain} --docker-username=admin --docker-password=${harbor_password} -o yaml --dry-run=client > charts/templates/image-pull-secret.yaml

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
![Alt text](image-4.png)
![Alt text](image-3.png)

### Harbor
Harbor是一个用于存储和分发Docker容器内容的企业级Registry服务器。harbor中可以设置只信任通过cosign签名的镜像，这样可以确保只有通过流水线的镜像才会被自动部署
![Alt text](image-26.png)

### ArgoCD
>Argo CD是一款开源的GitOps工具，可以实现Git仓库中的应用部署。
- 当镜像更新完成后，argoCD会从相应的helm仓库中拉取描述文件进行部署
- 可以通过不同的Namespaces区分不同的环境

argocd-image-updater会更新helm仓库中的image.tag信息
![Alt text](image-21.png)
当harbor上dev、main镜像签名通过后，ArgoCD可以看到两个应用
![Alt text](image-5.png)
ArgoCD开始将两个应用部署到k8s上
![Alt text](image-6.png)
部署完成后两个应用可以正常访问了
![Alt text](image-7.png)

## 提交代码更新
1. 首先提交dev分支代码到gitlab，jenkins流水线通过后ArgoCD部署dev环境
2. 通过merge_request将dev代码合并到main分支，触发流水线
3. jenkins流水线通过后ArgoCD部署main环境

先在dev分支下修改代码
![Alt text](image-8.png)
![Alt text](image-9.png)
然后会触发新的CI流水线
![Alt text](image-10.png)
通过后ArgoCD开始更新dev应用
![Alt text](image-11.png)
可以看到变化了
![Alt text](image-12.png)
再次提交代码到main分支,先创建一个merge request
![Alt text](image-13.png)
要取消勾选删除分支的选项
![Alt text](image-14.png)
提交mr后会再次触发CI流水线
![Alt text](image-15.png)
可以选则通过pipeline后自动提交合并，在合并后再次触发main分支的CI流水线
![Alt text](image-16.png)
main分支也完成了更新
![Alt text](image-17.png)
![Alt text](image-18.png)


## 创建新的测试环境
1. 创建一个新的testing分支
2. 将helm仓库中env目录里的main复制一份命名为testing
3. ArgoCD部署testing环境
4. 可以看到testing Namespace下有相应的deployment

先创建一个新的testing分支
![Alt text](image-19.png)
jenkins底下已经出testing分支
![Alt text](image-20.png)
在helm仓库中env目录里的main复制为testing，并修改颜色以做区分
![Alt text](image-22.png)
ArgoCD出现了第三个应用
![Alt text](image-23.png)
可以看到应用创建成功了
![Alt text](image-24.png)
![Alt text](image-25.png)
