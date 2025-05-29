# 인프라 관점의 성능 최적화

## 1️⃣ 기본 과제

<img width="1141" alt="image" src="https://github.com/user-attachments/assets/6c2a420d-1ab7-40e1-a91a-66f74b588c7c" />

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: ______
- CloudFrount 배포 도메인 이름: ______

## 주요 개념

### GitHub Actions과 CI/CD 도구

- GitHub Actions는 코드를 push할 때 자동으로 빌드하고, S3에 정적 파일을 배포하며, 필요 시 CloudFront 캐시 무효화까지 실행하는 자동화된 배포 도구(CI/CD)입니다. `📁.github/workflows/deployment.yml` 내용을 확인해보면...
  - **checkout**: 레포지토리 코드 받아오기
  - **build**: 정적 파일로 변환 (next.config.ts에서 `output: "export"` 설정)
  - **aws s3 sync**: 빌드 결과를 S3 버킷에 업로드
  - **cloudfront create-invalidation**: 변경된 경로의 캐시 무효화
 
<br/>

### S3와 스토리지

S3는 정적 웹사이트(HTML, CSS, JS 등) 파일을 저장하고 서비스할 수 있는 스토리지입니다.
S3만으로도 웹사이트 배포 가능하지만, 퍼블릭 접근을 막는 것이 일반적입니다. 대신 CloudFront를 통해 접근을 허용하고, S3는 CloudFront만 접근 가능하도록 설정합니다.

#### ❓CloudFront가 꼭 필요할까?
- 실무에서는 보안과 성능 이유로 대부분 S3에 직접 접근하지 못하게 퍼블릭 접근을 차단하고, 대신 CloudFront를 통해 우회하도록 설정함
- 그 과정에서 나오는 개념: OAI (Origin Access Identity) 와 OAC (Origin Access Control)
  - OAI: 기존 방식. CloudFront가 S3 버킷에 접근할 수 있도록 가상의 IAM 사용자(OAI)를 생성하고 S3 버킷에 권한을 부여하는 방식
  - OAC: 신규 방식. CloudFront 자체에 권한을 직접 부여할 수 있는 구조로, 세밀한 IAM 정책 설정이 가능하며 최신 보안 요구사항에 부합함

실제로 CloudFront만 접근할 수 있도록 설정하면, S3는 퍼블릭으로 열 필요가 없어져 보안이 강화됩니다. 퍼블릭 권한을 막는 이유 — 실수로 URL을 알고 있는 누군가가 S3에서 직접 접근해서 파일을 보게 되면, 캐시 우회가 되거나 의도하지 않은 파일 노출로 이어질 수 있기 때문!!

<br/>


### CloudFront와 CDN

CloudFront는 AWS의 글로벌 CDN(Content Delivery Network) 서비스로, 전 세계 엣지 로케이션을 통해 정적 파일을 빠르게 제공하고 S3 접근을 대리합니다.
일반적으로 사용자는 CloudFront의 도메인을 통해 웹사이트에 접속합니다.

#### ❓ CDN은 이미지나 폰트 같은 리소스 전용으로 쓰는 거 아니었나?
- 꼭 그렇지 않다!! 전체 정적 사이트를 CDN 위에 얹는 구조도 매우 일반적임, React나 Next.js의 output: export된 결과물 전체를 CloudFront가 캐시하고 서빙하는 구조 (현재 프로젝트에 해당함)
- 즉, 이 구조는 웹 페이지 전체를 CDN에서 서빙하는 구조이고, 개별 리소스뿐만 아니라 페이지 전체가 캐싱되어 더 빠른 응답이 가능하다!

<br/>


### 캐시 무효화(Cache Invalidation)

CloudFront는 성능을 위해 파일을 캐시하지만, 문제는 이 캐시가 변경 사항을 반영하지 못하면 사용자가 예전 콘텐츠를 보게 된다는 점입니다. 그래서 캐시 무효화(invalidation)가 필요합니다.
→ aws cloudfront create-invalidation 명령어를 통해 특정 경로나 전체(/*)를 무효화할 수 있습니다.

예를 들어 배포 시 새로 바뀐 index.html이 즉시 반영되지 않는 상황이 발생할 수 있습니다.
이럴 때 aws cloudfront create-invalidation 명령어를 실행해서 /index.html 혹은 전체 /* 경로를 무효화할 수 있습니다.
하지만 이 과정은 비용이 발생하고, 무효화 요청 수에 제한도 있으므로 꼭 필요한 경로만 invalidate하는 것이 좋습니다.

#### ❓ Cache Busting 전략
- 변경된 파일에 main.[해시].js처럼 해시값을 붙여 파일명을 바꾸면, CloudFront 입장에선 새 파일로 인식하여 무효화 없이도 새 버전이 배포됨
- Next.js, Vite 같은 빌드 도구는 기본적으로 이 방식을 지원하고 있어서, JS/CSS 등에는 무효화 없이도 캐시 우회가 가능하지만, HTML은 여전히 캐시 무효화 대상이 됨
- 즉, 캐시 무효화와 캐시 버스팅은 상황에 따라 병행되어야 하는 전략임
  - HTML은 무효화로, JS/CSS는 버스팅으로 대응하는 식이 실무에서 자주 보이는 형태라고 함

<br/>


### Repository secret과 환경변수

GitHub Actions에서 AWS 배포를 하기 위해선 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `DISTRIBUTION_ID` 등의 민감한 값을 GitHub의 Repository Secret에 저장해 env:로 불러와야 합니다.


## 2️⃣ 심화 과제


|         S3             |             CloudFronts         |
|------------------------|-----------------------------------|
|<img width="572" alt="스크린샷 2025-05-29 오후 10 23 57" src="https://github.com/user-attachments/assets/b2fa35c9-ea14-437b-984f-4a285c9080ad" />|<img width="578" alt="스크린샷 2025-05-29 오후 10 24 44" src="https://github.com/user-attachments/assets/80870d11-4aa0-4be1-8040-9afaf038faff" />|



