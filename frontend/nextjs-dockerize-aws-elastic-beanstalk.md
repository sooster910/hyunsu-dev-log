---
description: >-
  이번에 dockerize를 하게 되었다. 1년전까지만 해도 zip파일을 통해 eb에 수동으로 업로드 하는 방식에서 eb cli를 통해 배포
  방식으로 옮기고 거기서 다시 github action을 얹고 이제는 dockerize까지 왔다. aws eb를 1년동안 운영하면서 모든
  과정을 다 거쳐왔다.
hidden: true
---

# Nextjs Dockerize 및 AWS Elastic Beanstalk 배포기

먼저 결론을 말하자면, "Dockerize하길 잘했다" 이다. docker가 팀협업에서는 내 컴퓨터에서는 잘 되는데요? 를 해결하기 위해 필수적으로 쓰이지만, 1인개발 에서는 사실  docker를 사용하지 않고도 eb cli를 사용해 완전한 자동화는 아니지만 수동 배포 보단 자동화로 유지할 수 있었다. &#x20;

### TL:DR : Docker 배포 과정에서 Docker로 배포하길 잘했다.

1. docker로 배포함으로써 기존 설정들이 불필요해지는것들이 많아졌고 이를 지움으로써 간소해졌다. &#x20;
   1. 기존에는 EB환경에는 yarn이 없어 npm 을 따로 설치하거나 하는 설정을 해줘야했다. Elastic Beanstalk의 .platform/config.yml 과&#x20;
   2.  **Docker 환경에서는 EC2에 직접 명령을 내리지 않음**

       대신, **Docker 컨테이너 내부에서 필요한 모든 명령을 실행**해야 합니다.

       이 작업은 **Dockerfile로 대체**할 수 있습니다.
2. 환경 설정의 관리포인트가 중앙화 되었다.



### Dockerize하기 전 기존 문제점&#x20;

프로젝트를 사실 매일 보는 편이 아니였고, eb사용법이 많이 익숙치 않을 때는 자주 까먹기도 했고, 다시 문서를 찾아본다던가 하는 일들이 있었다. EB의 history가 있으나 사실 버전관리가 되고 있는건 아니였다. 그러니 저번엔 어디가 문제였더라.. 하면서 찾는데 시간을 많이 쓰이니 생산성이 너무 낮아졌다. 어떨 땐 수동으로 배포를 많이 하기도 했다 그러다 보니 완전 자동화를 해서 생산성이 좀 더 나아지길 기대 했었다. 그래서 git action을 적용하기로 하고 프로젝트를 진행하던 중 환경변수의 전달 타이밍 이슈를 해결하면서 환경변수들에대한 관리포인트가 많아짐을 경험했다.  구체적인 문제는 다음과 같다.

### 1. 환경 변수 전달 타이밍  문제점 :&#x20;

GitHub Actions에서 EB로 배포할 때, 만약 환경 변수들이 빌드 시점에 전달되지 않았다면, `next.config.js`에서 설정한 `process.env.NEXT_PUBLIC_*` 값이 **undefined**로 처리됩니다. 이 경우, Next.js는 `next.config.js`의 설정을 반영할 수 없고, 클라이언트 사이드에서 해당 환경 변수를 사용하려 할 때 `undefined`가 나오는 것입니다.

빌드된 파일에 환경변수가 없다. 이 에러가 계속 남&#x20;



### 원인

GitHub Actions에서 EB로 배포할 때 환경 변수를 `next.config.js`에서 설정한 것만으로는 부족할 수 있습니다. 그 이유는, Next.js가 **빌드 시점에 환경 변수**를 읽기 때문에, GitHub Actions에서 빌드를 할 때 해당 환경 변수가 설정되지 않으면 제대로 처리되지 않습니다.



배포하는 GitHub Actions 파이프라인에서는 **빌드 시점에 환경 변수 값을 전달**해줘야 합니다. 예를 들어, GitHub Actions의 `env`나 `secrets`를 사용하여 빌드 시점에 환경 변수를 명시적으로 전달해주어야 하는데, 그렇지 않으면 `next.config.js`에서 설정한 환경 변수는 **빌드 타임에 반영되지 않습니다.**





### **해결 방안:**

GitHub Actions에서 환경 변수를 전달

```

run: |
          yarn build
        env:
          NEXT_PUBLIC_API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}  # GitHub secrets에서 환경 변수를 전달
```

***

#### docker 이미지를 로컬에서 빌드 후 ECR 에 ㅎ푸시 -> EB에서 ECR 이미지를 참조해 컨테이너 실행

단점 : 추가적 ECR사요에 따른 저장 비용 발생&#x20;

ECR을 사용하지 않고 Docke Hub방식으로 이미지를 푸시하고 EB에서 해당 이미지를 가져오는 방식 이 경우,&#x20;

Docker Hub은 AWS의 서비스가 아니기 때문에 보안관리 및 통합 측면에서 AWS ECR보다 불편할 수 있다.



***

### 2. 하지만 관리포인트가 너무 많다.&#x20;

**Elastic Beanstalk** 배포 시의 환경 변수 설정 간소화 관리포인트를 줄이기 위한 대안은 없을까?

### Docker file을 이용한다면?&#x20;

* docker image자체에 환경 변수를 정의할 수 있다
* 어플리케이션의 모든 의존성 및 환경변수를 Docker 컨테이너 내부에서 관리할 수 있다.
* 이렇게 하면 개발환경과 배포 환경에서의 설정을 동일하게 유지할 수 있다.
* Dockerfile이나  Docker Compose파일에서 환경변수를 설정하면, 다양한 배포 환경에서도 일관되게 환경변술르 관리할 수 있다. 예를 들어 Github Actions에서 설정한 환경변수나 EB에서 설정한 환경변수들을 Dockerfile이나 Docker compose로 대체할 수 있다.&#x20;



그렇게 해서 시작된 Dockerize 이다.



### 관리포인트가 얼마나 줄었을까?&#x20;



###

### Deployment process <a href="#deployment-process" id="deployment-process"></a>

이번 배포 과정은 ECR을 사용하지 않고 , **Dockerfile**을 기준으로 애플리케이션을 Docker 컨테이너로 빌드하여 **Elastic Beanstalk**에 배포하는 방식으로 진행했다.

### 진행순서

* Next.js 버전 업그레이드
* npm 의존성 문제 해결
* Docker 환경 설정
  * dev.Dockerfile 생성
  * 이미지 빌드
  * 컨테이너 실행 테스트
* AWS IAM Access Key 생성
* GitHub Actions 설정
  * GitHub Secrets에 AWS 키 등록
  * Workflow 파일 (.github/workflows/deploy-dev.yml) 생성
* Elastic Beanstalk 환경 생성
  * `eb create development-aiff-front` 실행
  * 환경 설정 확인
* 배포 테스트
  * develop 브랜치에 push
  * GitHub Actions 동작 확인
  * 배포 결과 확인\


```yaml
# 특정 Dockerfile 지정하기
docker build -f dev.Dockerfile -t <imageName>:dev .

# 명령어 설명
-f dev.Dockerfile : "dev.Dockerfile"이라는 파일을 사용하겠다
-t next-aiff:dev  : 이미지 이름을 "next-aiff"로, 태그를 "dev"로 하겠다
.                 : 현재 디렉토리를 빌드 컨텍스트로 사용
```



### IAM&#x20;

<figure><img src="../.gitbook/assets/image (4).png" alt=""><figcaption></figcaption></figure>



권한의 경우,  AdministratorAccess 를 부여해 안정화 되기 전까진 권한 문제로 병목이 없게끔 모든 권한을 가지도록 했다.\
\


<figure><img src="../.gitbook/assets/image (5).png" alt=""><figcaption></figcaption></figure>



### Access key생성

*

### &#xD;

<figure><img src="../.gitbook/assets/image (6).png" alt=""><figcaption></figcaption></figure>





<figure><img src="../.gitbook/assets/image (7).png" alt=""><figcaption></figcaption></figure>

### Github Action설정

GitHub Actions를  진행하기 위해 workflow .yml 파일에 있는 태스크와 정보들을  통해  AWS에 배포 한다. 이 때 인증이 필요한데, 이 인증을 위해 AWS Access Key와 Secret Key가 필요하다.  예를 들어 develop 브랜치에 코드를 push ->GitHub Actions가 자동으로 배포 시작 -> 이때 AWS를 사용할 권한이 필요함 -> "너 누구니? 권한있어?" -> AccessKey와 SecretKey로 인증 ->인증 성공하면 Elastic Beanstalk에 배포 진행\
\
\


1. GitHub Secrets에 AWS 키 등록&#x20;

* Access key&#x20;
* Secret key&#x20;

\


<figure><img src="../.gitbook/assets/image (8).png" alt=""><figcaption></figcaption></figure>

<figure><img src="../.gitbook/assets/image (9).png" alt=""><figcaption></figcaption></figure>





2. Workflow 파일 (.github/workflows/deploy-dev.yml) 생성



배포를 하게 되면 현재  EB에서 http도메인이 주어진다.&#x20;

외부 와의 안전한 통신을 위해 https 를 사용하려면 Application Loadbalancer를 추가 하고 AWS Certification Manager에서 인증거를 발급받아 로드밸런서에 연결 시켜 준다.



### Elastic Beanstalk의 기본 동작 Customizing 처리

* 빌드에 실패한 경우가 있었는데  미처 변경하지 못한 package.json의 start  script를 변경하지 못해 이전 버전으로 실행된게 원인이다. start를 변경하거나 기본 동작을 start가 아닌 다른 명령어로 커스터 마이징할 수 있다.&#x20;

### 네트워크 흐름 정리&#x20;

* 클라이언트가 HTTPS 로 요청을 보낸다.
* 로드밸런서가 HTTPS 요청을 받아 암호를 해독하고 (SSL처리), HTTP어플리케이션 서버 (EC2 인스턴스)에 전달한다.
* EC2인스턴스 내에서 NGINX가 요청을 받아   정적파일을 서빙하거나 동적요청인 경우 실제 어플리케이션 서버로 전달을 한다.(리버스 프록시)&#x20;
* 다시 NGINX가 응답을 받아 클라이언트로 전달한다.

***

### 환경변수 핸들링 하기

* Next.js 는 빌드 시 환경 변수를 소스 코드에 하드코딩
* 빌드 시점에 **next.config.js**와 **.env.production**에 정의된 모든 환경 변수가 소스 코드에 포함됩니다.
* 예를 들어, `process.env.NEXT_PUBLIC_API_URL` 같은 코드는 **실제 URL 문자열로 대체됩니다**.

#### 디버깅 방법&#x20;

* **.env.production 파일이 있는지?**
* **Elastic Beanstalk 환경 변수가 설정되었는지?**
* **next.config.js에 env 속성이 설정되었는지?**
* **build 후 .next/server/pages/index.js에 환경 변수가 하드코딩되었는지**

***

###

<figure><img src="../.gitbook/assets/image (11).png" alt=""><figcaption></figcaption></figure>

배포 후 NEXT\_PUBLIC\_\*  을 빌드시 주입할 수 있었다. (기존에는 undefined)&#x20;

<figure><img src="../.gitbook/assets/image (14).png" alt=""><figcaption></figcaption></figure>



### 🔥 **전체 플로우 요약**

1️⃣ **Elastic Beanstalk에 Docker 배포**\
2️⃣ **CloudFront 배포 생성 및 오리진으로 Beanstalk 연결**\
3️⃣ **AWS ACM을 통해 SSL 인증서 발급 및 HTTPS 연결**\
4️⃣ **Route 53으로 CloudFront에 커스텀 도메인 연결**\
5️⃣ **HTTP → HTTPS 리다이렉트 설정**\
6️⃣ **캐싱 전략 (정적 파일 장기 캐싱, API no-cache) 설정**\
\
\
**CloudFront와 Elastic Beanstalk 연결 개요**

* **Elastic Beanstalk (EC2 인스턴스) → CloudFront (CDN) → 사용자**
* CloudFront는 글로벌 CDN(Content Delivery Network) 역할을 합니다.\
  **정적 파일**(JS, CSS, 이미지 등)을 더 빠르게 제공하고,\
  **HTTPS 및 캐싱**을 통해 사용자 경험을 향상시킵니다.

***

### 📋 **전체 플로우**

1️⃣ **Elastic Beanstalk에 앱 배포 (Docker 이미지 포함)**\
2️⃣ **CloudFront 배포 생성 및 Beanstalk 연결**\
3️⃣ **CloudFront 배포의 오리진(Origin) 구성**\
4️⃣ **SSL/TLS HTTPS 인증서 연결 (AWS Certificate Manager, ACM)**\
5️⃣ **캐싱 전략 및 경로 별 동작 설정**\
6️⃣ **도메인 연결 및 HTTPS 리다이렉트 설정**

***

#### 1단계: **GitHub Actions 설정**

* **목표**: 코드가 GitHub 리포지토리에 푸시될 때마다 Docker 이미지 빌드 및 Elastic Beanstalk에 자동 배포 설정

**1.1. GitHub Actions 워크플로 설정**

먼저 GitHub 리포지토리에 `.github/workflows` 디렉토리를 만들고, 그 안에 배포를 위한 YAML 파일을 작성해야 합니다. 예를 들어 `deploy.yml` 파일을 작성할 수 있습니다.

```yaml
yamlCopy codename: Deploy to AWS Elastic Beanstalk

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: Set up Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '20'

      - name: Install dependencies
        run: |
          yarn install

      - name: Build Docker image
        run: |
          docker build -t your-docker-image .

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v1

      - name: Push Docker image to Amazon ECR
        run: |
          docker tag your-docker-image:latest ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/your-ecr-repository:latest
          docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/your-ecr-repository:latest

      - name: Deploy to Elastic Beanstalk
        uses: einaregilsson/beanstalk-deploy@v21
        with:
          aws_access_key: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws_secret_key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          application_name: your-app-name
          environment_name: your-environment-name
          version_label: ${{ github.sha }}
          region: ap-northeast-2
          deployment_package: deploy.zip
```

**설명**:

* `docker build`: Docker 이미지를 빌드합니다.
* `aws-actions/amazon-ecr-login`: ECR에 로그인하여 이미지를 푸시할 수 있게 합니다.
* `docker push`: Docker 이미지를 ECR에 푸시합니다.
* `einaregilsson/beanstalk-deploy`: Elastic Beanstalk에 배포합니다.

여기서, `AWS_ACCOUNT_ID`, `AWS_REGION`, `your-docker-image`, `your-ecr-repository` 등을 실제 값으로 교체해야 합니다.

**1.2. GitHub Secrets 설정** GitHub에서 배포를 위해 AWS 키를 사용할 수 있도록 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`를 **GitHub Secrets**에 추가해야 합니다. GitHub의 **Settings > Secrets**로 가서 환경 변수로 추가합니다.

***

#### 2단계: **Dockerfile 작성**

* **목표**: 애플리케이션을 Docker 이미지로 빌드할 수 있도록 `Dockerfile`을 설정합니다.

**2.1. Dockerfile 예시**

```dockerfile
dockerfileCopy code# 베이스 이미지
FROM node:20-alpine

# 작업 디렉토리 설정
WORKDIR /app

# 의존성 파일 복사
COPY package.json yarn.lock ./

# 의존성 설치
RUN yarn install --frozen-lockfile

# 애플리케이션 소스 복사
COPY . .

# 빌드
RUN yarn build

# 서버 실행
CMD ["yarn", "start"]
```

**설명**:

* `FROM node:20-alpine`: Node.js 기반의 Alpine 이미지를 사용합니다.
* `WORKDIR /app`: 컨테이너 안에서 작업할 디렉토리입니다.
* `COPY package.json yarn.lock ./`: 애플리케이션의 의존성 파일을 복사합니다.
* `RUN yarn install`: 의존성 설치 명령입니다.
* `COPY . .`: 애플리케이션의 소스 코드를 복사합니다.
* `CMD ["yarn", "start"]`: 애플리케이션을 실행하는 명령입니다.

<figure><img src="../.gitbook/assets/image (12).png" alt=""><figcaption><p>docker build<br></p></figcaption></figure>

#### 생성된 Docker 이미지 실행

```
// Some code
docker images 


```

### Docker에서 포트 매핑 이해하기

* Docker 컨테이너는 기본적으로 외부와 격리된 환경에서 실행된다. 이는 보안이나 독립적인 환경에서 어플리케이션을 실행하기 위해 설계된 특징인데 사실 우리는 그 격리된 환경안에서 실행 중인 어플리케이션에 외부에서 접근하고 싶을 때가 많다 이럴 때 포트 매핑(port mapping)을 사용한다.

{% hint style="info" %}
```
docker run -p <호스트포트>:<컨테이너포트>
```
{% endhint %}

* 포트 매핑은 docker가 호스트 머신의 포트를  컨테이너 내의 포트에 연결하는 과정이다. 이를 통해 와부에서 들어오는 요청을 컨테이너 안에서 실행중인 서비스로 전달 할 수 있게 된다.
* 호스트 포트는 외부에서 접근하고 싶은 포트이다. 보통은 로컬 머신이나 서버에서 사용된다.&#x20;
* 컨테이너 포트는 Docker 컨테이너 안에서 실행중인 어플리케이션이 바인딩된 포트이다.
* 예를 들어 Next start를 하게 되면 기본포트인 3000 으로  실행된다.  만약 이 앱을 Docker컨테이너에서 실행하고, 외부에서 8080 포트로 접근하고 싶다면 다음과 같이 포트매핑을 할 수 있다.

```

docker run -p 8080:3000 <이미지 이름>
```

* 여기서 호스트 머신에서 접근할 8080은 외부에서 접근하고 싶은 포트 번호이므로 대체로 개발자가 원하는 아무포트나 지정할 수 있다. 하지만 포트 충돌을 고려해야 한다. 이미 8080포트가 다른 프로세스에서 사용중이라면 충돌이 발생한다.

{% hint style="info" %}
예약되어있는 특정 포트들



* **80**: HTTP 웹 서버
* **443**: HTTPS 웹 서버
* **22**: SSH
* **3306**: MySQL
* **5432**: PostgreSQL
{% endhint %}





***

#### 3단계: **Elastic Beanstalk 배포**

* **목표**: Docker 이미지를 Elastic Beanstalk에 배포합니다.

**3.1. Elastic Beanstalk 설정** Elastic Beanstalk 콘솔에서 `Docker` 플랫폼을 선택하고, `Dockerfile`이 있는 프로젝트를 배포할 준비를 합니다.

Elastic Beanstalk 으로의 배포는 GUI와 CLI 방법을 이용할 수 있다.&#x20;

GUI는 elasticbeantalk의 console을 이용하는것이고 zip 파일을 수동으로 업로드 하면 끝난다.&#x20;

하지만 여기서 나는 elasticbeanstalk가 내부적으로 어떻게 이 파일들을 읽는지 몰라 초반에는 꽤 고생했는데, 아무튼 .env.production파일을 포함시켜야 한다. &#x20;

경험상 eb cli를 통해 작업을 하는게 생산성 면에서도 훨씬 빨랐다. &#x20;



**3.2. EB CLI 설치**

&#x20;AWS Elastic Beanstalk CLI(`eb CLI`)를 사용하여 애플리케이션을 배포할 수 있습니다.

```bash
bashCopy code# EB CLI 설치
pip install awsebcli

# EB 환경 설정
eb init
```

이 과정에서 AWS 계정 정보와 Elastic Beanstalk 환경을 설정합니다.

```
eb create my-environment-name --platform "Docker running on 64bit Amazon Linux 2023"
```

명령어를 실행하면 elastic beanstalk에 배포까지 실행하는데, 여기서 나는 eb deploy를 했을 때 구체적으로 어떤 파일들이 어떻게 배포까지 이루어 지는지 백그라운드를 잘 몰랐어서 .env 파일을 찾을 수 없다는 에러를 추적해야 했었다.

{% hint style="info" %}
`eb deploy` 명령어는 **Git을 사용해서 소스를 압축하고 AWS로 전송한다**. 그래서 **.gitignore에 포함된 파일은 기본적으로 AWS EB 배포에 포함되지 않습니다.** 따라서, `.gitignore`에 **`.env.production`이 명시되어 있다면, AWS에 해당 파일이 업로드되지 않는다.**
{% endhint %}

그래서 .eb ignore 파일을 만들어준다.

```
// Some code
# .ebignore 예시
node_modules
.DS_Store
coverage
dist
build

# .env 파일은 무시하지만, .env.production은 허용
.env
!.env.production  # !로 예외 처리

```



{% hint style="info" %}
**중요 포인트**:

* `!.env.production` 이 구문은 **`.env.production`을 예외로 하여 EBS로 전송**합니다.
* `.gitignore`에 `.env.production`이 있어도, **.ebignore이 우선** 적용됩니다.
{% endhint %}

* **빠르게 배포해야 하는 경우**: `eb deploy --staged` 추천!
* **staged 파일만 포함**되므로, **현재 작업 중인 변경사항은 배포되지 않음**.
* **커밋 없이도 배포 가능**

배포 성공시 결과&#x20;

<figure><img src="../.gitbook/assets/image (15).png" alt=""><figcaption></figcaption></figure>

배포 실패 후 해결 방법:

1. **Elastic Beanstalk Logs**를 확인하여 오류 메시지 파악
2. `eb status`, `eb logs` 명령어로 환경 상태와 로그 확인
3. Dockerfile, 애플리케이션 코드, 환경 변수 등 설정을 점검
4. 배포가 실패한 원인을 수정한 후, `eb deploy`로 재배포
5. 롤백이 필요하면 이전 버전으로 롤백

***

### 🔥 **2️⃣ CloudFront 배포 생성 및 Beanstalk 연결**

> **Elastic Beanstalk URL을 CloudFront의 오리진으로 연결**합니다.

#### 🛠️ **1. CloudFront 배포 생성**

1. **AWS Management Console → CloudFront → 배포 생성**
2. **오리진 도메인 이름에 Elastic Beanstalk URL 입력**
   * 예: `http://my-app-env.eba-abc123.ap-northeast-2.elasticbeanstalk.com`
   * **HTTP → HTTPS로 변경**해 주세요.
3. **캐싱 정책 선택**
   * 기본 캐싱 정책을 사용할 수 있지만, 필요에 따라 동적 콘텐츠에 맞게 설정할 수 있습니다.

***

### 🔥 **3️⃣ CloudFront의 오리진(Origin) 구성**

1. **Origin 설정**
   * 오리진 도메인: **Elastic Beanstalk URL 입력**\
     (예: `my-app-env.eba-abc123.ap-northeast-2.elasticbeanstalk.com`)
   * 오리진 프로토콜 정책: **HTTPS 전용**
   * 연결 ID: `my-app-origin`
2. **경로 기반 규칙 추가**
   * `/static/*`, `/images/*` 등 **정적 파일 경로에 대한 별도의 캐싱 정책을 설정**할 수 있습니다.
   * 정적 파일은 장기 캐시를 적용하고, API 요청은 **no-cache**로 설정하는 것이 일반적입니다.
3. **캐싱 정책**
   * 동적 콘텐츠: **캐싱 비활성화 (최대 TTL = 0)**
   * 정적 콘텐츠 (JS, CSS, 이미지): **장기 캐시 적용** (최대 TTL = 1일 \~ 7일)
4. **HTTPS 연결**
   * \*\*AWS ACM (Certificate Manager)\*\*에서 HTTPS 인증서를 발급받아 연결해야 합니다.

***

### 🔥 **4️⃣ SSL/TLS HTTPS 인증서 연결 (AWS Certificate Manager, ACM)**

1. **ACM 인증서 생성**
   * **AWS Certificate Manager (ACM)로 인증서 생성**
   * **도메인 이름 입력 (예:** [**www.my-app.com**](http://www.my-app.com)**)**
   * **DNS 확인 방식 선택**
   * 발급된 CNAME 레코드를 **Route 53 (또는 외부 DNS 관리자)에서 추가**합니다.
   * 인증서가 발급될 때까지 기다립니다 (10분\~1시간).
2. **CloudFront에 인증서 연결**
   * CloudFront의 **SSL/TLS 설정**에서 **Custom SSL Certificate**을 선택합니다.
   * 방금 생성한 **ACM 인증서**를 선택합니다.
3. **HTTPS 리다이렉션**
   * 모든 HTTP 트래픽을 **HTTPS로 리다이렉트**합니다.
   * **AWS CloudFront의 동작 규칙 (Behavior)에서 설정**할 수 있습니다.

***

### 🔥 **5️⃣ 캐싱 전략 및 경로 별 동작 설정**

> 경로 기반으로 **캐싱 정책**을 다르게 설정하는 것이 핵심입니다.

1. **경로 별 동작(Behavior) 추가**
   * \*_/static/_ 경로에 대한 캐싱 정책\*\*을 추가합니다.
   * **API 요청**의 경우, no-cache 설정을 합니다.
2. **설정 예시**
   * `/static/*` → **장기 캐시 (1일\~7일)**
   * `/api/*` → **no-cache** (API는 즉시 새 요청으로 보냄)
   * `/` (기본 경로) → **짧은 캐싱 (1분\~5분)**

***

### 🔥 **6️⃣ 도메인 연결 및 HTTPS 리다이렉트**

> CloudFront에서 사용자에게 \*\*커스텀 도메인 (예: www.my-app.com)\*\*을 제공하려면 DNS 설정이 필요합니다.

1. **Route 53 (또는 외부 DNS 관리자) 연결**
   * **A 레코드 (ALIAS) 설정**:
     * 레코드 이름: **my-app.com** 또는 [**www.my-app.com**](http://www.my-app.com)
     * 레코드 유형: **A (Alias)**
     * 값: **CloudFront 배포 ID 연결**
2. **HTTPS 리다이렉션**
   * AWS S3를 사용해 리다이렉트 페이지를 설정하거나,\
     CloudFront **동작 규칙 (Behavior) 설정**에서 **HTTP를 HTTPS로 리다이렉트**합니다.

***

### 📘 **CloudFront 설정 예시**

| **항목**      | **설정값**                                                     |
| ----------- | ----------------------------------------------------------- |
| 오리진 URL     | `my-app-env.eba-abc123.ap-northeast-2.elasticbeanstalk.com` |
| 오리진 프로토콜 정책 | HTTPS Only                                                  |
| 오리진 연결 시간   | 30초 (기본값)                                                   |
| 캐싱 정책       | 기본 캐싱 정책, 경로 기반 캐싱 (`/static/*`)                            |
| 오리진 경로      | `/` (기본값)                                                   |
| 경로 규칙       | `/static/*` - 장기 캐싱, `/api/*` - no-cache                    |
| SSL 인증서     | ACM으로 발급받은 **my-app.com**                                   |
| HTTPS 리다이렉트 | **HTTP를 HTTPS로 리다이렉트**                                      |





***



### EB Command 다루기

```
// Some code//문제가 생기면 log보기
eb logs

// 배포 하기 
eb deploy

// staged 파일만 EB에 배포
eb deploy --staged

```

Elastic Beanstalk의 `eb deploy` 명령어는 **현재 디렉터리의 파일들을 ZIP으로 묶어 배포**합니다.\
이 ZIP 파일에 **Git에 남아 있는 삭제된 파일**이 포함될 수 있습니다.

> ❗ **중요한 점**\
> Git에서 파일을 삭제했다고 해도 **commit을 하지 않거나 Git의 캐시에 남아 있는 경우**, 이 파일이 여전히 ZIP에 포함될 수 있습니다.

### **`eb deploy --staged`란?**

* **Git에 staged 된 파일만** AWS Elastic Beanstalk(EB)로 배포합니다.
* **Git의 HEAD 상태와는 무관**하며, commit 하지 않아도 배포 가능.
* **커밋할 필요 없이 빠르게 배포**할 때 유용합니다.
