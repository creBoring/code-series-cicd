Code 시리즈를 이용한 CI/CD 구성
==================
Architecture Overview
------------------
<img src="./md-img/architecture.PNG">

시작하기 전에
------------------
아래 사항들은 AWS Seoul Region을 기준으로 작성되었으며, 아래 AWS 공식 LAB을 이용해 진행하였음을 안내드립니다.

>CodeBuild용 CodeDeploy 샘플<br>
>https://docs.aws.amazon.com/ko_kr/codebuild/latest/userguide/sample-codedeploy.html

목차
------------------

1. **CodeCommit**<br>
  1-1. CodeCommit 레포지토리 생성 및 설정<br>
  1-2. SVN 데이터 Git으로 이전<br>
2. **배포 시나리오 도출**<br>
  2-1. 신규 EC2 인스턴스에 서비스 배포
3. **CodeBuild**<br>
  3-1. buildspec.yml 파일 생성<br>
  3-2. Build 환경 구성
4. **CodeDeploy**<br>
  4-1. appspec.yml 파일 생성<br>
  4-2. 배포 방법 설정 및 대상 지정<br>
5. **CodePipeline**<br>
  5-1. 파이프라인 생성<br>
  5-2. 테스트<br>

------
# 1. CodeCommit
## 1-1. CodeCommit 레포지토리 생성 및 설정
### CodeCommit 레포지토리 생성
1. AWS Management Console 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 codecommit 을 검색하고 [CodeCommit] 을 선택합니다.
2. 좌측탭에서 **[리포지토리]** 선택 후 **[리포지토리 생성]** 클릭합니다
3. 레포지토리 이름 입력 후 **[생성]** 클릭합니다

### IAM User 생성 및 권한 부여
CodeCommit은 원격 접속이 가능한 User를 IAM User를 통해 제공하고 있습니다.<br>
CodeCommit에 접근하기 위한 방법으로는 HTTPS 방식과 SSH 방식이 있는데, 두 Credential 모두 IAM User에 등록 및 할당받을 수 있습니다.

#### IAM User 생성

>IAM 사용자 생성
>https://docs.aws.amazon.com/ko_kr/IAM/latest/UserGuide/id_users_create.html#id_users_create_console

#### IAM User에 권한 부여

IAM User는 CodeCommit Git 에서도 사용되기 때문에, IAM User에게 Git과 관련된 권한이 없으면 아무런 동작도 실행할 수 없습니다.<br>
따라서 아래 절차를 통해 CodeCommit 관련 권한을 부여하고, 유저별로 특정 권한을 Deny 해보도록 하겠습니다.

1. AWS Management Console 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 iam 을 검색하고 [IAM] 을 선택합니다
2. 좌측탭에서 **[사용자]** 선택 후 사용자 리스트에서 위에서 생성한 IAM User를 클릭합니다
3. 요약 페이지로 이동되었다면, **[권한 추가]** 버튼을 클릭합니다
4. 상단에서 **[기존 정책 직접 연결]** 을 선택한 후 검색창에서 "codecommit" 을 검색하고 [AWSCodeCommitFullAccess] 를 선택합니다.
5. **[다음: 검토]** 버튼을 누른 후 **[권한 추가]** 버튼을 누릅니다

위 작업을 통해 생성 된 IAM User에 CodeCommit 레포지토리에 commit, push 등 모든 작업을 할 수 있는 권한을 부여 했습니다.

#### IAM User에 권한 축소
위 단계에서 권한을 광범위하게 부여했기 때문에, 특정 작업에 대해서나 특정 브런치에 대해 권한을 축소시키고 싶다면, 아래 절차를 통해 Deny 권한을 만들어 IAM User에 부여해주면 됩니다.

1. AWS Management Console 좌측 상단에 있는 **[Services]** 를 선택하고 검색창에서 iam 을 검색하고 [IAM] 을 선택합니다
2. 좌측탭에서 **[정책]** 선택 후 **[정책 생성]** 버튼을 클릭합니다
3. 상단에서 **[JSON]** 탭을 선택한 후 아래와 같이 입력합니다.

```json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Deny",
            "Action": "codecommit:GitPush",
            "Resource": "arn:aws:codecommit:**(리전코드)**:**(AccountID)**:**(레포지토리 이름)**",
            "Condition": {
                "StringEqualsIfExists": {
                    "codecommit:References": [
                        "refs/heads/**(브런치명)**"
                    ]
                },
                "Null": {
                    "codecommit:References": false
                }
            }
        }
    ]
}

```

위 JSON 정책은 특정 브런치에 대해 Git Push Event 를 Deny 하는 정책입니다.
실제 값으로 채울 경우 아래와 같이 만들어집니다.

```json

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Deny",
            "Action": "codecommit:GitPush",
            "Resource": "arn:aws:codecommit:ap-northeast-2:640969500000:ldh-repo",
            "Condition": {
                "StringEqualsIfExists": {
                    "codecommit:References": [
                        "refs/heads/master"
                    ]
                },
                "Null": {
                    "codecommit:References": false
                }
            }
        }
    ]
}

```

4. JSON을 통해 정책을 모두 입력했다면, **[정책 검토]** 를 클릭합니다.
5. 정책 이름을 입력한 후 **[정책 생성]** 을 클릭합니다.
6. 정책이 만들어졌다면, **"IAM User에 권한 부여"** 단계에서 권한만 방금 만들어준 정책으로 부여해줍니다.

## 1-2. SVN 데이터 Git으로 이전
기존에 Git이 아닌 SVN을 사용하고 있던 경우, SVN에 저장되어 있던 데이터들을 CodeCommit 레포지토리로 옮겨주어야 합니다.

이 과정에서 기존 SVN에서 commit 되었던 내역들이 유지되지 않습니다. 이 점을 유의하여 기존 history 데이터를 별도로 보관하는 것을 권해드립니다.

--------------------

# 2. 배포 시나리오 도출
