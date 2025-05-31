# 인프라 관점의 성능 최적화
정적 사이트를 배포할 때는 단순히 HTML/CSS/JS를 보여주는 것을 넘어서, 빠른 응답 속도, 전 세계 사용자 대상 안정적인 제공, 보안 등을 고려한 인프라 구성이 필요합니다.

# 1️⃣ 기본 과제

<img width="1141" alt="image" src="https://github.com/user-attachments/assets/6c2a420d-1ab7-40e1-a91a-66f74b588c7c" />

# 주요 링크

- S3 버킷 웹사이트 엔드포인트: https://d388ywrs6ccope.cloudfront.net
- CloudFrount 배포 도메인 이름: http://kanghyewon.s3-website.ap-northeast-2.amazonaws.com/

# 주요 개념

## GitHub Actions과 CI/CD 도구

- GitHub Actions는 코드 push할 때 자동으로 빌드하고, S3에 정적 파일을 배포하며, CloudFront 캐시 무효화까지 실행하는 자동화된 배포 도구입니다.
- `📁.github/workflows/deployment.yml` 스크립트는 아래와 같은 순서로 동작합니다:


| 항목                             | 설명                                               |
| ------------------------------ | ------------------------------------------------ |
| checkout                       | GitHub 저장소 코드 받아오기                               |
| build                          | 정적 파일로 변환 (`next.config.ts`의 `output: "export"`) |
| aws s3 sync                    | 정적 파일을 S3 버킷에 업로드                                |
| cloudfront create-invalidation | 변경된 경로의 캐시 무효화                                   |

이 과정은 사람의 개입 없이 자동화된 배포 흐름을 구성할 수 있어, 유지보수성과 협업 효율성이 크게 향상됩니다.


<br/>
<br/>

## S3와 스토리지

S3와 정적 웹 파일 스토리지
S3는 HTML/CSS/JS와 같은 정적 자산을 저장할 수 있는 스토리지 서비스입니다.
기본적으로 퍼블릭하게 열어두면 바로 배포가 가능하지만, 실무에서는 보안과 성능을 위해 보통 CloudFront를 거쳐서만 접근하도록 구성합니다.

### ❓ 퍼블릭 접근을 막는 이유:

- 파일 URL을 아는 외부인이 직접 S3에서 접근할 수 있음
- 캐시 우회로 인한 트래픽 증가 및 최신 버전 미반영 문제 발생
- 의도치 않은 민감한 자산 노출 가능성

이를 해결하기 위해 CloudFront와 함께 OAI (Origin Access Identity) 또는 OAC (Origin Access Control) 설정을 통해 S3 접근을 제어합니다:

| 개념  | 설명                                                    |
| --- | ----------------------------------------------------- |
| OAI | CloudFront가 S3에 접근할 수 있도록 가상의 IAM 사용자(OAI)를 부여        |
| OAC | CloudFront 배포 자체에 IAM 권한을 직접 부여 (더 세밀하고 보안 강화된 최신 방식) |


<br/>
<br/>


## CloudFront와 CDN

CloudFront는 AWS의 글로벌 CDN(Content Delivery Network) 서비스로, 전 세계 엣지 로케이션을 통해 정적 파일을 빠르게 제공하고 S3 접근을 대리합니다.
일반적으로 사용자는 CloudFront의 도메인을 통해 웹사이트에 접속합니다.

### ❓ CDN은 이미지나 폰트 같은 리소스 전용으로 쓰는 거 아니었나?
- 전체 정적 사이트를 CDN 위에 얹는 구조도 매우 일반적
- React나 Next.js의 output: export된 결과물 전체를 CloudFront가 캐시하고 서빙하는 구조 (현재 프로젝트에 해당함)
- 즉, 이 구조는 웹 페이지 전체를 CDN에서 서빙하는 구조이고, 개별 리소스뿐만 아니라 페이지 전체가 캐싱되어 더 빠른 응답이 가능

이렇게 구성하면 다음과 같은 장점이 있습니다
  - 전 세계 사용자에게 동일한 속도로 빠르게 제공 가능
  - S3 퍼블릭 접근 차단 → 보안 강화
  - CloudFront가 캐시를 제공하므로 트래픽 비용 절감

<br/>
<br/>

## 캐시 무효화(Cache Invalidation)

> 무효화는 유료이고 요청 수 제한이 있으므로, 꼭 필요한 경우에만 사용해야 합니다.

CloudFront는 성능을 위해 파일을 캐시하지만, 문제는 이 캐시가 변경 사항을 반영하지 못하면 사용자가 예전 콘텐츠를 보게 된다는 점입니다. 그래서 캐시 무효화(invalidation)가 필요합니다.
→ aws cloudfront create-invalidation 명령어를 통해 특정 경로나 전체(/*)를 무효화할 수 있습니다.

| 전략                     | 설명                         |
| ---------------------- | -------------------------- |
| 전체 무효화 (`/*`)          | 전체 파일을 무효화. 즉각 반영되지만 비용이 큼 |
| 부분 무효화 (`/index.html`) | 최소한의 비용으로 필요한 파일만 무효화 가능   |

### ❓ 캐시 버스팅(Cache Busting) 병행 전략
- 변경된 파일에 main.[해시].js처럼 해시값을 붙여 파일명을 바꾸면, CloudFront 입장에선 새 파일로 인식하여 무효화 없이도 새 버전이 배포됨
- Next.js, Vite 같은 빌드 도구는 기본적으로 이 방식을 지원하고 있어서, JS/CSS 등에는 무효화 없이도 캐시 우회가 가능하지만, HTML은 여전히 캐시 무효화 대상이 됨

| 파일 유형         | 전략     |
| ------------- | ------ |
| HTML          | 캐시 무효화 |
| JS/CSS/Assets | 캐시 버스팅 |

<br/>
<br/>


## Repository secret과 환경변수

- GitHub Actions에서 AWS 리소스에 접근하려면 민감한 키들이 필요합니다.
- 이 값들은 Repository Settings > Secrets에 등록해 안전하게 관리합니다.

| 키 이름                    | 용도                            |
| ----------------------- | ----------------------------- |
| `AWS_ACCESS_KEY_ID`     | IAM 사용자 키                     |
| `AWS_SECRET_ACCESS_KEY` | IAM 사용자 비밀 키                  |
| `DISTRIBUTION_ID`       | CloudFront 배포 ID (캐시 무효화에 사용) |

<br/>
<br/>
<br/>

# 2️⃣ 심화 과제

## CDN과 성능최적화

정적 파일(SVG, JS, CSS, 폰트 등)을 Amazon S3에서 직접 서빙하는 경우,
사용자의 요청은 S3 버킷이 위치한 리전까지 직접 전송되어야 하므로 **지연 시간(latency)**이 발생합니다.
CloudFront는 Amazon에서 제공하는 CDN(Content Delivery Network)으로,
전 세계에 분산된 엣지 로케이션에서 사용자와 가장 가까운 서버가 요청을 처리하게 해줍니다.

|         S3             |             CloudFronts         |
|------------------------|-----------------------------------|
|<img width="572" alt="스크린샷 2025-05-29 오후 10 23 57" src="https://github.com/user-attachments/assets/b2fa35c9-ea14-437b-984f-4a285c9080ad" />|<img width="578" alt="스크린샷 2025-05-29 오후 10 24 44" src="https://github.com/user-attachments/assets/80870d11-4aa0-4be1-8040-9afaf038faff" />|

<br/>

### CloudFront 적용 전과 후: 성능 차이

아래는 동일한 페이지에 대해, CloudFront 적용 전(S3 직접 서빙)과 적용 후의 리소스 응답 시간 비교입니다.

| 리소스 유형       | 예시 파일명             | S3 로딩 시간  | CloudFront 로딩 시간 | 차이        |
| ------------ | ------------------ | --------- | ---------------- | --------- |
| `document`   | `document`         | 102ms     | 11ms             | -91ms     |
| `font`       | Pretendard 폰트 파일   | 46\~68ms  | 11\~12ms         | -35\~57ms |
| `script`     | 번들 JS (169\~175kB) | 74\~120ms | 18\~22ms         | -56\~98ms |
| `stylesheet` | CSS 파일 (2\~3kB)    | 47\~50ms  | 12ms             | -35\~38ms |
| `image`      | SVG, favicon.ico 등 | 25\~30ms  | 11\~13ms         | -12\~17ms |

평균적으로 60~80% 이상의 로딩 시간 개선이 관측됨

<br/>

### 성능 최적화 측면에서의 CDN 장점

| 최적화 항목               | CDN 도입 전               | CDN 도입 후 (CloudFront 기준) |
| -------------------- | ---------------------- | ------------------------ |
| 요청 경로             | S3 리전까지 직접 전송          | 가장 가까운 엣지 로케이션에서 응답      |
| Time to First Byte | 높음 (네트워크 왕복 비용)        | 낮음 (캐시된 리소스 즉시 응답)       |
| 정적 리소스 로딩 속도      | 느림                     | 빠름                       |
| 렌더링 체감 속도          | 느림, 끊김 있음              | 부드럽고 빠름                  |
| 오리진 부하            | 모든 요청 S3로 전달           | 캐시 히트율 증가로 S3 트래픽 감소     |
| 퍼포먼스 지표           | CLS, FCP, TTFB 등 수치 낮음 | 지표 개선, UX 향상             |

<br/>
<br/>
<br/>




