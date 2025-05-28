# 인프라 관점의 성능 최적화

## 1️⃣ 기본 과제

<img width="1141" alt="image" src="https://github.com/user-attachments/assets/6c2a420d-1ab7-40e1-a91a-66f74b588c7c" />

### 주요 링크

- S3 버킷 웹사이트 엔드포인트: ______
- CloudFrount 배포 도메인 이름: ______

### 주요 개념

#### GitHub Actions과 CI/CD 도구

- GitHub Actions는 코드를 push할 때 자동으로 빌드하고, S3에 정적 파일을 배포하며, 필요 시 CloudFront 캐시 무효화까지 실행하는 자동화된 배포 도구(CI/CD)입니다. `📁.github/workflows/deployment.yml` 내용을 확인해보면...
  - **checkout**: 레포지토리 코드 받아오기
  - **build**: 정적 파일로 변환 (next.config.ts에서 `output: "export"` 설정)
  - **aws s3 sync**: 빌드 결과를 S3 버킷에 업로드
  - **cloudfront create-invalidation**: 변경된 경로의 캐시 무효화 

#### S3와 스토리지

S3는 정적 웹사이트(HTML, CSS, JS 등) 파일을 저장하고 서비스할 수 있는 스토리지입니다.
S3만으로도 웹사이트 배포 가능하지만, 보안상 퍼블릭 접근을 막는 것이 일반적입니다. 대신 CloudFront를 통해 접근을 허용하고, S3는 CloudFront만 접근 가능하도록 설정하는 구조(OAI/OAC)를 씁니다.

- OAI (Origin Access Identity) 와 OAC (Origin Access Control)
  - CloudFront가 S3에 안전하게 접근하도록 하는 메커니즘
  - OAI는 가상 사용자 방식, OAC는 IAM 정책 기반 방식으로 더 세밀한 권한 제어 가능
  - 실무에선 S3 버킷을 퍼블릭으로 열지 않고, CloudFront만 접근 가능하도록 설정함

#### CloudFront와 CDN

CloudFront는 AWS의 글로벌 CDN(Content Delivery Network) 서비스로, 전 세계 엣지 로케이션을 통해 정적 파일을 빠르게 제공하고 S3 접근을 대리합니다.
일반적으로 사용자는 CloudFront의 도메인을 통해 웹사이트에 접속합니다.

#### 캐시 무효화(Cache Invalidation)

CloudFront는 성능을 위해 파일을 캐시하지만, 콘텐츠가 바뀌면 즉시 반영되지 않기 때문에 캐시 무효화가 필요합니다.
aws cloudfront create-invalidation 명령어를 통해 특정 경로나 전체(/*)를 무효화할 수 있습니다.
또는 파일 이름에 해시를 붙여 URL을 변경하는 방식(Cache Busting)을 사용하면 무효화 없이도 새 파일을 배포할 수 있습니다.

#### Repository secret과 환경변수

GitHub Actions에서 AWS 배포를 하기 위해선 `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `DISTRIBUTION_ID` 등의 민감한 값을 GitHub의 Repository Secret에 저장해 env:로 불러와야 합니다.




