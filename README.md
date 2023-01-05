<H1># Monolith-to-Microservices(모놀리스 환경에서 MSA환경으로의 전환)</H1>
Migrating NCP Monolith Application to AWS EKS(NCP에서 사용하던 Monolith 애플리케이션을 AWS EKS로 마이그레이션)

Underlying Infrastructure(AWS에서 사용 할 인프라)
---------------------------------------------
![monolith to msa](https://user-images.githubusercontent.com/92728844/209612798-5c9ec304-6bea-4d74-96ca-b8e508afd5e7.jpg)
---------------------------------------------
# EKS클러스터 관리에 용이한 워크스페이스 생성
1. AWS Console로 접근 후 Cloud9 서비스 선택
2. Create Environment 선택
3. Cloud9 환경의 이름 지정(인스턴스명은 자유)
4. Environment type은 'New EC2 instance'로 지정
5. 인스턴스 타입은 t3.micro 또는 t3.small로 지정 및 Create Environment 선택 후 Cloud9 환경 빌드
6. Platform은 Amazon Linux 2에 Timeout은 30minutes로 지정
7. Connection은 SSM이 아닌 SSH로 변경
8. VPC setting은 사전에 생성한 VPC 또는 Default VPC 사용
9. Create 클릭
10. Open 하이퍼링크 클릭
![Screenshot 2022-12-27 at 12 46 52](https://user-images.githubusercontent.com/92728844/209608225-eeacbf2c-919c-4a4c-9024-dacd1b00e90c.jpg)
11. 아래와 같은 Cloud9 환경이 열리는 것을 확인
![Screenshot 2022-12-27 at 12 45 50](https://user-images.githubusercontent.com/92728844/209607888-fb783a07-c8e2-479b-ae00-9eeb6fb3a543.png)
12. Cloud9에 추가 프로그램 설치를 위해 EBS볼륨 용량 증설
```bash
pip3 install --user --upgrade boto3
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
python -c "import boto3
import os
from botocore.exceptions import ClientError
ec2 = boto3.client('ec2')
volume_info = ec2.describe_volumes(
    Filters=[
        {
            'Name': 'attachment.instance-id',
            'Values': [
                os.getenv('instance_id')
            ]
        }
    ]
)
volume_id = volume_info['Volumes'][0]['VolumeId']
try:
    resize = ec2.modify_volume(    
            VolumeId=volume_id,    
            Size=30
    )
    print(resize)
except ClientError as e:
    if e.response['Error']['Code'] == 'InvalidParameterValue':
        print('ERROR MESSAGE: {}'.format(e))"
if [ $? -eq 0 ]; then
    sudo reboot
fi
```
# Cloud9에 쿠버네티스 툴 설치
kubectl 설치
```bash
sudo curl --silent --location -o /usr/local/bin/kubectl \
   https://s3.us-west-2.amazonaws.com/amazon-eks/1.21.5/2022-01-21/bin/linux/amd64/kubectl

sudo chmod +x /usr/local/bin/kubectl
```
awscli 업데이트
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```
jq, envsubst, bash-completion 설치
```bash
sudo yum -y install jq gettext bash-completion moreutils
```
yq 설치
```bash
echo 'yq() {
  docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"
}' | tee -a ~/.bashrc && source ~/.bashrc
```
설치한 바이너리의 경로 및 실행 가능여부 확인
```bash
for command in kubectl jq envsubst aws
  do  
   which $command &>/dev/null && echo "$command in path" || echo "$command NOT FOUND"  
  done
```

AWS 로드밸런서 버전 지정
```bash
echo 'export LBC_VERSION="v2.4.1"' >>  ~/.bash_profile
echo 'export LBC_CHART_VERSION="1.4.1"' >>  ~/.bash_profile
.  ~/.bash_profile
```

# 워크스페이스를 위한 IAM role(역할) 생성
1. Create role 선택
![Screenshot 2022-12-27 at 15 15 47](https://user-images.githubusercontent.com/92728844/209620366-b695430d-9e5f-4265-b617-aa9fc4e39070.png)
2. AWS Service -> EC2 선택
![Screenshot 2022-12-27 at 15 16 54](https://user-images.githubusercontent.com/92728844/209620508-77e28687-1fb7-451a-a823-b45bc8883125.png)
3. Add Permissions에서 AdministratorAccess 선택
![Screenshot 2022-12-27 at 15 20 03](https://user-images.githubusercontent.com/92728844/209620887-78ce1973-2705-4fd1-8e86-f6ccc2037a87.png)
4. Role 이름 지정 후 'AdministratorAccess' policy가 적용되어 있는지 확인
![Screenshot 2022-12-27 at 15 22 04](https://user-images.githubusercontent.com/92728844/209621181-2d574c39-3f90-4646-adc0-7878a2c6fdf9.jpg)

# 워크스페이스에 IAM role(역할) 추가
1. 워크스페이스 인스턴스 선택 후 Actions -> Security -> Modify IAM role
![Screenshot 2022-12-27 at 15 25 19](https://user-images.githubusercontent.com/92728844/209621517-b840bf7c-dbec-41ab-925c-b9a6671a623c.png)
2. 생성한 IAM role 선택 후 Update IAM Role 
![Screenshot 2022-12-27 at 15 27 18](https://user-images.githubusercontent.com/92728844/209621736-8e1491db-8db5-434c-ae9c-dc3bc7bb5068.png)


# 워크스페이스에 대한 IAM 설정값 변경
1. AWS가 관리하는 일시으로 부여하는 credential 삭제
```bash
aws cloud9 update-environment  --environment-id $C9_PID --managed-credentials-action DISABLE
rm -vf ${HOME}/.aws/credentials
```
2. 계정ID, 리전, 가용영역을 환경변수로 저장
```bash
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
export AZS=($(aws ec2 describe-availability-zones --query 'AvailabilityZones[].ZoneName' --output text --region $AWS_REGION))
```
3. 리전이 제대로 설정되어있는지 확인
```bash
test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || echo AWS_REGION is not set
```
4. bash_profile에 해당 변수들 저장
```bash
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
echo "export AZS=(${AZS[@]})" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```
5. IAM role(역할)이 유효한지 확인
```bash
aws sts get-caller-identity --query Arn | grep EKS-role -q && echo "IAM role valid" || echo "IAM role NOT valid"
```
```json
만약 IAM role NOT valid가 뜬다면 '워크스페이스를 위한 IAM role(역할) 생성' 섹션으로 돌아가서 해당 단계부터 다시 진행
```
# eksctl을 사용해 eks클러스터 생성을 위한 바이너리 설치
1. eksctl 바이너리 설치
```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv -v /tmp/eksctl /usr/local/bin
```
2. eksctl 비전 확인
```bash
eksctl version
```
3. eksctl bash-completion 활성화
```bash
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

# eksctl을 사용해 eks클러스터 생성
1. eks클러스터 배포를 위한 yaml파일을 생성
```yaml
cat << ZZZ > eksworkshop.yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: eksworkshop-eksctl
  region: ${AWS_REGION}
  version: "1.21"

availabilityZones: ["${AZS[0]}", "${AZS[1]}", "${AZS[2]}"]

managedNodeGroups:
- name: nodegroup
  desiredCapacity: 3
  instanceType: t3.medium
  ssh:
    enableSsm: true
ZZZ
```

2. eks클러스터 생성
```zsh
eksctl create cluster -f eksworkshop.yaml
```

# eks클러스터 테스트
1. kubernetes 노드 확인
```bash
kubectl get nodes # yaml 파일에 desiredCapacity를 3으로 명시했기 때문에 3개의 node가 있는지 확인
```
2. kubeconfig 파일 변경
```bash
aws eks update-kubeconfig --name eksworkshop-eksctl --region ${AWS_REGION}
```
3. Worker Role 이름을 변수로 저장
```bash
STACK_NAME=$(eksctl get nodegroup --cluster eksworkshop-eksctl -o json | jq -r '.[].StackName')
ROLE_NAME=$(aws cloudformation describe-stack-resources --stack-name $STACK_NAME | jq -r '.StackResources[] | select(.ResourceType=="AWS::IAM::Role") | .PhysicalResourceId')
echo "export ROLE_NAME=${ROLE_NAME}" | tee -a ~/.bash_profile
```

# 쿠버네티스 클러스터의 작업공간 분리
1. kubernetes frontend workload로 사용 할 namespace 생성
```bash
kubectl create ns frontend
```
2. kubernetes backend workload로 사용 할 namespace 생성
```bash
kubectl create ns backend
```
3. kubernetes frontend, backend ns 생성됬는지 조회
```bash
kubectl get ns --sort-by=.metadata.creationTimestamp | tac
```
![Screenshot 2022-12-28 at 9 54 26](https://user-images.githubusercontent.com/92728844/209741885-099dfd50-d13f-4a21-9568-8a825d5541e3.jpg)

4. Cloud9 환경에서 frontend 및 backend 디렉터리 생성
```bash
mkdir frontend
mkdir backend
```

# namespace관련 추가적인 툴 설치
1. krew 설치(cloud9 환경이 아닐 시 'git --version'을 통해 git이 설치되어있는지 확인)
```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
```
2. krew 디렉터리를 환경변수에 저장
```bash
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"
```
3. krew의 ns기능 설치
```bash
kubectl krew install ns
```
![Screenshot 2022-12-28 at 10 11 52](https://user-images.githubusercontent.com/92728844/209742671-b829191f-3d3d-4d88-bc2a-507d912ddc89.png)

4. ns사용방법(ns 앞에 작업하고 싶은 namespace명을 명시하고 ns 앞에 -를 붙여서 이전 namespace로 이동)
```bash
kubectl ns frontend
kubectl ns -
```
![Screenshot 2022-12-28 at 10 28 52](https://user-images.githubusercontent.com/92728844/209743225-bd9b7b30-c881-4cc4-9bff-030929aa5d94.png)
# Ingress를 통한 AWS Load balancer controller 만들기
1. AWS Load Balancer 컨트롤러를 배포하기 전, 클러스터에 대한 IAM OIDC(OpenID Connect) identity Provider 생성
```bash
eksctl utils associate-iam-oidc-provider \
    --region ${AWS_REGION} \
    --cluster eksworkshop-eksctl \
    --approve
```
2. 클러스터의 OIDC provider URL을 해당 명령어로 확인
```bash
aws eks describe-cluster --name eksworkshop-eksctl --query "cluster.identity.oidc.issuer" --output text
```
3. 명령어 결과 나오는 값은 아래와 같은 형식임을 확인
![Screenshot 2023-01-03 at 17 06 51](https://user-images.githubusercontent.com/92728844/210319706-7f4c3c1a-2cc6-41e8-b67f-0211f8c5e14b.png)
4. 위의 결과 값에서 /id/ 뒤에 있는 값을 복사한 후, 아래의 명령어를 실행
![Screenshot 2023-01-03 at 17 11 39](https://user-images.githubusercontent.com/92728844/210320137-24f3060e-2902-463f-91ba-47a9f157a564.png)
```json
결과 값이 출력되면 IAM OIDC identity provider가 클러스터에 생성이 된 상태이며, 아무 값도 나타나지 않으면 생성 작업을 수행
```
5. AWS Load Balancer Controller에 부여할 IAM Policy를 생성하는 작업을 수행
```bash
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.4/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
```
6. AWS Load Balancer Controller를 위한 ServiceAccount를 생성
```bash
eksctl create iamserviceaccount \
    --cluster eksworkshop-eksctl \
    --namespace kube-system \
    --name aws-load-balancer-controller \
    --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
```
# 클러스터에 컨트롤러 추가하기
1. AWS Load Balancer controller를 클러스터에 추가하는 작업 수행
```bash
kubectl apply --validate=false -f https://github.com/jetstack/cert-manager/releases/download/v1.5.4/cert-manager.yaml
```
2. Load balancer controller yaml 파일을 다운로드
```bash
wget https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases/download/v2.4.4/v2_4_4_full.yaml
```
3. yaml 파일에서 클러스터의 cluster-name을 편집
```yaml
spec:
    containers:
    - args:
        - --cluster-name=eksworkshop-eksctl # 생성한 클러스터 이름을 입력
        - --ingress-class=alb
        image: amazon/aws-alb-ingress-controller:v2.4.4
```
![Screenshot 2023-01-03 at 17 24 08](https://user-images.githubusercontent.com/92728844/210321646-0cd05c2e-a45d-4bab-ad95-ba2e06ad5d8c.png)

4. yaml 파일에서 ServiceAccount yaml spec을 삭제. AWS Load Balancer Controller를 위한 ServiceAccount를 이미 생성했기 때문에 아래 내용을 삭제 후 yaml 파일 저장
```yaml
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    app.kubernetes.io/component: controller
    app.kubernetes.io/name: aws-load-balancer-controller
  name: aws-load-balancer-controller
  namespace: kube-system
```
5. AWS Load Balancer controller 파일을 배포
```bash
kubectl apply -f v2_4_4_full.yaml
```
6. 배포가 성공적으로 되고 컨트롤러가 실행되는지 아래의 명령어를 통해 확인. 결과 값이 도출되면 정상!
```bash
kubectl get deployment -n kube-system aws-load-balancer-controller
```
7. 아래의 명령어를 통해 service account가 생성됨을 확인
```bash
kubectl get sa aws-load-balancer-controller -n kube-system -o yaml
```
8. 속성 값을 파악
```bash
ALBPOD=$(kubectl get pod -n kube-system | egrep -o "aws-load-balancer[a-zA-Z0-9-]+")
kubectl describe pod -n kube-system ${ALBPOD}
```

# frontend-deployment, service 및 ingress 배포하기
1. frontend-deployment.yaml 파일 확인 및 frontend-deployment.yaml frontend 디렉터리로 이동
```yaml
cat <<ZZZ> frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: carflix-frontend
  labels:
    app: carflix-frontend
  namespace: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: carflix-frontend
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  minReadySeconds: 30
  template:
    metadata:
      labels:
        app: carflix-frontend
    spec:
      containers:
      - image: public.ecr.aws/w3v4z9i9/carflix-prod:front.v3
        imagePullPolicy: Always
        name: carflix-frontend
        ports:
        - containerPort: 8110
          protocol: TCP
ZZZ
 ```
```bash
mv frontend-deployment.yaml ./frontend/frontend-deployment.yaml
```
 2. frontend-deployment.yaml 파일을 사용해 deployment를 frontend namespace에 생성(pwd로 현재 경로 확인)
 ```bash
 kubectl apply -f ./frontend/frontend-deployment.yaml
 ```
 3. frontend namespace에 frontend deployment가 생성됬는지 확인
 ```bash
 kubectl get deploy carflix-frontend -n frontend
 kubectl get po -n frontend
 ```
 ![Screenshot 2023-01-03 at 11 08 10](https://user-images.githubusercontent.com/92728844/210291675-f214fd96-d046-4cba-89cc-b3704f679c68.png)
 
 4. frontend-service.yaml 파일 확인 및 frontend-service.yaml 파일 frontend 디렉터리로 이동
 ```yaml
cat <<ZZZ> frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: carflix-frontend-internal
  namespace: frontend
spec:
  selector:
    app: carflix-frontend
  type: ClusterIP
  ports:
   -  name: https
      protocol: TCP
      port: 443
      targetPort: 8110
ZZZ
```
```bash
mv frontend-service.yaml ./frontend/frontend-service.yaml
```
 5. frontend-service.yaml 파일을 사용해 service를 frontend namespace에 생성
 ```bash
 kubectl apply -f ./frontend/frontend-service.yaml
 ```
 6. frontend namespace에 frontend service가 생성됬는지 확인
 ```bash
 kubectl get svc carflix-frontend -n frontend
```
![Screenshot 2023-01-03 at 11 13 02](https://user-images.githubusercontent.com/92728844/210292087-24cc8600-d9ad-48cd-97fd-e3406119835f.png)

7. carflix-ingress-frontend 파일 확인 및 carflix-ingress-frontend 파일을 frontend 디렉터리로 이동
```yaml
cat <<ZZZ> carflix-ingress-frontend.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: carflix-ingress-frontend
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: carflix-ingress
  namespace: frontend
spec:
  rules:
  - host: www.mdswebservices.com
    http:
      paths:
        - pathType: Prefix
          path: /
          backend:
            serviceName: carflix-frontend-internal
            servicePort: 443
ZZZ
```
```bash
mv carflix-ingress-frontend.yaml ./frontend/carflix-ingress-frontend.yaml
```
8. carflix-ingress-frontend.yaml 파일을 사용해 ingress를 frontend namespace에 생성
```bash
kubectl create -f ./frontend/carflix-ingress-frontend.yaml
```
9. frontend namespace에 frontend ingress가 생성됬는지 확인
```bash
kubectl get ingress -n frontend
```
![Screenshot 2023-01-03 at 17 38 41](https://user-images.githubusercontent.com/92728844/210323654-f67e9de8-8b65-401e-9ba2-1831253da572.jpg)

10. 콘솔로 들어가 AWS ALB가 frontend ingress를 통해 생성됬는지 확인
![Screenshot 2023-01-03 at 17 40 43](https://user-images.githubusercontent.com/92728844/210323919-ddea592b-49cd-4f88-848d-7af21fa96d4e.jpg)
* View/edit rules 눌러서 세부규칙 확인
![Screenshot 2023-01-04 at 9 18 29](https://user-images.githubusercontent.com/92728844/210462534-e4de53c3-3a3d-41a6-adf8-33492baf333a.jpg)
* 타겟그룹 선택
![Screenshot 2023-01-04 at 9 22 49](https://user-images.githubusercontent.com/92728844/210463026-2f1c36c7-1f52-4797-833e-8b70eb68001f.jpg)
* frontend pod 3개가 정상적으로 할당되어있는 것을 확인
![Screenshot 2023-01-04 at 9 20 51](https://user-images.githubusercontent.com/92728844/210462696-42e5552b-86ff-4139-9d74-a43ae9a86325.png)
* 로드밸런서 주소로 접속 후 frontend 애플리케이션이 잘 보이는지 확인
![Screenshot 2023-01-04 at 9 26 37](https://user-images.githubusercontent.com/92728844/210463330-92c390c4-a3ee-4bee-9cd9-d3dcc2f578be.jpg)
```json
* 로드밸런서는 80번 포트로 listener가 설정되어 있지만 쿠버네티스 내부 service가 443번 포트로 listener가 설정되어 있기 때문에 통신 오류가 발생
* 해당 문제를 해결하기 위해 Load balancer의 target group 규칙을 수정(listener 80 -> 443으로 리다이렉트)
```
11. target group 수정
![Screenshot 2023-01-04 at 10 02 20](https://user-images.githubusercontent.com/92728844/210465865-0d19cc38-15d9-4a89-8e2d-558919641503.jpg)
* Edit 버튼 클릭
![Screenshot 2023-01-04 at 10 05 17(1)](https://user-images.githubusercontent.com/92728844/210466813-50dcb8f3-9719-4f5d-b8cc-7e431ea8b017.jpg)
* 기존 Default actions 'Return fixed response' 삭제(Remove)
![Screenshot 2023-01-04 at 10 16 01](https://user-images.githubusercontent.com/92728844/210467085-4a65e632-f5e6-45bc-a783-fefb68c9ddfc.jpg)
* Default actions에서 Add action -> Redirect 선택
![Screenshot 2023-01-04 at 10 19 01](https://user-images.githubusercontent.com/92728844/210467185-8b340450-47f3-4bfa-900e-c191126a99b6.jpg)
* Protocol은 HTTPS, Port는 443, 그리고 Custom host, path, query 선택 후 Host, Path, Query 값은 기본값 유지. Status code는 301 
![Screenshot 2023-01-04 at 10 20 28](https://user-images.githubusercontent.com/92728844/210467329-0e1441b0-63d8-42ac-a4e1-6effb6a102da.jpg)
```json
* EC2 -> Load Balancer -> Load Balancer 선택 -> Listener 탭 선택 -> View/Edit rules
HTTP 80: default action 규칙이 아래와 같이 변경되어 있는지 확인(파란줄 부분 확인)
```
![Screenshot 2023-01-04 at 10 22 26(1)](https://user-images.githubusercontent.com/92728844/210472664-2f50b470-820b-47b4-93cc-ae5c5f670ef7.jpg)
# 로드밸런서의 HTTPS 통신을 위해 SSL TLS 인증서 만들기
1. AWS Console에서 Certificate Manager(ACM) 검색 -> List certificates -> Request 
![Screenshot 2023-01-04 at 15 26 38](https://user-images.githubusercontent.com/92728844/210496875-30fda187-a5e4-4f1b-a70a-13ffe74ba4d8.jpg)
2. Request a public certificate -> Next 
![Screenshot 2023-01-04 at 15 45 57](https://user-images.githubusercontent.com/92728844/210499183-2740b988-cdef-466b-9103-31e07f021251.jpg)
3. Fully qualified domain name(FQDN)애 사용할 도메인 주소 입력(여러 개의 DNS주소를 사용할 경우 sub domain에 * 사용 권장) <br />
Validation Method는 DNS validation 사용
![Screenshot 2023-01-04 at 15 48 39](https://user-images.githubusercontent.com/92728844/210499502-3422453e-e01b-4b5b-b9c2-8131979e812c.jpg)
4. Key algorithm은 RSA 2048 사용 -> Request
![Screenshot 2023-01-04 at 15 50 25](https://user-images.githubusercontent.com/92728844/210499643-a20c0467-2273-4641-9dd4-cb0a7c9cf533.jpg)
5. List certificates -> Certificate ID 선택
![Screenshot 2023-01-04 at 15 51 38](https://user-images.githubusercontent.com/92728844/210499890-6217ffa6-1674-4262-b755-0aaeba1a0b15.jpg)
6. Create records in Route 53 선택
![Screenshot 2023-01-04 at 15 52 57](https://user-images.githubusercontent.com/92728844/210500018-4122a9f8-e4d6-474e-b612-c2690bc9a307.jpg)
7. Record 선택 후 Create records
![Screenshot 2023-01-04 at 15 53 59](https://user-images.githubusercontent.com/92728844/210500144-0cf7e9d9-6e9b-4868-b86e-188db052aea7.jpg)
8. AWS Console에서 Route 53 검색 -> Dashboard -> Hosted zone
![Screenshot 2023-01-04 at 15 56 46](https://user-images.githubusercontent.com/92728844/210500722-02369709-19d4-4a8e-ba0e-6d8d4bf73807.jpg)
9. 구매한 DNS 주소 선택(외부 DNS 호스팅 업체 사용 시 Route 53으로 DNS 서비스 이전)
![Screenshot 2023-01-04 at 15 59 25](https://user-images.githubusercontent.com/92728844/210500959-99793cd2-932c-439e-93e5-b7be40361564.jpg)
10. 새로운 CNAME 레코드가 생성된 것을 확인
![Screenshot 2023-01-04 at 16 01 02](https://user-images.githubusercontent.com/92728844/210501263-8856feb0-f3f6-4f30-a391-7d8a2f94b7c2.jpg)
11. 다시 ACM으로 돌아와서 Certificate의 Status가 Pending에서 Issued로 변경된 것을 확인 
![Screenshot 2023-01-04 at 16 03 35](https://user-images.githubusercontent.com/92728844/210501783-8f1b6e54-1f9f-48ae-bee9-4ff54403a910.jpg)
12. AWS Console에서 EC2 검색 -> 왼쪽 선택 메뉴에서 Load Balancing -> Load Balancers 선택 -> 생성된 Load Balancer 선택 후 Listener 탭으로 이동 -> Add Listener
![Screenshot 2023-01-04 at 16 12 04](https://user-images.githubusercontent.com/92728844/210502636-9d241184-808c-4d41-b42c-b9a0d9ff41c7.jpg)
13. Protocol은 HTTPS, Port는 443, Default actions는 Forward to 선택 후 Target group은 기존 frontend target group으로 지정
![Screenshot 2023-01-04 at 16 15 05](https://user-images.githubusercontent.com/92728844/210503279-d36294e2-e270-4600-8073-9e897af8b9cc.jpg)
14-1. 엔드유저 세션 유지를 위해 Enable group-level stickiness 체크 후 Stickiness duration은 3 hours로 지정
14-2. Secure policy는 'ELBSecurityPolicy-2016-08' 선택 후 Default SSL/TLS certificate은 'From ACM' 선택 및 발급 받은 인증서 선택
![Screenshot 2023-01-04 at 16 15 26](https://user-images.githubusercontent.com/92728844/210503300-84f16d65-1e74-4d85-9f3b-79d7b2aff073.jpg)
15. Add 선택
![Screenshot 2023-01-04 at 16 15 39](https://user-images.githubusercontent.com/92728844/210503308-84e2bab1-f024-4a5f-bd67-b32bcb8bd214.jpg)
16. HTTPS로 리다이렉션하는 규칙을 만들었지만 Security Group에 443 포트가 오픈이 안되어있어 주황색 경고가 떠있는 것을 확인
![Screenshot 2023-01-04 at 16 18 45](https://user-images.githubusercontent.com/92728844/210504689-a7e1b3d8-2c7f-4c91-9cb1-75ffec5278df.jpg)
17. 주황색 경고 아이콘 클릭 후 로드밸런서명과 일치하는 Security Group 선택
![Screenshot 2023-01-04 at 16 18 55](https://user-images.githubusercontent.com/92728844/210504707-7a7ae502-b244-4f66-ae99-842d4486d287.jpg)
18. Security Group 선택 후 Edit inbound rules 선택
![Screenshot 2023-01-04 at 16 21 56](https://user-images.githubusercontent.com/92728844/210504723-e6080057-b6ad-4d1f-bafb-938aaa8a0496.jpg)
19. Add rule -> Type은 HTTPS -> Source는 Custom으로 지정 후 돋보기 칸에 0.0.0.0/0으로 지정 -> Save rules
![Screenshot 2023-01-04 at 16 22 55](https://user-images.githubusercontent.com/92728844/210504741-aa071703-8110-4ec0-8f7e-de0bcbfe3832.jpg)
20. AWS Console에서 Route 53 검색 -> Hosted zones -> Create record
![Screenshot 2023-01-04 at 16 24 21](https://user-images.githubusercontent.com/92728844/210504754-d002e039-d096-490a-adde-4bed26d9ad53.png)
21. Record name에 www 또는 원하는 sub domain 지정 -> Record type은 A 레코드 -> Alias 활성화 -> <br />
Alias to Application and Classica Load Balancer -> ap-northeast-2 -> 알맞는 Load Balancer명 선택 -> Create records
![Screenshot 2023-01-04 at 16 26 26](https://user-images.githubusercontent.com/92728844/210504774-79af7813-e938-44ab-a717-e2f678e9a588.jpg)
22. 생성한 A 레코드가 정상적으로 등록됬는지 확인
![Screenshot 2023-01-04 at 16 29 22](https://user-images.githubusercontent.com/92728844/210505061-13f037b9-22f0-4d16-a047-4a4d7ef5de49.jpg)
23. 애플리케이션이 HTTPS로 정상적으로 동작하는지 확인
![Screenshot 2023-01-04 at 16 32 00](https://user-images.githubusercontent.com/92728844/210673442-352fc172-1982-408e-b082-9e97979f6c00.jpg)

# backend-deployment, service 및 ingress 배포하기
1. backend-deployment.yaml 파일 확인 및 backend-deployment.yaml 파일을 backend 디렉터리로 이동
```yaml
cat <<ZZZ> backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: carflix-backend
  labels:
    app: carflix-backend
  namespace: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: carflix-backend
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  minReadySeconds: 30
  template:
    metadata:
      labels:
        app: carflix-backend
    spec:
      containers:
      - image: public.ecr.aws/w3v4z9i9/carflix-prod:api.3
        name: carflix-backend
        ports:
        - containerPort: 8082
          protocol: TCP
ZZZ
 ```
```bash
mv backend-deployment.yaml ./backend/backend-deployment.yaml
```
 2. backend-deployment.yaml 파일을 사용해 deployment를 backend namespace에 생성(pwd로 현재 경로 확인)
 ```bash
 kubectl apply -f ./backend/backend-deployment.yaml
 ```
 3. backend namespace에 backend deployment가 생성됬는지 확인
 ```bash
 kubectl get deploy carflix-backend -n backend
 kubectl get po -n backend
 ```
![Screenshot 2023-01-05 at 9 34 35](https://user-images.githubusercontent.com/92728844/210675666-9b83f159-93e1-4f58-8169-6b21f0c38469.png)

 4. backend-service.yaml 파일 확인 및 backend-service.yaml 파일을 backend 디렉터리로 이동
 ```yaml
cat <<ZZZ> backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: carflix-backend-internal
  namespace: backend
spec:
  selector:
    app: carflix-backend
  type: ClusterIP
  ports:
   -  name: https
      protocol: TCP
      port: 443
      targetPort: 8082
ZZZ
```
```bash
mv backend-service.yaml ./backend/backend-service.yaml
```
 5. backend-service.yaml 파일을 사용해 service를 backend namespace에 생성
 ```bash
 kubectl apply -f ./backend/backend-service.yaml
 ```
 6. backend namespace에 backend service가 생성됬는지 확인
 ```bash
 kubectl get svc carflix-backend -n backend
```
![Screenshot 2023-01-05 at 9 39 08](https://user-images.githubusercontent.com/92728844/210676069-3050aefc-6ca0-4f48-a48f-aedbb7e83fd5.png)

7. carflix-ingress-backend 파일 확인 및 carflix-ingress-backend 파일을 backend 디렉터리로 이동
```yaml
cat <<ZZZ> carflix-ingress-backend.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: carflix-ingress-backend
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/group.name: carflix-ingress
  namespace: backend
spec:
  rules:
  - host: api.mdswebservices.com
    http:
      paths:
        - pathType: Prefix
          path: /
          backend:
            serviceName: carflix-backend-internal
            servicePort: 443
             
ZZZ
```
```bash
mv carflix-ingress-backend.yaml ./backend/carflix-ingress-backend.yaml
```
8. carflix-ingress-backend.yaml 파일을 사용해 ingress를 backend namespace에 생성
```bash
kubectl create -f ./backend/carflix-ingress-backend.yaml
```
9. backend namespace에 backend ingress가 생성됬는지 확인
```bash
kubectl get ingress -n backend
```
![Screenshot 2023-01-05 at 9 50 34](https://user-images.githubusercontent.com/92728844/210677210-29b63efc-3a60-406f-b014-20eb2c78b602.png)

10. 콘솔로 들어가 AWS ALB에 backend ingress가 target group에 추가됬는지 확인
![Screenshot 2023-01-05 at 10 17 22](https://user-images.githubusercontent.com/92728844/210680131-c7fbe003-134f-4b7f-9f8e-f3ebe0234607.jpg)
11. api.mdswebservices DNS 주소 앞으로 backend target group이 생성된 것을 확인
![Screenshot 2023-01-05 at 10 20 45](https://user-images.githubusercontent.com/92728844/210680367-40d20424-8e46-4c4f-b4f9-ce3cdac2d9ce.jpg)
12. 해당 target group을 https 규칙의 target group에 추가
![Screenshot 2023-01-05 at 10 23 13](https://user-images.githubusercontent.com/92728844/210681948-57395168-b67f-4837-b962-0d4ca3719c71.jpg)
13. https 규칙에 규칙 추가
![Screenshot 2023-01-05 at 10 38 10](https://user-images.githubusercontent.com/92728844/210682341-8ce3d0b4-4463-457f-b875-1a831d04452c.jpg)
14. frontend target group 규칙 추가
![Screenshot 2023-01-05 at 10 41 51](https://user-images.githubusercontent.com/92728844/210682607-b1769291-08ee-4bc8-93b0-e5f121ae5aa6.jpg)
15. 규칙 순서 변경 
![Screenshot 2023-01-05 at 10 43 32](https://user-images.githubusercontent.com/92728844/210682740-5b4a32ce-4fc2-4154-b9f8-c34de772f3d2.jpg)
16. backend target group 규칙 추가
![Screenshot 2023-01-05 at 10 45 30](https://user-images.githubusercontent.com/92728844/210682907-a0e56284-5870-4b4b-8a9b-8d83793ae7d6.jpg)
17. 규칙이 잘 적용됬는지 확인
![Screenshot 2023-01-05 at 10 46 40](https://user-images.githubusercontent.com/92728844/210683052-dbab9374-8328-4058-a53f-5eee26c8b353.png)
# frontend 및 backend 서비스가 통신되서 웹앱이 잘 동작하는지 확인


















