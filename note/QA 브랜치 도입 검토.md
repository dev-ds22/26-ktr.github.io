## 검토 이유

```text
이하 배포 구조에서 jenkins 배포 job이 오류로 종료되는 것을 방지하기 위햐여 별도의 qa 브랜치를 생성하여
feature 브랜치에서 작업 후 qa 브랜치 에 MR, qa 브랜치에서 sonarQube 검증 후 문제가 없을 경우에만
qa 브랜치의 소스를 dev 에 반영하여 dev 배포 job을 실행 시키려고 함.
 gilab과 jenkins에서 새로 생성 할 것과 수정할 부분을 절차대로 상세히 설명 
 

1. feature 브랜치에서 개발자 작업 후 dev 브랜치에 MR 요청.
2. dev 브랜치 merge  시 gitlab 의 webhook 에 의해 jenkins pipeline에 의해 jenkins dev 배포 job 실행
3. jenkins dev 배포 job 에서 SonarQube Stage 에서 수정 소스 검증
4. SonarQube Stage 에서 검증 오류 시 jenkins 배포 job 오류
```

## 결론

현재 목적이 **“SonarQube 검증 실패 때문에 dev 배포 Job이 실패로 끝나는 상황을 줄이는 것”**이라면 구조는 아래처럼 바꾸는 것이 적절합니다.

```text
feature/* 작업
→ feature/* → qa MR
→ qa 병합 후 Jenkins QA 검증 Job 실행
→ SonarQube Quality Gate 통과
→ qa → dev MR
→ dev 병합
→ Jenkins dev 배포 Job 실행
```

핵심은 **SonarQube Quality Gate 실패를 dev 배포 Job이 아니라 qa 검증 Job에서 발생시키는 것**입니다. Jenkins의 `waitForQualityGate abortPipeline: true`는 Quality Gate가 통과하지 않으면 Pipeline을 중단할 수 있으므로, 이 단계는 dev 배포 Job이 아니라 qa 검증 Job에 두는 것이 맞습니다. SonarQube Jenkins 공식 문서도 `withSonarQubeEnv` 이후 `waitForQualityGate`를 사용하며, `abortPipeline: true`이면 Quality Gate 실패 시 Pipeline이 중단된다고 설명합니다. ([Sonar Documentation](https://docs.sonarsource.com/sonarqube-server/analyzing-source-code/ci-integration/jenkins-integration/pipeline-pause "Setting up a pipeline pause | SonarQube Server | Sonar Documentation"))

## 1. 변경 전/후 구조

|구분|현재 구조|변경 후 구조|
|---|---|---|
|개발 브랜치|`feature/*`|`feature/*`|
|1차 병합 대상|`dev`|`qa`|
|SonarQube 실행 위치|`dev` 배포 Job 내부|`qa` 검증 Job 내부|
|Sonar 실패 영향|`dev` 배포 Job 실패|`qa` 검증 Job 실패|
|dev 반영 조건|dev MR 승인 후 바로 반영|qa 검증 통과 후 `qa → dev` MR|
|dev Job 역할|검증 + 배포|배포 중심|

## 2. GitLab에서 새로 생성할 것

### 2-1. `qa` 브랜치 생성

`qa` 브랜치는 현재 배포 기준인 `dev`에서 생성하는 것이 안전합니다.

```bash
git fetch origin
git checkout dev
git pull origin dev
git checkout -b qa
git push -u origin qa
```

GitLab UI로도 생성 가능합니다.

```text
GitLab Project
→ Code
→ Branches
→ New branch
→ Branch name: qa
→ Create from: dev
```

GitLab 공식 문서 기준으로 새 브랜치는 기존 브랜치, 태그, 커밋을 기준으로 생성할 수 있습니다. ([GitLab 문서](https://docs.gitlab.com/user/project/repository/branches/protected/ "Protected branches | GitLab Docs"))

### 2-2. `qa` 브랜치 보호 설정

```text
GitLab Project
→ Settings
→ Repository
→ Branch rules
→ Add branch rule
→ Branch name: qa
```

권장 설정은 다음과 같습니다.

|항목|권장값|
|---|---|
|Branch|`qa`|
|Allowed to push|일반 개발자 직접 push 금지|
|Allowed to merge|Maintainer 또는 QA/리드 개발자|
|Force push|비활성화|
|Code Owner approval|가능하면 활성화|
|GitLab protected branch에서는 브랜치별 push/merge 권한, force push 허용 여부, Code Owner 승인 요구를 설정할 수 있습니다. Code Owner approval을 켜면 해당 규칙에 걸린 MR은 Code Owner 승인이 필요하고, 직접 push도 제한될 수 있습니다. ([GitLab 문서](https://docs.gitlab.com/user/project/repository/branches/protected/ "Protected branches \| GitLab Docs"))||

### 2-3. `dev` 브랜치 보호 설정 강화

`dev`는 실제 배포 Job을 트리거하는 브랜치이므로 더 강하게 보호해야 합니다.

```text
GitLab Project
→ Settings
→ Repository
→ Branch rules
→ dev
```

권장 설정은 다음과 같습니다.

|항목|권장값|
|---|---|
|Branch|`dev`|
|Allowed to push|No one 또는 Maintainer만|
|Allowed to merge|Maintainer 또는 배포 승인자|
|Force push|비활성화|
|feature → dev 직접 MR|프로세스상 금지|
|qa → dev MR|허용|
|주의할 점은 GitLab 기본 기능만으로 “source branch가 반드시 qa일 때만 dev merge 허용”을 완벽히 강제하기는 어렵습니다. 실무에서는 보통 `dev` 브랜치 merge 권한을 Maintainer에게만 주고, 승인 규칙/Code Owner/운영 규칙으로 `feature → dev` 직접 MR을 차단합니다.||

## 3. GitLab에서 수정할 것

### 3-1. MR 흐름 변경

기존:

```text
feature/* → dev MR
```

변경:

```text
feature/* → qa MR
qa → dev MR
```

개발자 작업 흐름은 다음과 같이 바뀝니다.

```bash
git checkout dev
git pull origin dev
git checkout -b feature/aaa
# 개발 작업
git add .
git commit -m "작업 내용"
git push -u origin feature/aaa
```

이후 GitLab에서 MR 생성:

```text
Source branch: feature/aaa
Target branch: qa
```

QA 검증 통과 후:

```text
Source branch: qa
Target branch: dev
```

### 3-2. Merge checks 설정

가능하면 GitLab의 `Pipelines must succeed`를 활성화합니다.

```text
GitLab Project
→ Settings
→ Merge requests
→ Merge checks
→ Pipelines must succeed 체크
→ Save
```

GitLab 공식 문서상 `Pipelines must succeed`를 켜면 MR은 성공한 Pipeline이 있어야 merge할 수 있으며, GitLab CI/CD뿐 아니라 외부 CI Provider의 Pipeline도 이 설정과 함께 사용할 수 있습니다. 단, MR에 성공 Pipeline이 아예 없으면 merge가 막힐 수 있으므로 Jenkins가 GitLab에 빌드 상태를 정상적으로 보고해야 합니다. ([GitLab 문서](https://docs.gitlab.com/user/project/merge_requests/auto_merge/ "Auto-merge | GitLab Docs"))

### 3-3. Webhook 설정 분리 또는 필터링

기존에는 `dev` merge 시 webhook이 Jenkins dev 배포 Job을 실행하고 있습니다. 변경 후에는 최소한 아래 2개 흐름이 필요합니다.

|Webhook/Trigger|대상 Jenkins Job|이벤트|목적|
|---|---|---|---|
|QA 검증 Trigger|`PROJECT_QA_SONAR`|`qa` push 또는 MR event|SonarQube 검증|
|DEV 배포 Trigger|`PROJECT_DEV_DEPLOY`|`dev` push|dev 배포|
|GitLab Webhook은 push, MR 생성 등 여러 이벤트를 외부 시스템에 전달할 수 있습니다. MR event는 MR 생성, 업데이트, 승인, merge, commit 추가 등 다양한 경우에 발생하므로, Jenkins 쪽에서 target branch 또는 push branch를 반드시 필터링해야 합니다. ([GitLab 문서](https://docs.gitlab.com/user/project/integrations/webhooks/ "Webhooks \| GitLab Docs"))||||
|권장:||||

```text
GitLab Project
→ Settings
→ Webhooks
→ Jenkins Webhook URL 등록 또는 기존 URL 유지
→ Push events 활성화
→ Merge request events는 MR 검증 Job을 쓸 때만 활성화
→ Secret Token 설정
```

## 4. Jenkins에서 새로 생성할 것

## 4-1. QA 검증 전용 Job 생성

새 Jenkins Job 예:

```text
PROJECT_QA_SONAR
```

권장 방식:

```text
기존 dev 배포 Job 복사
→ Job 이름을 PROJECT_QA_SONAR로 변경
→ deploy 관련 Stage 제거
→ branch를 qa로 변경
→ SonarQube Stage와 Quality Gate Stage 유지
```

이 Job은 **배포하지 않고 검증만 수행**해야 합니다.

### QA Job 구성

|Stage|필요 여부|설명|
|---|--:|---|
|Checkout|필수|`qa` 브랜치 checkout|
|Build/Compile|필수|Maven compile/package/verify|
|Unit Test|권장|테스트가 있으면 포함|
|SonarQube Analysis|필수|정적 분석 수행|
|Quality Gate|필수|실패 시 QA Job만 실패|
|Deploy|제거|qa 검증 Job에서는 배포 금지|

### Jenkins Pipeline 예시

아래는 `qa` 브랜치 merge 후 push event로 실행되는 구조입니다.

```groovy
pipeline {
    agent any
    options {
        disableConcurrentBuilds()
    }
    environment {
        BRANCH_NAME_FIXED = 'qa'
        SONAR_PROJECT_KEY = 'bk-bo-qa'
    }
    stages {
        stage('Checkout QA') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/qa']],
                    userRemoteConfigs: [[
                        url: "${GIT_URL}",
                        credentialsId: "${GIT_CREDENTIALS_ID}"
                    ]]
                ])
            }
        }
        stage('Build and SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh """
                        ${MAVEN_HOME}/bin/mvn clean verify sonar:sonar \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY}
                    """
                }
            }
        }
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
    post {
        success {
            echo 'QA SonarQube 검증 성공: qa → dev MR 진행 가능'
        }
        failure {
            echo 'QA SonarQube 검증 실패: dev 반영 금지'
        }
    }
}
```

SonarQube `waitForQualityGate`를 쓰려면 Jenkins Pipeline에 `withSonarQubeEnv`가 먼저 있어야 하고, SonarQube Server에서 Jenkins의 `/sonarqube-webhook/` URL로 webhook을 설정해야 합니다. Jenkins 공식 문서도 trailing slash가 포함된 `<Jenkins instance>/sonarqube-webhook/` 설정을 요구합니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/sonar/ "SonarQube Scanner for Jenkins"))

## 4-2. SonarQube Webhook 확인

SonarQube에서 아래 설정을 확인합니다.

```text
SonarQube
→ Administration
→ Configuration
→ Webhooks
→ Create
→ URL: http(s)://<jenkins-url>/sonarqube-webhook/
```

주의:

```text
/sonarqube-webhook/
```

마지막 `/`가 필요합니다. 이 설정이 없으면 `waitForQualityGate`가 정상적으로 결과를 받지 못하고 timeout까지 대기할 수 있습니다. ([Jenkins](https://www.jenkins.io/doc/pipeline/steps/sonar/ "SonarQube Scanner for Jenkins"))

## 5. Jenkins에서 수정할 것

## 5-1. 기존 dev 배포 Job 수정

기존 dev 배포 Job:

```text
PROJECT_DEV_DEPLOY
```

수정 방향:

|항목|기존|변경|
|---|---|---|
|Trigger branch|`dev`|`dev` 유지|
|SonarQube Stage|있음|제거 권장|
|Quality Gate|있음|제거 권장|
|Deploy Stage|있음|유지|
|실패 기준|Sonar 실패 포함|checkout/build/deploy 실패 중심|
|즉, dev 배포 Job에서는 아래 Stage를 제거하거나 비활성화합니다.|||

```groovy
stage('SonarQube Analysis') { ... }
stage('Quality Gate') { ... }
```

변경 후 dev 배포 Job 예시:

```groovy
pipeline {
    agent any
    options {
        disableConcurrentBuilds()
    }
    stages {
        stage('Checkout DEV') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/dev']],
                    userRemoteConfigs: [[
                        url: "${GIT_URL}",
                        credentialsId: "${GIT_CREDENTIALS_ID}"
                    ]]
                ])
            }
        }
        stage('Build') {
            steps {
                sh "${MAVEN_HOME}/bin/mvn clean package -DskipTests"
            }
        }
        stage('Deploy DEV') {
            steps {
                echo '기존 dev 배포 스크립트 실행'
                // 기존 ssh/scp/JBoss deploy/restart/health check 유지
            }
        }
        stage('Health Check') {
            steps {
                echo '기존 health check 실행'
            }
        }
    }
}
```

이렇게 해야 SonarQube 품질 오류는 `PROJECT_QA_SONAR`에서만 실패하고, `PROJECT_DEV_DEPLOY`는 검증 완료된 `dev` 소스를 배포하는 역할에 집중합니다.

## 5-2. dev Job Trigger 필터 확인

Jenkins dev Job은 반드시 `dev` 브랜치 push에서만 실행되어야 합니다.  
확인 위치:

```text
Jenkins Job
→ Configure
→ Build Triggers
→ Build when a change is pushed to GitLab
→ Branch filter
```

권장:

```text
Include branches: dev
또는 Regex: ^dev$
```

Jenkins GitLab Plugin은 GitLab의 push 또는 MR 이벤트로 Jenkins build를 트리거할 수 있고, GitLab에 build status를 다시 보낼 수도 있습니다. 다만 GitLab Plugin은 GitLab Inc.나 CloudBees의 공식 지원 제품은 아니므로, 플러그인 버전과 GitLab 버전 호환성 확인이 필요합니다. ([Jenkins Plugins](https://plugins.jenkins.io/gitlab-plugin/ "GitLab | Jenkins plugin"))

## 5-3. QA Job Trigger 설정

QA Job은 `qa` 브랜치 push에서만 실행하도록 설정합니다.  
권장:

```text
Jenkins Job
→ PROJECT_QA_SONAR
→ Configure
→ Build Triggers
→ Build when a change is pushed to GitLab
→ Push Events 체크
→ Branch filter: qa 또는 ^qa$
```

이 구조에서는 개발자가 `feature/* → qa` MR을 merge하면 `qa` 브랜치에 push event가 발생하고, Jenkins QA 검증 Job이 실행됩니다.

## 6. SonarQube Edition에 따른 주의점

|SonarQube 구성|권장 방식|주의점|
|---|---|---|
|Developer Edition 이상|`sonar.branch.name=qa` 또는 PR 분석 사용|branch/PR 분석 정식 지원|
|Community 계열|별도 projectKey 사용 검토|공식 branch/PR 분석 제약 가능|
|SonarQube 공식 문서 기준으로 branch analysis는 Developer Edition부터 사용할 수 있습니다. Pull Request analysis도 Developer Edition부터 사용할 수 있고, GitLab의 Merge Request도 SonarQube 문서에서는 Pull Request와 같은 의미로 취급됩니다. ([Sonar Documentation](https://docs.sonarsource.com/sonarqube-server/10.6/analyzing-source-code/branch-analysis/setting-up-the-branch-analysis "Setting up the branch analysis \| SonarQube Server 10.6 \| Sonar Documentation"))|||
|따라서 현재 SonarQube가 Community 계열이면 아래처럼 분리하는 방식이 현실적입니다.|||

```bash
-Dsonar.projectKey=bk-bo-qa
```

Developer Edition 이상이면 다음처럼 `qa` 브랜치 분석을 명시할 수 있습니다.

```bash
-Dsonar.projectKey=bk-bo
-Dsonar.branch.name=qa
```

## 7. 운영 절차

### 개발자

```text
1. dev 최신화
2. feature/* 브랜치 생성
3. 개발 및 commit
4. feature/* push
5. GitLab에서 feature/* → qa MR 생성
```

### 리뷰어 또는 QA 담당자

```text
1. feature/* → qa MR 코드 리뷰
2. MR 승인
3. qa 브랜치에 merge
4. Jenkins PROJECT_QA_SONAR 실행 확인
5. SonarQube Quality Gate 성공 확인
6. qa → dev MR 생성 또는 승인
```

### 배포 담당자

```text
1. qa → dev MR 최종 확인
2. dev에 merge
3. Jenkins PROJECT_DEV_DEPLOY 자동 실행 확인
4. dev 배포 결과 및 health check 확인
```

## 8. 최종 체크리스트

### GitLab

-  `qa` 브랜치 생성
    
-  `qa` protected branch 설정
    
-  `dev` protected branch 설정 강화
    
-  `feature/* → qa` MR 프로세스로 변경
    
-  `qa → dev` MR 프로세스 추가
    
-  `feature/* → dev` 직접 MR 금지 운영 규칙 수립
    
-  가능하면 `Pipelines must succeed` 활성화
    
-  Jenkins build status가 GitLab에 보고되는지 확인
    

### Jenkins

-  `PROJECT_QA_SONAR` Job 신규 생성
    
-  QA Job은 `qa` 브랜치만 trigger
    
-  QA Job에 SonarQube Analysis 포함
    
-  QA Job에 `waitForQualityGate abortPipeline: true` 포함
    
-  QA Job에는 deploy stage 제거
    
-  기존 dev Job은 `dev` 브랜치만 trigger
    
-  기존 dev Job에서 SonarQube Quality Gate 제거
    
-  dev Job은 build/deploy/health check 중심으로 단순화
    

### SonarQube

-  Jenkins의 SonarQube Server 설정 확인
    
-  SonarQube token 확인
    
-  SonarQube webhook URL 확인: `/sonarqube-webhook/`
    
-  Quality Gate 기준 확인
    
-  Edition에 따라 branch analysis 가능 여부 확인
    

## 9. 권장 최종 구조

```text
[개발]
feature/aaa
   ↓ MR
[검증 통합]
qa
   ↓ Jenkins PROJECT_QA_SONAR
   ↓ SonarQube Quality Gate OK
   ↓ MR
[개발 배포]
dev
   ↓ Jenkins PROJECT_DEV_DEPLOY
   ↓ DEV 서버 배포
```

가장 중요한 수정 포인트는 이것입니다.

```text
dev 배포 Job에서 SonarQube 실패를 발생시키지 말고,
qa 검증 Job에서 SonarQube 실패를 발생시킨다.
```

단, 이 구조도 **dev 배포 Job의 모든 실패를 막는 것은 아닙니다.** Maven 빌드 실패, Nexus 의존성 오류, SSH 접속 실패, JBoss 재기동 실패, health check 실패는 여전히 dev 배포 Job 실패 원인이 될 수 있습니다. 이번 변경으로 줄일 수 있는 것은 주로 **SonarQube 검증 실패로 인한 dev 배포 Job 실패**입니다.