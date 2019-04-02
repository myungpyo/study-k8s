# 파이프라인 아키텍쳐
지속적으로 최신 버전의 애플리케이션을 사용자에게 제공하기 위해서는 신뢰성 있게 자동화된 소프트웨어 빌드와 테스트, 그리고 업데이트가 필요하다. 코드 변경은 자동적으로 파이프라인을 타고 아티펙트를 생성하고, 단위 테스팅, 기능 테스팅, 제품 배포로 이어져야 한다. 어떤 경우에는 일부 사용자에게만 코드 업데이트를 수행함으로써 서비스 전체적으로 변경사항을 배포하는 위험 없이 사전에 변경사항을 실험해 보고 싶을 수 있다. 만약 이 실험 배포 버전중 하나가 만족스럽지 않다면 이 자동화 프로세스는 빠르게 소프트웨어 변경사항을 이전 상태로 돌릴 수 있어야 한다(RollBack).  

쿠버네티스 엔진과 스피너커(Spinnaker)를 이용하여 견고한 지속적 배포 프로세스를 생성할 수 있고, 이것은 소프트웨어가 개발되고 기대하는 동작이 확인됨과 동시에 배포될 수 있도는 환경을 만들 수 있도록 돕는다.  
빠른 개발 프로세스가 최종 목표라고 하더라도, 각 애플리케이션 리비전들이 실제 배포 후보가 되기 전에 모든 자동화된 검사 프로세스를 통과해야 하는 것은 꼭 필요한 사항이다. 자동화된 검사를 이용하여 애플리케이션이 확인되었다 하더라도, 필요하다면 수동으로 애플리케이션을 검사하고 추가적인 배포 전 테스트를 수행할 수 있다.

팀이 애플리케이션의 배포 준비가 끝났다고 결정하고 나서는, 팀원 중 한명이 실제 서비스 배포를 수행할 수 있다.  

이번 랩에서 다음 다이어그램과 같은 지속적 배포 파이프라인을 만들어볼 것이다.

diagram 1

---

## 환경 설정
랩 진행을 위해 필요한 인프라스트럭처와 아이덴티티를 설정.
먼저 Spinnaker 와 샘플 애플리케이션을 배포할 Kubernetes 엔진 생성.

Set the default compute zone:
gcloud config set compute/zone us-central1-f

Spinnaker 학습 샘플 애플리케이션으로 Kubernetes 엔진 생성:
gcloud container clusters create spinnaker-tutorial --machine-type=n1-standard-2 --enable-legacy-authorization

이 배포는 완료될 때까지 5~10분정도 걸림. 기본 스코프에 대한 경고를 볼 수 있지만 이번 랩에는 지장이 없으므로 무시해도 좋음.
완료되고나면 실행 중인 클러스터의 다음과 같은 상세한 상태들이 보고됨.
name, location, version, ip-address, machine-type, node version, number of nodes and status of the cluster

아이덴티티 및 접근 제어 설정
Spinnaker 가 클라우드 스토리지에 데이터를 쓸수있게 권한 부여하기 위한 클라우드 IAM 서비스 계정을 생성. Spinnaker 는 파이프라인 데이터를 클라우드 스토리지에 저장하므로써 신뢰성있고 회복 가능한 상태로 동작함. 만약 Spinnaker 배포가 예상치 못하게 실패할 경우 수 분 내에 원본 파이프라인 데이터와 동일한 데이터에 접근하여 동일한 배포를 생성할 수 있다.

다음 단계들을 따라서 시작 스크립트를 클라우드 스토리지 버킷에 업로드

서비스 계정 생성:
gcloud iam service-accounts create spinnaker-storage-account \
    --display-name spinnaker-storage-account

서비스 계정 이메일 주소와 현재 프로젝트 ID 를 환경 변수에 저장하여 나중에 사용 :
export SA_EMAIL=$(gcloud iam service-accounts list \
    --filter="displayName:spinnaker-storage-account" \
    --format='value(email)')

export PROJECT=$(gcloud info --format='value(config.project)')

서비스 계정에 storage.admin 롤을 부여:

gcloud projects add-iam-policy-binding \
    $PROJECT --role roles/storage.admin --member serviceAccount:$SA_EMAIL

서비스 계정 키 다운로드. 나중 단계에서 Spinnaker 를 설치하고 이 키를 Kubernetes 엔진으로 업로드.

gcloud iam service-accounts keys create spinnaker-sa.json \
     --iam-account $SA_EMAIL

========================================================================================================================

Helm 을 이용한 Spinnaker 배포
이번 섹션에서는 Chart 레퍼지토리로부터 Spinnaker 를 배포하기위허 Helm 을 이용함.
Helm 은 Kubernetes 애플리케이션을 설정, 배포하기위해 사용할 수 있는 패키지 매니저.

Helm 설치
Helm 바이너리 다운로드 및 설치:
wget https://storage.googleapis.com/kubernetes-helm/helm-v2.5.0-linux-amd64.tar.gz

로컬에 압축해제
tar zxfv helm-v2.5.0-linux-amd64.tar.gz
cp linux-amd64/helm .

클러스터에 Helm 서버사이드인 Tiller 설치하기위해 Helm 초기화:
./helm init
[Output]
Creating /home/google2918759_student/.helm
Creating /home/google2918759_student/.helm/repository
Creating /home/google2918759_student/.helm/repository/cache
Creating /home/google2918759_student/.helm/repository/local
Creating /home/google2918759_student/.helm/plugins
Creating /home/google2918759_student/.helm/starters
Creating /home/google2918759_student/.helm/cache/archive
Creating /home/google2918759_student/.helm/repository/repositories.yaml
$HELM_HOME has been configured at /home/google2918759_student/.helm.

Tiller (the helm server side component) has been installed into your Kubernetes Cluster.
Happy Helming!

./helm repo update
[Output]
Hang tight while we grab the latest from your chart repositories...
...Skip local chart repository
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈

Helm 이 제대로 설치되었는지 다음 명령으로 확인. 정상적으로 서리되었다면 클라이언트와 서버에 v2.5.0 버전이 보임.
./helm version
[Output]
E0331 11:38:59.404255     477 portforward.go:212] Unable to create listener: Error listen tcp6 [::1]:42121: bind: cannot assign requested address
Client: &version.Version{SemVer:"v2.5.0", GitCommit:"012cb0ac1a1b2f888144ef5a67b8dab6c2d45be6", GitTreeState:"clean"}
Server: &version.Version{SemVer:"v2.5.0", GitCommit:"012cb0ac1a1b2f888144ef5a67b8dab6c2d45be6", GitTreeState:"clean"}

Helm 을 설치하고 설정하는동안 리스너 서비스나 포트 바인딩 설정과 관련된 오류를 볼 수 있다. 이것은 현재 Helm 버전이 가진 이슈로 클라우드 쉘 IPv6 설정과 관련이 있다. 이 오류는 랩에 영향을 주지 않으므로 무시해도 좋다.

Spinnaker 설정
In Cloud Shell, create a bucket for Spinnaker to store its pipeline configuration:
클라우드 쉘에서 Spinnaker 의 파이프라인 설정을 저장하기 위한 버킷을 생성

export PROJECT=$(gcloud info \
    --format='value(config.project)')

export BUCKET=$PROJECT-spinnaker-config

gsutil mb -c regional -l us-central1 gs://$BUCKET
[Output]
Creating gs://qwiklabs-gcp-74ae5829762b41c7-spinnaker-config/...

다음 명령을 수행해서 설정 파일 생성:

export SA_JSON=$(cat spinnaker-sa.json)
export PROJECT=$(gcloud info --format='value(config.project)')
export BUCKET=$PROJECT-spinnaker-config
cat > spinnaker-config.yaml <<EOF
storageBucket: $BUCKET
gcs:
  enabled: true
  project: $PROJECT
  jsonKey: '$SA_JSON'

# Disable minio the default
minio:
  enabled: false

# Configure your Docker registries here
accounts:
- name: gcr
  address: https://gcr.io
  username: _json_key
  password: '$SA_JSON'
  email: 1234@5678.com
EOF


Spinnaker chart 배포
Helm 명령행 인터페이스를 사용하여 설정 세트와 차트를 배포. 이 명령은 보통 5~10분정도 걸림.

./helm install -n cd stable/spinnaker -f spinnaker-config.yaml --timeout 600 \
    --version 0.3.1

여기서 생성되지 않은 리스너 경고를 볼 수 있지만 무시해도 좋은. 이후로 몇분정도 설치 과정 계속.
명령이 완료된 이후, 클라우드 쉘에서 Spinnaker 로 포트포워딩 설정을 위해 다음 명령 수행.

export DECK_POD=$(kubectl get pods --namespace default -l "component=deck" \
    -o jsonpath="{.items[0].metadata.name}")
kubectl port-forward --namespace default $DECK_POD 8080:9000 >> /dev/null &

Spinnaker 사용자 인터페이스를 열기위해 클라우드 쉘 윈도우 좌상단의 Web Preview 아이콘을 누르고 포트 8080으로 사용.

SPinnaker 사용자 인터페이스에서 환영 메시지 확인.
(Spinnaker 사용자 인터페이스 접근을 위해 사용할 것이므로 탭을 연상태도 유지)

========================================================================================================================

도커 이미지 생성
다음으로 애플리케이션 소스 코드 변경을 탐지하고 도커 이미지를 빌드하고 컨테이너 레지스트리에 등록하기 위한 컨테이너 빌더 설정

소스 코드 레지스트리 생성
클라우드 쉘 탭으로 돌아가서 샘플 애플리케이션 소스코드 다운로드:

wget https://gke-spinnaker.storage.googleapis.com/sample-app.tgz

소스코드 압축 해제:

tar xzfv sample-app.tgz

샘플 애플리케이션 소스 디렉토리로 이동:

cd sample-app

레퍼지토리에서 커밋을 위한 이메일 주소와 사용자 이름 설정.
[EMAIL_ADDRESS], [USERNAME] 을 랩에서사용할 메일 주소와 사용자명으로 변경.

git config --global user.email "[EMAIL_ADDRESS]"

git config --global user.name "[USERNAME]"

소스 코드 레퍼지토리에서 최초 커밋 생성.

git init

git add .

git commit -m "Initial commit"

코드를 저장할 레퍼지토리 생성:

gcloud source repos create sample-app

과금될 수 있다는 메시지 무시.

git config credential.helper gcloud.sh

새로 생성된 레퍼지토리를 리모트에 추가.

export PROJECT=$(gcloud info --format='value(config.project)')

git remote add origin https://source.developers.google.com/p/$PROJECT/r/sample-app

코드를 새 레퍼지토리 마스터 브랜치로 푸시:

git push origin master

콘솔에서 소스코드 확인해보기
Navigation menu > Tools section > Source Repository > Repositories

빌드 트리거 설정
컨테이너 빌더를 설정하여 소스 레퍼지토리에 git 태그가 푸시될 때마다 도커 이미지를 생성하고 푸시하도록 만들기.
컨테이너 빌더는 자동적으로 소스 코드를 체크아웃하고 레퍼지토리 Dockerfile 을 이용하여 이미지를 생성하고 Google Cloud Container Registry로 푸시.

클라우드 플랫폼 콘솔에서 Navigation menu > Cloud Build > Build triggers.
트리거 생성 클릭
Cloud Source Repository 선택 후 [계속] 클릭.
새로 생성 한 sample-app 라디오 버튼 클릭하고 [계속] 클릭.

다음과 같이 트리거 설정:

Name:sample-app-tags
Trigger type: Tag
Tag (regex): v.*
Build configuration: cloudbuild.yaml
cloudbuild.yaml location: /cloudbuild.yaml

트리거 생성 클릭.

지금부터 소스코드 레퍼지토리에 v로 시작하는 태그를 푸시 할 때마다 컨테이너 빌더는 자동적으로 애플리케이션을 도커 이미지로 빌드하고 컨테이너 레지스트리에 푸시한다.

이미지 빌드
다음 단계를 따라 첫번째 이미지를 빌드하자:

클라우드 쉘로 돌아와서 git 태그 생성:

git tag v1.0.0

현재 sample-app 디렉토리인 것을 확인하고 tag 를 푸시:

git push --tags

콘솔로 돌아와서, 클라우드 빌드에서 빌드 기록을 클릭하여 빌드가 시작되었는지 확인. 아니라면 이전 섹션에서 트리거가 제대로 설정되었는지 확인.

========================================================================================================================

개발 파이프라인 설정
이제 이미지들이 자동 빌드되고 있으며, 이것들을 Kubernetes 클러스터로 배포할 필요가 있다.

테스트를 위해 축소된 환경으로 배포를 수행해보자. 테스트가 성공한 후에 코드 변경사항을 실제 서비스에 배포하기 위해서는 수동으로 승인해야 한다.

애플리케이션 생성
Spinnaker 탭에서 액션을 클릭하고 애플리케이션 생성 선택

새 애플리케이션 다이얼로그에서 다음과 같이 입력

Name: sample
Owner Email: [your lab username/email address]
생성 클릭


서비스 로드밸런서 생성
정보를 수동으로 입력하는 것을 피하기위해서 Kubernetes 명령행 인터페이스를 사용하여 서비스 로드밸런서를 생성하자.
대신해서, Spinnaker 페이지에서 이 동작을 수행할 수도 있다.

클라우드 쉘로 이동하여 샘플 앱의 루트 디렉토리에서 다음 명령 실행:

kubectl apply -f k8s/services
[Output]
service/sample-backend-canary created
service/sample-backend-prod created
service/sample-frontend-canary created
service/sample-frontend-prod created

배포 파이프라인 생성
다음으로 지속적 배포를 위한 파이프라인을 생성하자. 이 랩을 위해서 파이프라인은 컨테이너 레지스트리에 v로 시작하는 태그를 갖는 도커 이미지를 탐지하도록 설정될 것이다.

Spinnaker 인스턴스에 예제 파이프라인을 업로드 하기위해서 다음 명령을 실행하자:

export PROJECT=$(gcloud info --format='value(config.project)')
sed s/PROJECT/$PROJECT/g spinnaker/pipeline-deploy.json | curl -d@- -X \
    POST --header "Content-Type: application/json" --header \
    "Accept: /" http://localhost:8080/gate/pipelines

Spinnaker 탭의 상단 네비게이션 바에서 파이프라인 클릭. 이곳에 Deploy 라는 1개 파이프라인이 준비되어 있는 것을 확인.

설정을 클릭하고 배포 선택.

지속적 배포 파이프라인 설정이 다음과 같이 보임.

방금 만든 설정은 v로 시작하는 git 태그르 푸시할 때 파이프라인을 시작하는 트리거를 포함함. 다음에는 수동으로 파이프라인을 실행하여 테스트 해보고 나서, git 태그를 푸시해서 테스트해보면서 자동으로 파이프라인이 동작하는지 확인해보자.

========================================================================================================================

파이프라인 수동으로 실행
파이프라인을 클릭하여 파이프라인 페이지로 돌아가자.

배포 다음에 있는 수동 실행 시작 클릭
태그 드롭-다운 목록은 기본적으로 v1.0.0 태그가 선택되어 있음. 클릭하여 실행.
만약 페이지 상단에서 글로벌 수동 실행 버튼을 클릭했다면 아마도 파이프라인 드롭다운 목록에서 먼저 배포 파이프라인을 선택해야 할 수 있음.
파이프라인이 시작된 후, 빌드 프로세스에 대해서 더 자세히 확인해보기 위해서 상세를 펼침.

이 섹션은 배포 파이프라인의 상태와 단계를 보여줌:

blue - 현재 실행 중
green - 성공적으로 완료
red - 실패함
상세한 내용을 보기 위해서는 해당 단계 클릭.

3분정도 후 테스트 단계는 끝이나고 파이프라인은 이어서 배포를 수행하기 위해서 수동 승인이 필요함.
노란색 "사람" 아이콘을 클릭하고 "계속" 클릭.
배포가 실서비스 프론트엔드와 백엔드 배포로 이어짐. 몇분정도 후에 완료됨.
앱을 보기 위해서 Spinnaker 우상단에서 로드밸런서를 클릭.
밸런서들 목록에서 아래로 스크롤하여 sample-frontend-proj 아래 Default 클릭.

우측에 상세 화면을 아래로 스크롤하고 접속을 위해서 애플린케이션 IP 주소에 있는 클립보드 버튼을 클릭하여 복사함.
애플리케이션을 신버전 실서비스를 확인하기 위해서 브라우저 새탭에서 주소를 붙여넣기.
지금까지 애플리케이션을 빌드, 테스트, 배포하기위한 파이프라인을 수동으로 시작시켜보았음.

========================================================================================================================

코드 변경으로 파이프라인 시작시키기.
이제 코드를 변경하고 git 태그를 푸시하고 그에 반응하여 수행되는 파이프라인을 살펴보면서 이 과정의 시작부터 끝을 확인해보자.
v 로 시작하는 git 태그를 푸시하면 컨테이너 빌더가 새로운 도커 이미지를 빌드해서 컨테이너 레지스트리로 푸시하게 만든다.
Spinnaker 는 v로 시작하는 새로운 이미지를 탐지하고 해당 이미지를 카나리(시험)버전으로 푸시하고, 테스트하고, 동일 버전을 모든 pod에 배포하기 위해서 파이프라인을 시작시킴.

클라우드 쉘로 돌아와서 애플리케이션의 컬러를 오렌지에서 블루로 변경하기 위한 코드 변경을 수행하자.

sed -i 's/orange/blue/g' cmd/gke-info/common-service.go

변경사항을 태깅하고 소스 코드 저장소에 푸시하자.

git commit -a -m "Change color to blue"

git tag v1.0.1

git push --tags

콘솔에서 클라우드 빌드 > 빌드 히스토리를 열고 새로운 빌드가 보일때까지 몇분 기다리자. 페이지 갱신을 해야 할 수도있다.
Spinnaker 탭으로 돌아와서 새로운 이미지를 배포하기 위한 파이프라인이 시작된 것을 확인하기 위해서 파이프라인을 클릭하자. 자동으로 시작된 파이프라인은 보일때까지 몇분 걸릴 것이다. 패이지 갱신이 필요할 수 있다.

앱의 카나리 배포를 확인하기 위해서 Spinnaker 페이지 우측 상단에서 로드밸런서를 클릭.
로드 밸런서 목록을 아래로 스크롤하여 sample-fronted-canary 아래 Default 클릭.
우측에 상세 화면을 아래로 스크롤하여 애플리케이션 IP 주소 영역의 클립보드 버튼을 클릭하여 복사. 
주소를 브라우저 새로운 탭에 붙여넣기. 새로운 파란색 버전의 애플리케이션을 확인하여 성공적으로 배포되었음을 확인.

테스트가 끝난 후, Spinnaker 탭으로 돌아와서 파이프라인 클릭.
(사람 아이콘의) 노란색 승인 버튼을 클릭하고 배포를 계속하기 위해서 "계속" 클릭.

파이프라인이 완료되면 실 서비스 애플리케이션 탭을 새로고침. 이제 실 서비스 애플리케이션도 파란색으로 보이며 버전 필드는 v1.0.1 임을 확인.

전체 서비스 환경으로 애플리케이션 배포를 성공적으로 완료.

========================================================================================================================

변경사항 롤백 (Optional)
이전 커멧을 되돌림으로써 변경을 롤백할 수 있음. 롤백은 새로운 태그(v1.0.2)를 추가하고, 추가된 태그를 배포 v1.0.1 에서 사용된 파이프라인으로 푸시함.
테스트해보고자 한다면 클라우드 쉘로 돌아가서 변경사항을 롤백하기 위해서 다음 명령을 수행.

git revert v1.0.1 --no-edit

git tag v1.0.2

git push --tags

롤백을 확인해보기 위해서, sample-frontend-canary 로드 밸런서 클릭하고 Default 클릭하고 새로운 탭에서 IP 주소를 붙여넣어 확인.
이제 앱이 다시 오렌지 색상으로 돌아오고 버전이 변경된 것을 확인할 수 있음.

