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
4. 인스턴스 타입은 t3.micro 또는 t3.small로 지정 및 Create Environment 선택 후 Cloud9 환경 빌드
5. Open 하이퍼링크 클릭
![Screenshot 2022-12-27 at 12 46 52](https://user-images.githubusercontent.com/92728844/209608225-eeacbf2c-919c-4a4c-9024-dacd1b00e90c.jpg)
6. 아래와 같은 Cloud9 환경이 열리는 것을 확인
![Screenshot 2022-12-27 at 12 45 50](https://user-images.githubusercontent.com/92728844/209607888-fb783a07-c8e2-479b-ae00-9eeb6fb3a543.png)
7. Cloud9에 추가 프로그램 설치를 위해 디스크 용량 증설
```bash
pip3 install --user --upgrade boto3
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
python -c "import boto3
import os
from botocore.exceptions import ClientError
ec2 = boto3.client('ec2')
volume_info = ec2.describe_volumes
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
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"<br />
unzip awscliv2.zip<br />
sudo ./aws/install<br />
```
jq, envsubst, bash-completion 설치
```bash
sudo yum -y install jq gettext bash-completion moreutils
```
yq 설치
```bash
echo 'yq() {<br />
  &ensp;docker run --rm -i -v "${PWD}":/workdir mikefarah/yq "$@"<br />
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
echo 'export LBC_VERSION="v2.4.1"' >>  ~/.bash_profile <br />
echo 'export LBC_CHART_VERSION="1.4.1"' >>  ~/.bash_profile<br />
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


# 워크스페이스에 대한 IAM 설정값 경
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
aws sts get-caller-identity --query Arn | grep eksworkshop-admin -q && echo "IAM role valid" || echo "IAM role NOT valid"
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
  region: ap-northeast-2
  version: "1.21"

availabilityZones: ["ap-northeast-2a,ap-northeast-2b,ap-northeast-2c"]

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








