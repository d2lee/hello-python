# 핸즈온랩 가이드
본 자료는 EKS를 위한 CI/CD 환경을 구축하는 실습 자료입니다.

## 1. 시작하기에 앞서
본 핸즈온은 https://eksworkshop.com에 따라 EKS cluster가 생성되어 있어야 합니다.아직 EKS cluster가 생성 전이라면 1)https://www.eksworkshop.com/020_prerequisites/, 2)https://www.eksworkshop.com/030_eksctl/ 단계를 완료하기 바랍니다.

## 2. Cloud9 터미널 띄우기
Cloud9 서비스은 클라우드 기반의 개발환경으로 본 실습이 이 서비스를 이용합니다. 앞서 eksworkshop 사이트에 있는 실습 1,2를 완료했으면 `eksworkshop`이름으로 개발환경이 생성됩니다. 이 개발 환경으로 이동합니다. 이동 방법은 AWS console에 로그인한 다음에 화면 상단에 있는 검색 창에서 해당 서비스 (여기서는 cloud9)명을 입력하면 쉽게 찾아갈 수 있습니다. 

## 3. 핸즈온랩 자료 받기
Cloud9 터미널에서 아래 명령어를 통해 본 실습 자료를 받습니다.

```
git clone https://github.com/d2lee/hello-python.git
```

## 4. EKS cluster에 접근하기 위한 보안 설정
### 4.1 IAM role 생성 
본 실습에서는 CI/CD 환경을 구축하기 위해서 AWS CodeCommit, CodeBuild, CodePipeline 서비스를 이용합니다. 특히 CodeBuild 서비스는 소스코드가 저장되어 있는 CodeCommit에 새로운 소스가 커밋되면 그 소스를 다운로드 받아서 도커 이미지를 만들고 이 이미지를 컨테이너 이미지 저상소엔 ECR에 저장하고, 실제 EKS에 해당 컨테이너 이미지를 배포하게 됩니다. 이를 위해서 CodeCommit이 EKS에 접근할 수 있도록 IAM role를 만드는 과정입니다. Cloud9 터미널에서 다음 명령어를 복사해서 실행합니다. 
```
cd ~/environment

TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

echo '{ "Version": "2012-10-17", "Statement": [ { "Effect": "Allow", "Action": "eks:Describe*", "Resource": "*" } ] }' > /tmp/iam-role-policy

aws iam create-role --role-name EksWorkshopCodeBuildKubectlRole --assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe --policy-document file:///tmp/iam-role-policy
```
### 4.2 EKS 클러스터에 보안 설정

이전 단계에서는 IAM role만 만들었고 이를 실제 aws-auth ConfigMap을 통해서 EKS에 적용하기 해야 합니다. 이 매핑을 위해서 아래 명령어를 복사하여 Cloud9 터미널에서 실행합니다.

```
ROLE="    - rolearn: arn:aws:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml

kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```

## 5. CI/CD 환경 생성
CI/CD 환경은 코드로 자동으로 생성될 수 있도록 CloudFormation 파일로 작성되어 있습니다. 핸즈온 자료에 있는 `mix-pipeline.yml` 파일이 CloudFormation 파일입니다. 
이 파일을 이용해서 CI/CD 환경을 자동 생성할수 있도록 다음 명령어를 복사해서 실행합니다.

```
cd ~/environment/hello-python
aws cloudformation create-stack --stack-name mix-pipeline --template-body file://./mix-pipeline.yml --capabilities CAPABILITY_IAM CAPABILITY_AUTO_EXPAND

```

위의 명령어를 실행하면 아래와 같이 새로운 cloudformation 스택이 출력됩니다. 실제 xxxx값은 account ID로 표시됩니다.
```
CAPABILITY_IAM CAPABILITY_AUTO_EXPAND
{
    "StackId": "arn:aws:cloudformation:ap-northeast-2:xxxx:stack/mix-pipeline/ff990de0-9ab9-11eb-b997-062c90f678fa"
}
```

cloudformation이 생성하고 있는 리소스 목록을 보려면 AWS console에서 cloudformation 서비스 콘솔에서 `mix-pipeline` 스택을 클릭한 뒤에 [Resource] 메뉴를 보시면 됩니다. 파이프라인이 문제없이 설치되면 마지막 resource에 mix-pipeline 생성이 표시됩니다. 참고로 AWS console에서 특정 서비스로 이동하려면 AWS console 상단에 있는 검색 창에서 해당 서비스 이름을 입력하면 쉽게 이동이 가능합니다. 

## 6. 소스 코드 커밋
AWS console에서 codecommit 화면으로 이동합니다. CodeCommit은 git의 저장소와 같으며 본 실습에서는 `mix-repo`라는 이름의 Repository를 생성해 뒀습니다.CodeCommit화면에서 `mix-repo` 를 아직은 저장된 소스나 파일이 없으므로 비워있게 됩니다. 

### 6.1. codecommit 저장소를 Cloud9에 복제
CodeCommit에 있는 Repository를 복제합니다. 아래 명령어를 실행하면 `mix-repo` 저장소를 cloud9 개발환경에 복사본이 생성됩니다.

```
cd ~/environment
git clone https://git-codecommit.ap-northeast-2.amazonaws.com/v1/repos/mix-repo
```

다운로드 받은 실습 자료를 복제된 폴더에 복사합니다.
```
cd ~/environment/mix-repo
cp -r ../hello-python/* .

```

이제 git명령어를 이용하여 commit하여 push 합니다. 
```
git add .
git commit -m "Add source"
git push

```
AWS Console에 있는 CodeCommit화면에서 `mix-repo` 저장소에 가면 소스파일들이 저장되어 있는 것을 확인할 수 있습니다.

### 6.2. CodePipeline에서 배포 확인
AWS console 창에서 CodePipeline 서비스 화면으로 이동하면 `mix-pipeline`으로 시작하는 Pipeline을 확인할 수 있습니다. 이를 클릭하면 해당 파이프라인이 현재 실행되고 있는 상태를 확인할 수 있습니다. 
위에서 소스를 push 하는 순간 CodeCommit에 새로운 파일이 저장되는 것을 감지하여 CodeBuild가 실행되게 됩니다. CodeBuild는 CodeCommit의 소스를 받아다가 docker 컨테이너로 빌드를 한 뒤에 그 결과를 컨테이너 저장소인 ECR에 저장하게 됩니다. 그 뒤에 EKS클러스터에 그 도커 파일을 배포하게 됩니다. EKS 클러스터에 배포할 떄 사용하는 yaml 파일은 핸즈온 실습에 있는 `deploy-hello-python.yml`파일이며,CodeBuild가 수행하는 이런 일련의 과정은 `buildspec.yml`파일에 기술되어 있습니다.

### 6.3. EKS에 배포된 결과 확인
EKS 클러스터에 어플리케이션이 정상적으로 실행되어 있는지 확인합니다. 
```
kubectl get svc
```
이 명령어를 실행한 결과중에 `hello-k8s`의 서비스의 external IP 란에 있는 도메인 주소를 웹 브라우저에서 오픈합니다.

### 6.4 소스 수정
어플리케이션의 소스를 수정해서 CodeCommit에 commit하면 자동으로 새로운 빌드가 만들어지는 확인합니다.

Cloud9에서 터미널에서 친숙한 에디터를 이용해서 `app.py` 파일을 오픈합니다. `hello_world()`함수에 있는 `Hello, Docker!`를 `Hello, Mix team!`으로 수정합니다. 

수정한 뒤에 아래와 명령을 통해 CodeCommit에 commit합니다.
```
cd ~/environment/mix-repo
git add .
git commit -m "Modify welcome msg"
git commit
git push
```
CodePipeline화면에서 코드 변경을 인지하여 도커 빌드가 시작되고 EKS에 배포되는 것을 확인할 수 있습니다. [정보] CodePipeline이 소스가 변환된 것을 감지하는데 1-2분 정도 걸립니다.

## 7. ARGOCD를 통해 GitOps 핸즈온

