<H1># Monolith-to-Microservices(모놀리스 환경에서 MSA환경으로의 전환)</H1>
Migrating NCP Monolith Application to AWS EKS(NCP에서 사용하던 Monolith 애플리케이션을 AWS EKS로 마이그레이션)

Underlying Infrastructure(AWS에서 사용 할 인프라)
---------------------------------------------
![monolith to msa](https://user-images.githubusercontent.com/92728844/209612798-5c9ec304-6bea-4d74-96ca-b8e508afd5e7.jpg)
---------------------------------------------
<H3>EKS클러스터 관리에 용이한 워크스페이스 생성</H3>
1. AWS Console로 접근 후 Cloud9 서비스 선택
2. Create Environment 선택
3. Cloud9 환경의 이름 지정(인스턴스명은 자유)
4. 인스턴스 타입은 t3.micro 또는 t3.small로 지정 및 Create Environment 선택 후 Cloud9 환경 빌드
5. Open 하이퍼링크 클릭
---------------------------------------------
![Screenshot 2022-12-27 at 12 46 52](https://user-images.githubusercontent.com/92728844/209608225-eeacbf2c-919c-4a4c-9024-dacd1b00e90c.jpg)
---------------------------------------------
6. 아래와 같은 환경이 열리는 것을 확인
---------------------------------------------
![Screenshot 2022-12-27 at 12 45 50](https://user-images.githubusercontent.com/92728844/209607888-fb783a07-c8e2-479b-ae00-9eeb6fb3a543.png)
---------------------------------------------

7. Cloud9에 추가 프로그램 설치를 위해 디스크 용량 증설

pip3 install --user --upgrade boto3 <br />
export instance_id=$(curl -s http://169.254.169.254/latest/meta-data/instance-id) <br />
python -c "import boto3 <br />
import os <br />
from botocore.exceptions import ClientError <br />
ec2 = boto3.client('ec2') <br />
volume_info = ec2.describe_volumes( <br />
    Filters=[ <br />
        { <br />
            'Name': 'attachment.instance-id', <br />
            'Values': [ <br />
                os.getenv('instance_id') <br />
            ] <br />
        } <br />
    ] <br />
) <br />
volume_id = volume_info['Volumes'][0]['VolumeId'] <br />
try: <br />
    resize = ec2.modify_volume( <br />
            VolumeId=volume_id, <br />
            Size=30 <br />
    ) <br />
    print(resize) <br />
except ClientError as e: <br />
    if e.response['Error']['Code'] == 'InvalidParameterValue': <br />
        print('ERROR MESSAGE: {}'.format(e))" <br />
if [ $? -eq 0 ]; then <br />
    sudo reboot <br />
fi <br />

