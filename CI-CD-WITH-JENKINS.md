# 젠킨스 를 이용한 지속적 배포
이 랩에서는 쿠버네티스 상에서 Jenkins를 이용한 지속적 배포 파이프라인을 어떻게 구성할 수 있는지 배운다.  
Jenkins는 공유 저장소를 두고 자신들의 코드를 계속해서 통합하고자 하는 개발자들이 사용하는 자동화 툴이다.  
이 랩에서 최종적으로 만들 솔루션은 다음 다이어그램과 같다.

![diagram](https://github.com/myungpyo/study-k8s/blob/master/k8s_jenkins_1.png)

## 이 랩에서 하게될 것
이 랩에서는 다음과 같은 태스크들을 진행하게 된다.
- 젠킨스 애플리케이션을 쿠버네티스 엔진 클러스터에 배포
- Helm 패키지 매니저를 이용한 젠킨스 애플리케이션 설치
- 젠킨스 애플리케이션 기능들 살펴보기
- 젠킨스 파이프라인 생성 및 실습

## 쿠버네티스 엔진이란 무엇인가?

## 젠킨스란 무엇인가?

## 지속적 전달(Continuous Delivery)과 지속적 배포(Continuous Deployment)란 무엇인가?

## 젠킨스 설치하기
### Kubernetes 클러스터 생성
쿠버네티스 클러스터를 생성하기 위해서 다음 명령을 수행하자.
```bash
gcloud container clusters create jenkins-cd \
--num-nodes 2 \
--machine-type n1-standard-2 \
--scopes "https://www.googleapis.com/auth/projecthosting,cloud-platform"
```
이 과정은 완료되는데까지 몇분이 소요될 수 있다. 추가로 정의한 `scopes` 를 통해서 젠킨스가 클라우드 소스 저장소와 구글 컨테이너 레지스트리에 접근을 가능하게 한다.

### 클러스터가 실행 중인지 확인
```bash
gcloud container clusters list
```

### 클러스터의 인증 정보 획득 (쿠버네티스 엔진은 새로 생성한 클러스터로 접근할 때 이 인증 정보를 사용)
```bash
gcloud container clusters get-credentials jenkins-cd
```

### 인증 정보 획득이 제대로 이루어졌는지 확인을 위해서 클러스터에 접근
```bash
kubectl cluster-info
```
[출력 결과]
```bash
Kubernetes master is running at https://35.239.18.122
GLBCDefaultBackend is running at https://35.239.18.122/api/v1/namespaces/kube-system/services/default-http-backend:http/proxy
Heapster is running at https://35.239.18.122/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://35.239.18.122/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://35.239.18.122/api/v1/namespaces/kube-system/services/https:metrics-server:/proxy
```

## Helm 설치
Helm은 쿠버네티스 애플리케이션의 설정과 배포를 용이하게 해주는 패키지 매니저로써 Helm 을 이용하여 Charts 레퍼지토리로부터 젠킨스를 설치할 것이다.
젠킨스 설치 후에는 필요한 CI/CD 파이프라인를 생성할 수 있게된다.

### Helm 바이너리 다운로드
```bash
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz
```
### 클라우드 쉘에서 파일 압축 해제
```bash
tar zxfv helm-v2.9.1-linux-amd64.tar.gz
cp linux-amd64/helm .
```
### 현재 사용자가 젠킨스에 클러스터 권한들을 추가할 수 있도록 클러스터 RBAC의 관리자로 설정
```bash
kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
```
[출력결과]
```bash
Your active configuration is: [cloudshell-7267]
clusterrolebinding.rbac.authorization.k8s.io/cluster-admin-binding created
```

### Tiller (Helm 의 서버사이드)에게 클러스터의 클러스터 관리자(cluster-admin) 권한 부여
```bash
kubectl create serviceaccount tiller --namespace kube-system
kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
```

### Helm 초기화 (Tiller 가 클러스터에 제대로 설치되었는지 확인)
```bash
./helm init --service-account=tiller
```
[출력결과]
```bash
Creating /home/google2914107_student/.helm
Creating /home/google2914107_student/.helm/repository
Creating /home/google2914107_student/.helm/repository/cache
Creating /home/google2914107_student/.helm/repository/local
Creating /home/google2914107_student/.helm/plugins
Creating /home/google2914107_student/.helm/starters
Creating /home/google2914107_student/.helm/cache/archive
Creating /home/google2914107_student/.helm/repository/repositories.yaml
Adding stable repo with URL: https://kubernetes-charts.storage.googleapis.com
Adding local repo with URL: http://127.0.0.1:8879/charts
$HELM_HOME has been configured at /home/google2914107_student/.helm.
Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
Please note: by default, Tiller is deployed with an insecure 'allow unauthenticated users' policy.
For more information on securing your installation see: https://docs.helm.sh/using_helm/#securing-your-helm-installation
Happy Helming!
```
```bash
./helm update
```
[출력결과]
```bash
Command "update" is deprecated, use 'helm repo update'
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

### Helm 이 제대로 설치되었는지 확인
```bash
./helm version
```
[출력결과]
```bash
Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
```

=======================================================================================================================

* Jenkins 설정과 설치
Cloud Source Repository 접근을 위해서는 서비스 계정 인증 정보를 사용해야하는데, 이를 위해서 GCP 특화된 플러그인을 추가 하기 위한 사용자정의 values 파일을 사용.

[contents in values file]
Master:
  InstallPlugins:
    - kubernetes:1.12.6
    - workflow-job:2.31
    - workflow-aggregator:2.5
    - credentials-binding:1.16
    - git:3.9.3
    - google-oauth-plugin:0.7
    - google-source-plugin:0.3
  Cpu: "1"
  Memory: "3500Mi"
  JavaOpts: "-Xms3500m -Xmx3500m"
  ServiceType: ClusterIP
Agent:
  Enabled: true
  resources:
    requests:
      cpu: "500m"
      memory: "256Mi"
    limits:
      cpu: "1"
      memory: "512Mi"
Persistence:
  Size: 100Gi
NetworkPolicy:
  ApiVersion: networking.k8s.io/v1
rbac:
  install: true
  serviceAccountName: cd-jenkins


* Helm CLI 를 이용하여 설정 정보와함께 chart 배포
./helm install -n cd stable/jenkins -f jenkins/values.yaml --version 0.16.6 --wait
[Output]
NAME:   cd
LAST DEPLOYED: Sat Mar 30 21:47:24 2019
NAMESPACE: default
STATUS: DEPLOYED

RESOURCES:
==> v1/Secret
NAME        TYPE    DATA  AGE
cd-jenkins  Opaque  2     6s

==> v1/ConfigMap
NAME              DATA  AGE
cd-jenkins        4     6s
cd-jenkins-tests  1     6s

==> v1/PersistentVolumeClaim
NAME        STATUS  VOLUME                                    CAPACITY  ACCESS MODES  STORAGECLASS  AGE
cd-jenkins  Bound   pvc-f7bee8b8-52e9-11e9-b6d3-42010a80004a  100Gi     RWO           standard      6s
==> v1/ServiceAccount
NAME        SECRETS  AGE
cd-jenkins  1        6s
==> v1beta1/ClusterRoleBinding
NAME                     AGE
cd-jenkins-role-binding  6s
==> v1/Service
NAME              TYPE       CLUSTER-IP     EXTERNAL-IP  PORT(S)    AGE
cd-jenkins-agent  ClusterIP  10.11.245.132  <none>       50000/TCP  6s
cd-jenkins        ClusterIP  10.11.248.23   <none>       8080/TCP   6s
==> v1beta1/Deployment
NAME        DESIRED  CURRENT  UP-TO-DATE  AVAILABLE  AGE
cd-jenkins  1        1        1           0          6s
==> v1/Pod(related)
NAME                         READY  STATUS    RESTARTS  AGE
cd-jenkins-5dc9cd6487-zv9r9  0/1    Init:0/1  0         5s
NOTES:
1. Get your 'admin' user password by running:
  printf $(kubectl get secret --namespace default cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
2. Get the Jenkins URL to visit by running these commands in the same shell:
  export POD_NAME=$(kubectl get pods --namespace default -l "component=cd-jenkins-master" -o jsonpath="{.items[0].metadata.name}")
  echo http://127.0.0.1:8080
  kubectl port-forward $POD_NAME 8080:8080
3. Login with the password from step 1 and the username: admin
For more information on running Jenkins on Kubernetes, visit:
https://cloud.google.com/solutions/jenkins-on-container-engine
Configure the Kubernetes plugin in Jenkins to use the following Service Account name cd-jenkins using the following steps:
  Create a Jenkins credential of type Kubernetes service account with service account name cd-jenkins
  Under configure Jenkins -- Update the credentials config in the cloud section to use the service account credential you created in the step above.


* Jenkins pod 이 실행중이고 container 가 READY 상태인지 확인
kubectl get pods
[Output]
NAME                          READY     STATUS    RESTARTS   AGE
cd-jenkins-5dc9cd6487-zv9r9   1/1       Running   0          3m

* Cloud Shell 에서 Jeknins UI 로 포트포워딩 설정
export POD_NAME=$(kubectl get pods -l "component=cd-jenkins-master" -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
(POD_NAME : cd-jenkins-5dc9cd6487-zv9r9)

* Jenkins 서비스 생성 확인
kubectl get svc
[Output]
NAME               TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)     AGE
cd-jenkins         ClusterIP   10.11.248.23    <none>        8080/TCP    5m
cd-jenkins-agent   ClusterIP   10.11.245.132   <none>        50000/TCP   5m
kubernetes         ClusterIP   10.11.240.1     <none>        443/TCP     15m

Jenkins 마스터가 요청할 때마다 빌더 노드들이 자동으로 실행될 수 있도록 Jenkins 의 Kubernetes 플러그인을 사용.
빌더 노드들은 작업이 끝나면 자동으로 종료되고 사용한 리소드들은 클러스터 리소스 풀로 반환.

이 서비스는 selector 와 매치되는 Pod 들에 8080과 50000 포트를 노출함.
이것들은 Kubernetes 클러스터에서 Jenkins 웹 UI 와 builder/agent 등록 포트들임.
추가적으로 jenkins-ui 서비스들은 ClusterIP 를 이용하여 노출 함으로써 클러스터 밖에서는 접근할 수 없도록 함.

=======================================================================================================================

* Jenkins 연결
Jenkins chart 는 관리자 암호를 자동으로 생성하는데 이 암호를 확인하기 위해서 다음을 실행.

printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo

Jenkins user interface 확인을 위해서 Web Preview 를 이용
여기서 admin / 자동 생성된 암호 를 이용하여 로그인

=======================================================================================================================

* gceme 샘플 애플리케이션 이해
Go 언어로 작성되어 있으며 레퍼지토리 sample-app 디렉토리에 위치함.
Compute Engine instance 에서 실행하면 인스턴스의 메타데이터를 카드 형대로 보여줌.
이 애플리케이션은 두개의 모드를 지원함으로써 MSA 를 묘사함.
Backend mode : gceme 는 8080 포트에서 수신 대기하다가 Compute Engine Instance 메타데이터를 JSON 포멧으로 반환
Frontend mode : gceme 는 backend gceme 서비스에 정보를 요청하고 결과 JSON 을 UI 로 출력

=======================================================================================================================

* 애플리케이션 배포
두개의 다른 환경으로 배포
Production : 사용자에게 실제 서비스하는 환경
Canary : 사용자 트래픽중 일부 적은 트래픽만 수용하는 환경으로 전체 사용자에게 서비스하기 전에 테스트를 위해 사용

* 샘플 앱 디렉토리로 이동
cd sample-app

* 배포를 논리적으로 구분짓기 위해서 네임스페이스 생성
kubectl create ns production

* kubectl apply 명령을 통해 production 배포와 canary 배포 그리고 서비스를 생성

kubectl apply -f k8s/production -n production

kubectl apply -f k8s/canary -n production

kubectl apply -f k8s/services -n production

기본적으로 frontend replica 한개만 배포됨.
항상 적어도 4개의 replica 는 실행되도록 하기 위해서 kuvectl scale 명령 실행
production 환경 frontend 스케일 업을 위해서 다음 명령 수행
kubectl scale deployment gceme-frontend-production -n production --replicas 4

* 이제 frontend 를 위해서는 5개의 pod 들을 가지고 있게 되는데, 4개는 production 용이고 1개는 canary 용임.
(canary 배포는 1/5(20%) 사용자에게만 영향을 줌)

kubectl get pods -n production -l app=gceme -l role=frontend
[Output]
NAME                                         READY     STATUS    RESTARTS   AGE
gceme-frontend-canary-c9846cd9c-v8gvz        1/1       Running   0          46s
gceme-frontend-production-6976ccd9cd-9p8mb   1/1       Running   0          12s
gceme-frontend-production-6976ccd9cd-fc4t2   1/1       Running   0          12s
gceme-frontend-production-6976ccd9cd-wczz4   1/1       Running   0          2m
gceme-frontend-production-6976ccd9cd-zpm8m   1/1       Running   0          12s

* 이제 backend 를 위해서 2개의 pod 이 있는 것을 확인하자. 1개는 production 용이고 1 개는 canary 용.

kubectl get pods -n production -l app=gceme -l role=backend
[Output]
NAME                                        READY     STATUS    RESTARTS   AGE
gceme-backend-canary-59dcdd58c5-p7shh       1/1       Running   0          1m
gceme-backend-production-7fd77cf554-66l8w   1/1       Running   0          3m

* production 서비스를 위한 외부 IP 확인

kubectl get service gceme-frontend -n production
[Output]
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
gceme-frontend   LoadBalancer   10.11.253.197   35.232.86.233   80:30989/TCP   2m

* 브라우저에 외부 IP 를 입력하면 결과 확인 가능

* 이제, 나중에 사용하기 위해서 frontend 서비스의 로드밸런서 IP 를 환경 변수에 저장하자.

export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

브라우저에서 frontend 외부 IP를 연결함으로써 두개의 서비스들이 동작함을 확인하자.
다음 명령을 실행하여 서비스의 출력 버전을 확인하자.

curl http://$FRONTEND_SERVICE_IP/versiona

샘플 애플리케이션 배포 완료.
다음은 변경사항을 지속적이고 신뢰성있게 배포하기 위한 파이프라인 설정을 수행.

=======================================================================================================================

Jenkins 파이프라인 생성

* 샘플앱 소스코드를 저장하기 위한 레퍼지토리 생성
gceme 샘플앱을 복사해서 Cloud Source Repository 로 업로드

gcloud alpha source repos create default

* 이 레퍼지토리에는 과금되지 않을 것이므로 경고는 무시.

git init

* 샘플앱 디렉토리를 git 저장소로 초기화

git config credential.helper gcloud.sh

* 다음 명령 실행:

git remote add origin https://source.developers.google.com/p/$DEVSHELL_PROJECT_ID/r/default

* git commit 을 위한 사용자명과 이메일주소 설정. 

git config --global user.email "[EMAIL_ADDRESS]"

git config --global user.name "[USERNAME]"

* 파일을 변경내역에 추가하고 커밋 후 푸시:

git add .

git commit -m "Initial commit"

git push origin master


서비스 계정 인증 정보 추가

Jenkins 가 코드 저장소에 접근하는 것을 허용하기 위해서 인증 정보 수정.
Jenkins 는 Cloud Source Repositories 에서 코드를 다운로드하기 위해서 클러스터의 서비스 계정 인증 정보를 사용할 것임.

Step 1: Jenkins 사용자 인터페이스에서 좌측 네비게이션 메뉴의 Credentials 클릭
Step 2: Jenkins 클릭
Step 3: Global credentials (unrestricted) 클릭.
Step 4: 좌측 메뉴의 Add Credentials 클릭.
Step 5: Kind 드롭다운 메뉴에서 Google Service Account 선택 후 OK.

전역 인증 정보가 추가됨. 인증 정보의 이름은 랩의 CONNECTION DETAILS 에서 찾을 수 있는 GCP Project ID 임.


Jenkins Job 생성

Jenkins 사용자 인터페이스에서 파이프라인 Job 을 설정하기 위해서 다음 단계를 따름.

Step 1: Jenkins 클릭 > 좌측 메뉴에서 New Item :

Step 2: 프로젝트명은 sample-app으로 하고, Multibranch Pipeline 옵션 선택 후 OK.

Step 3: 다음 페이지의 브랜치 소스 섹션에서 Add Source 클릭 후 git 선택.

Step 4: Cloud Source Repositories 에 있는 샘플앱의 HTTPS clone URL 을 Project Repository 필드에 붙여넣기.
[PROJECT_ID] 는 GCP Project ID로 치환:

https://source.developers.google.com/p/[PROJECT_ID]/r/default

Step 5: Credentials 드롭다운에서, 이전 단계에서 서비스 계정 추가 시 생성한 인증 정보의 이름을 선택.

Step 6: Scan Multibranch Pipeline Triggers section 에서, Periodically if not otherwise run 박스를 체크하고 Interval 값을 1 분으로 지정.
Step 7: Job 설정 확인

Step 8: 다른 옵션들은 기본 상태로 두고 저장 클릭.

이 단계들을 완료하고 난 후, "Branch indexing" Job 실행. 이 meta-job은 레퍼지토리에서 브랜치들을 식별하고 존재하는 브랜치들에 변경사항이 없음을 확인함.
좌상단의 sample-app 을 클릭하면, 마스터 Job 이 보임.

마스터 Job 의 첫실행은 다음 단계에서 코드를 조금 수정하기 전까지는 실패할 것임.
Jenkins 파이프라인 생성 성공. 다음 단계에서 CI 를 위한 개발 환경을 생성할 것임.


=======================================================================================================================

개발 환경 생성
개발 브랜치들은 개발자들이 실제 서비스로 코드를 반영하기 전에 테스트하기 위해서 사용하는 환경의 집합임.
이 환경들은 애플리케이션의 축소버전이지만 실제 환경과 동일한 매커니즘을 사용하여 배포될 필요가 있음.

개발 브랜치 생성
feature 브랜치에서 개발 환경을 생성하기 위해서, 브랜치를 Git 서버로 푸시하고 Jenkins 가 여러분의 환경을 보포하도록 할 수 있다.

개발 브랜치 생성 및 Git server로 푸시:

git checkout -b new-feature

파이프라인 설정 변경
파이프라인을 정의하는 Jenkinsfile 은 Jenkins 파이프라인 그루비 문법을 이용해 쓰여졌음. Jenkinsfile 을 사용하면 전체 빌드 파이프라인을 소스 코드와 함께 존재하는 하나의 파일로 포현할 수 있음. 파이프라인은 병렬화 같은 강력한 기능을 지원하며 사용자의 직접 승인을 필요로 함.

파이프라인이 예상대로 동작하도록 하기 위해서 Jenkinsfile 에 project ID 설정이 필요.

vi 등의 에디터로 Jenkinsfile 오픈.

PROJECT_ID 값에 project id 할당. (PROJECT_ID 는 랩의 CONNECTION DETAILS에서 찾을 수 있는 GCP Project ID. gcloud config get-value project 명령으로도 찾을 수 있음):

def project = 'PROJECT_ID'
def appName = 'gceme'
def feSvcName = "${appName}-frontend"
def imageTag = "gcr.io/${project}/${appName}:${env.BRANCH_NAME}.${env.BUILD_NUMBER}"

저장하고 종료


사이트 변경
애플리케이션의 변경을 묘사하기 위해서, gceme 카드를 파란색에서 오렌지 색으로 변경할 것임.

html.go 파일 오픈:
<div class="card blue"> 가 있는 엘리먼트 두개를 다음과 같이 변경:
<div class="card orange">
저장 후 종료

main.go 파일 오픈:
다음 라인에 버전이 명시됨:
const version string = "1.0.0"
해당 라인을 다음과 같이 변경:
const version string = "2.0.0"
파일 저장 후 종료


=======================================================================================================================
배포 시작

변경 사항을 커밋 후 푸시:

git add Jenkinsfile html.go main.go

git commit -m "Version 2.0.0"

git push origin new-feature

이것은 개발 환경의 빌드를 트리거함.
변경사항이 Git 저장소에 푸시된 이후, Jenkins 사용자 인터페이스로 이동하면 새로운 new-feature 브랜치의 빌드가 시작되었음을 확인 가능.
이 변경사항이 적용될 때까지 1분정도 걸림.

After the build is running, click the down arrow next to the build in the left navigation and select Console output:
빌드가 수행된 후, build 다음에 있는 아래방향 화살표를 클릭하고 콘솔 출력을 선택:

시작하기 위해서 몇분간 출력되는 빌드 결과를 지켜보며 kubectl --namespace=new-feature apply... 메시지가 나오는지 확인.
new-feature 브랜치는 클러스터에 배포될 것임.

개발 시나리오에서는 퍼블릭 로드 밸런서를 사용하지 않음. 애플리케이션의 보안을 위해서 kubectl 프록시를 사용 가능.
이 프록시는 Kubernetes API 로 자체 인증을 수행하고, 로컬 머신에서 클러스터로의 요청 프록시를 서비스가 인터넷에 노출될 우려 없이 수행해 줌.

만약 Build Executor 에서 어떠한 결과도 확인할 수 없더라도 걱정하지 말자. Jenkins 홈으로 이동하여 new-feature 파이프라인이 생성되었는지 확인하자.

모두 정상적으로 설정되고나면, 프록시를 백그라운드에서 시작.

kubectl proxy &

만약 실패하면 Ctrl + X 를 눌러 빠져나온 후 localhost 로 요청을 보내 애플리케이션이 접근 가능한지 확인하고 kubectl 프록시가 서비스로 전달할 수 있도록 함.

curl \
http://localhost:8001/api/v1/proxy/namespaces/new-feature/services/gceme-frontend:80/version

2.0.0 응답을 볼 수 있으며, 이것이 현재 수행중인 버전임.

개발 환경 설정 완료. 다음은, 이전에 배운것을 토대로 새로운 기능을 테스트하기 위한 canary 릴리스 배포.

=======================================================================================================================

Canary 릴리스 배포
개발환경에서 앱이 최신 버전으로 동작중임을 확인함. 이제 이것을 canary 환경에 배포하자.

canary 브랜치 생성 및 Git 서버로 푸시

git checkout -b canary

git push origin canary

Jenkins 에서 canary 파이프라인이 시작된 것을 확인. 완료되고 나면, 몇몇 트래픽은 새로운 버전으로 전달되는 것을 확인하기 위한 서비스 URL 확인 가능.
순서에 관계없이 1/5의 요청은 2.0.0 버전을 반환함을 확인할 수 있음.

export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done

If you keep seeing 1.0.0, try running the above commands again. Once you've verified that the above works, end the command with Ctrl+C.
만약 계속해서 1.0.0 이 보인다면 위 명령을 다시한번 수행하자. 위 변경 사항이 정상동작하는 것을 확인하고 나면, Ctrl+C 를 눌려 명령 종료.
Canary 릴리스 배포 완료! 다음은, 새로운 버전을 production 으로 배포.

=======================================================================================================================

production 으로 배포

지금까지 canary 릴리스는 성공적이었으며 고객의 불만은 없었으므로 전체 production 으로 배포를 진행.

canary 브랜치를 머지하고 Git 서버로 push:

git checkout master

git merge canary

git push origin master

Jenkins 에서 마스터 파이프라인이 시작된 것을 확인. 완료된 후에, 모든 트래픽이 새로운 버전으로 전달되는 서비스 URL 확인.

export FRONTEND_SERVICE_IP=$(kubectl get -o \
jsonpath="{.status.loadBalancer.ingress[0].ip}" --namespace=production services gceme-frontend)

while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1; done

다시 한번, 만약 1.0.0 인스턴스를 계속 보게 된다면 위 명령을 다시 실행. Ctrl + C 로 명령 정지 가능.

gceme 애플리케이션이 카드 형태로 정보를 보여주는 사이트로 이동 가능.
카드 색상이 파란색에서 오렌지 색으로 변경됨.
다음 명령으로 외부 IP 주소 확인 가능.

kubectl get service gceme-frontend -n production

성공적으로 production 배포 완료.











