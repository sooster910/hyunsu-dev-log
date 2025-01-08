---
hidden: true
---

# Nextjs와 Elasticbeanstalk를 1년간 운영하면서 배운점

### 혼자서 온전히 기획부터, 개발, 운영까지 한 경험은 처음이었다.

*

### Elastic Beanstalk과 Next.js 통합 시 주의점

{% hint style="info" %}
TR:DL

* **서버와 클라이언트 분리**: 서버 측 환경 변수는 Elastic Beanstalk에 설정하고, 클라이언트 측 변수는 빌드 시점에 Next.js에서 처리해야 한다.
* **배포 시점 처리**: Next.js 앱을 빌드할 때 필요한 클라이언트 측 변수는 CI/CD 도구(Jenkins 등)에 설정하고, 서버 측 변수는 Elastic Beanstalk에 설정해야 합니다
{% endhint %}



```bash
# Next.js의 NEXT_PUBLIC_ 환경변수
- 빌드 시점에 코드에 포함됨
- 빌드 후에는 변경 불가능

# Elastic Beanstalk 환경 변수
- 서버 런타임에서 process.env로 접근
- 클라이언트 코드에서는 접근 불가
```

따라서:

```bash
# GitHub Actions에서 빌드할 때
NEXT_PUBLIC_BACKEND_URI 필요
-> 클라이언트 코드에 포함됨

# Elastic Beanstalk 환경 변수
- 서버사이드 코드에서만 사용 가능
- NEXT_PUBLIC_ 변수는 이미 늦음
```

이것이 GitHub Actions의 빌드 시점에 환경변수를 설정해야 하는 이유 이다.

* **빌드 시 환경 변수 적용**
  * `env:`에 **NEXT\_PUBLIC\_으로 시작하는 변수를 추가**합니다.
  * `NEXT_PUBLIC_API_URL: ${{ secrets.NEXT_PUBLIC_API_URL }}`\
    이 명령어로, Next.js 빌드 과정에서 해당 변수가 인식됩니다.\
    이를 통해 **클라이언트 코드에서도 NEXT\_PUBLIC\_ 변수를 사용할 수 있습니다.**
* **런타임 환경 변수 설정**
  * `eb setenv` 명령어로 EB의 런타임 환경 변수를 설정합니다.
  * 이 변수들은 Next.js 서버 측 코드 (`getServerSideProps`, API 경로)에서는 접근할 수 있습니다.
  * 하지만, **빌드 후에 설정된 EB 환경 변수는 클라이언트 코드에 반영되지 않습니다**

### 빌드 시점에 환경변수 주입시 관리 포인트 줄이기&#x20;

아직까지 .env 파일들의 관리만으로 하는 방법은 찾지 못했다.

1. GitAction을 이용하니 GitHub Secrets 사용

GitHub Secrets를 추가로 사용하면 환경 변수 관리 포인트가 늘어나 복잡성이 증가할 수 있습니다. 이미 .env 파일과 Elastic Beanstalk 콘솔에서 환경 변수를 관리하고 있기 때문.



2. next.config.js 파일 활용:

조금 못마땅하긴하다 이것도 관리 포인트가 .local 파일, next.config.js그리고 elasticbeantalk콘솔 환경변수까지 3가지 이기 때문이다.

3. eb setenv&#x20;

`eb setenv` 명령어는 Elastic Beanstalk 환경에 **환경변수**를 설정하는 명령어이다. 기본적으로 이 명령어는 **개별 환경변수**를 하나씩 설정하는 방식으로 사용되는데  **파일로 환경변수를 미리 만들어 놓고 이를 한 번에 적용**할 수 있는 방법을 적용할 수도 있다.







### 무중단 배포 그린-블루 환경 스왑시 dns, cloud front 는 자동으로 스왑이 되지 않으니, 이것은 수동으로 미리 설정해두어야 한다.

### EB CLI를 적극 활용해 배포 후 이상시 logs를 바로 확인해 빠른 대응을 한다. ( console에서는 뭐든지 느림)



### Log 에 모든 답이 있다.&#x20;

\


이 경우에는 **Route 53 → CloudFront → Elastic Beanstalk**의 구조로 트래픽이 흐르게 됩니다.

***

#### 🔥 **CloudFront와 함께 Elastic Beanstalk 연결하기**

**1️⃣ CloudFront 배포 생성**

1. **CloudFront 배포 생성**
   * AWS Management Console에서 **CloudFront**로 이동합니다.
   * **Create Distribution** 클릭 후 **Web** 배포를 선택합니다.
2. **원본(Origin) 설정**
   * **Origin Domain Name**: Elastic Beanstalk의 URL을 입력합니다. (예: `xxxxxx.us-west-2.elasticbeanstalk.com`)
   * **Origin Protocol Policy**: HTTPS 만 허용하는 것이 좋습니다.
3. **캐시 정책 및 동작 설정**
   * **Viewer Protocol Policy**: HTTPS로 리디렉션 (Redirect HTTP to HTTPS)
   * **Allowed HTTP Methods**: 필요한 HTTP 메서드(GET, POST 등)만 허용합니다.
   * **Cache Policy**: 필요에 따라 기본 정책을 사용하거나, 커스텀 정책을 만듭니다.
4. **배포 생성**
   * **Create Distribution**을 클릭합니다.
   * 배포에는 10\~15분 정도 소요됩니다.

***

#### 🔥 **2️⃣ Route 53에 CloudFront 연결**

1. **Route 53 레코드 수정**
   * AWS Route 53으로 이동합니다.
   * **Hosted Zone**에 들어가서 `development.aiff.co.kr` 레코드를 편집합니다.
   * \*\*A 레코드 (Alias)\*\*를 생성합니다.
     * **Name**: `development`
     * **Type**: A (Alias)
     * **Alias Target**: 방금 만든 CloudFront 배포의 **Domain Name**을 입력합니다. (예: `xxxxxxx.cloudfront.net`)
2. **TTL(Time To Live) 설정**
   * 기본값(300초)을 사용하거나, 더 빠른 반영이 필요하면 60초로 설정할 수 있습니다.

***

#### 🔥 **3️⃣ HTTPS 인증서 연결**

1. **ACM에서 인증서 요청**
   * AWS ACM으로 이동하여 **Request Certificate**를 클릭합니다.
   * **domain name**으로 `development.aiff.co.kr`을 입력하고 인증서를 요청합니다.
2. **CloudFront에 HTTPS 인증서 연결**
   * CloudFront 배포 설정으로 이동합니다.
   * **Edit**를 클릭하고, **Custom SSL Certificate** 항목에 AWS ACM 인증서에서 발급된 인증서를 연결합니다.
   * **Minimum Protocol Version**을 TLS 1.2로 설정하여 보안을 강화합니다.

***

#### 📘 **전체 흐름**

**이제 전체 흐름을 한 번 살펴보자.**

1. 사용자가 **development.aiff.co.kr**에 접속
2. DNS(Route 53)에서 **CloudFront**로 요청을 전달
3. **CloudFront**는 콘텐츠가 캐시에 있는 경우 이를 반환하고, 없으면 **Elastic Beanstalk**에 요청
4. **Elastic Beanstalk**에서 콘텐츠를 응답

***

#### 🔥 **마무리 체크리스트**

* [ ] **CloudFront 배포 완료** (배포가 완료되었는지 확인)
* [ ] **Route 53에서 A 레코드(Alias) 추가** (CloudFront의 URL로 연결)
* [ ] **ACM SSL 인증서 연결** (HTTPS 연결을 위해)
* [ ] **CloudFront 캐싱 정책 확인** (필요에 따라 캐시 갱신 설정)

***

#### 📢 **자주 발생하는 문제**

1. **DNS 전파가 느림**\
   → `nslookup development.aiff.co.kr` 명령어로 확인할 수 있습니다.
2. **CloudFront 캐시 문제**\
   → 캐시에 오래된 버전의 파일이 남아 있는 경우 \*\*CloudFront 캐시 무효화(Invalidation)\*\*를 실행해야 합니다.
3. **HTTPS 연결 문제**\
   → Route 53이 **CloudFront의 HTTPS 인증서**를 인식하지 못할 수 있습니다.\
   → CloudFront에서 인증서를 다시 연결하거나, CloudFront에서 **Custom SSL Certificate** 설정을 확인하세요.

***

이 과정을 따라가면 **development.aiff.co.kr**에 웹페이지가 정상적으로 표시될 것입니다. 과정 중에 특정 단계에서 막히면 알려주세요. 🚀

**You said:**

그럼 총 시간은 얼마나 소요될까? 오늘의 계획에 넣으려고 해

**ChatGPT said:**

ChatGPT

Elastic Beanstalk과 CloudFront를 연결하여 `development.aiff.co.kr`에 배포하는 데 걸리는 **예상 소요 시간**은 약 **2\~4시간** 정도입니다.

***

#### **🔍 세부 시간 분배**

| **작업 단계**                    | **소요 시간 (예상)** | **설명**                              |
| ---------------------------- | -------------- | ----------------------------------- |
| **1️⃣ Elastic Beanstalk 배포** | 30분 \~ 1시간     | 코드 빌드 및 배포 대기 시간 (기존에 배포된 경우 생략 가능) |
| **2️⃣ CloudFront 배포 생성**     | 10 \~ 20분      | CloudFront 배포 생성 및 설정 작업            |
| **3️⃣ Route 53 연결 (DNS 전파)** | 30분 \~ 1시간     | Route 53의 A 레코드 연결 및 전파 대기          |
| **4️⃣ HTTPS 인증서 생성 (ACM)**   | 20 \~ 40분      | 인증서 요청, 이메일 인증 및 ACM에 적용            |
| **5️⃣ CloudFront 캐시 초기화**    | 10 \~ 30분      | 캐시 무효화(Invalidation) 작업 (필요시)       |
| **✅ 전체 예상 소요 시간**            | **2 \~ 4시간**   | 단계별 작업 및 대기 시간 포함                   |

***

#### 📘 **시간 절약 팁**

1. **동시 작업**
   * **CloudFront 배포**와 **ACM 인증서 요청**을 동시에 진행하면 시간을 절약할 수 있습니다.
2. **DNS 전파 기다리는 동안 다른 작업**
   * DNS 전파에 시간이 걸리므로 이 시간에 **SSL 인증서 연결 작업**이나 **CloudFront 캐시 무효화**를 진행하면 효율적입니다.
3. **CloudFront 캐시 초기화**
   * 캐시 초기화(Invalidation)를 미리 예약해두면 **최대 30분 절약**할 수 있습니다.

***

#### 📅 **오늘의 작업 일정 예시**

| **시간**    | **작업 내용**                             |
| --------- | ------------------------------------- |
| **09:00** | Elastic Beanstalk 배포 시작 (대기 중)        |
| **09:30** | CloudFront 배포 생성 및 설정                 |
| **10:00** | Route 53 연결 및 DNS 전파 시작               |
| **10:30** | ACM 인증서 요청 (이메일 확인)                   |
| **11:00** | CloudFront에 인증서 연결 및 HTTPS 활성화        |
| **11:30** | 캐시 무효화(Invalidation) 요청               |
| **12:00** | 최종 테스트 (development.aiff.co.kr) 접속 확인 |

***

#### 💡 **결론**

* 최소 **2시간**, 최대 **4시간**을 잡고 작업을 계획하는 것이 좋습니다.
* **DNS 전파 시간**과 **CloudFront 배포 대기 시간**이 변수이므로, 이 시간에 다른 작업을 병행하는 것이 효율적입니다.

&#x20;

### &#x20;
