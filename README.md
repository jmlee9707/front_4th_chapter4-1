# 프론트엔드 배포 파이프라인

## 개요

![](https://velog.velcdn.com/images/jmlee9707/post/7a7a7224-e91b-44b9-8c94-86cdcfb49a7a/image.png)

GitHub Actions에 워크플로우를 작성해 다음과 같이 배포가 진행되도록 합니다.

(사전작업: Ubuntu 최신 버전 설치)

1. Checkout 액션을 사용해 코드 내려받기
2. `npm ci` 명령어로 프로젝트 의존성 설치
3. `npm run build` 명령어로 Next.js 프로젝트 빌드
4. AWS 자격 증명 구성
5. 빌드된 파일을 S3 버킷에 동기화
6. CloudFront 캐시 무효화

```yaml
name: Deploy Next.js to S3 and invalidate CloudFront

    on:
      push:    # push 될때 workflow 자동 실행
        branches:
          - main  # 또는 master, 프로젝트의 기본 브랜치 이름에 맞게 조정
      workflow_dispatch:   # 수동 실행 기능, 필요할 때 직접 설정 가능

    jobs:
      deploy:
        runs-on: ubuntu-latest # 어떤 OS 에서 실행할 것인지 명시 (unbuntu 최신 버전)

        steps: # job이 가질 수 있는 동작의 나열, 각 step은 독립적인 프로세스를 ㅏ짐
        - name: Checkout repository
          uses: actions/checkout@v4   # 해당 step에서 사용할 action

        - name: Install dependencies
          run: npm ci

        - name: Build
          run: npm run build

        - name: Configure AWS credentials
          uses: aws-actions/configure-aws-credentials@v1
          with:         # 해당 action에 의해 정의되는 input 파라미터
            aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
            aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
            aws-region: ${{ secrets.AWS_REGION }}

        - name: Deploy to S3
          run: |
            aws s3 sync out/ s3://${{ secrets.S3_BUCKET_NAME }} --delete

        - name: Invalidate CloudFront cache
          run: |
            aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
```

![](https://velog.velcdn.com/images/jmlee9707/post/1154ef81-8599-4ac1-bfab-07c6ae8ab919/image.png)

## 주요 링크

- S3 버킷 웹사이트 엔드포인트: [엔드포인트](http://hanghae99-week9-bucket.s3-website.eu-north-1.amazonaws.com/)
- CloudFrount 배포 도메인 이름: [도메인 이름](https://d3p2prjorrvvlr.cloudfront.net/)

<br />

## 주요 개념

### GitHub Actions과 CI/CD 도구

**CI/CD**

- 작업한 소스 코드를 빌드하고, 저장소에 전달 후 배포까지 하는 과정
- CI (Continuous Integration : 지속적 통합) : 여러 개발자의 코드 변경사항을 정기적으로 통합하는 프로세스 (테스트, 빌드를 포함한 과정)
- CD (Continuous Delivery : 지속적 전달) : 통합된 코드를 실제 운영 환경에 자동으로 배포하는 프로세스
- CI/CD 도구 : Github Actions, Jenkins, TeamCity, Gitlab CI, Azure DevOps ...

**Github Actions**

- Github에서 공식적으로 제공하는 CI/CD tool.
- 즉, 개발의 workflow(전체 프로세스)를 자동화할 수 있게 도와주는 툴.
- YAML 파일을 사용하며 workflow를 구성할 repository에 `.github/workflows/github-actions.yml` 형태로 파일을 생성한다.

---

### S3와 스토리지

**Storage : 스토리지**

- 데이터룰 저장하기 위한 별도의 장소 또는 장치

**Amazon S3 ㅣ Simple Storage Service**

- 아마존 웹 서비스(AWS)가 제공하는 클라우드 **Storage Service.**
- 파일, 데이터 및 다양한 유형의 미디어 등을 저장하고 관리하는 데 사용되는 웹 기반 스토리지 시스템
- 저장되는 데이터들은 "버킷"이라고 불리는 저장 공간에 저장되며, 각 버킷은 전 세계 어느 곳에서나 고유한 이름을 가지고 이를 통해 데이터를 관리하고 접근할 수 있다.

---

### CloudFront와 CDN

![](https://velog.velcdn.com/images/jmlee9707/post/a4cbe14b-65cc-47ae-9e1b-a85e480d44d5/image.png)

**CDN | Content Delivery Network**

- 콘텐츠 전송 네트워크로써 지리, 물리적으로 떨어져 있는 사용자에게 컨텐츠를 더 빠르게 제공하는 시스템
- CDN을 사용하면 웹사이트 로딩 속도가 개선되고, 인터넷 회선 비용이 절감되며 대규모의 분산 서버 장비로 공격 트래픽을 완화시켜 웹사이트의 보안을 개선할 수 있다.

- 작동원리 :

  1. 사용자는 연결하고자 하는 도메인 주소를 Local DNS 서버에 요청
  2. Local DNS 서버는 CDN DNS 서버에게 적합한 Edge 서버의 주소를 요청.
  3. 해당하는 Edge 서버의 주소가 사용자에게 반환
  4. 사용자는 Edge 서버에게 콘텐츠 요청 메시지를 전달. 최초 연결 혹은 캐싱되어 있던 콘텐츠 만료 시, Edge 서버는 원본 서버(Content Provider)에게 필요한 콘텐츠를 요청
  5. 원본 서버는 필요한 콘텐츠를 Edge 서버로 전달하고, Edge 서버는 해당 콘텐츠를 캐시에 저장. (저장 기간은 HTTP 헤더의 TTL(Time-To-Live)에 명시)
  6. Edge 캐시에 저장된 콘텐츠를 최종적으로 사용자에게 전달!

  ![](https://velog.velcdn.com/images/jmlee9707/post/a2bd97a3-d0be-4507-9a38-22af2c1ba972/image.png)

  <br />

**CloudFront**

- AWS에서 제공하는 **CDN 서비스**
- .html, .css, .js 및 이미지 파일과 같은 정적 및 동적 웹 콘텐츠를 사용자에게 더 빨리 배포하도록 지원하는 웹 서비스
- 엣지 로케이션이라고 하는 데이터 센터의 전 세계 네트워크를 통해 콘텐츠를 제공

---

### 캐시 무효화(Cache Invalidation)

**캐시 무효화**

- 캐시에서 데이터를 제거하여 캐시를 무효화하는 프로세스
- 객체가 캐시된 후 일반적으로 객체가 만료되거나 새 콘텐츠를 위한 공간 확보를 위해 삭제될 때까지 캐시에 남아있는데, 정상적인 만료 시간 전에 캐시에서 객체를 삭제해야 하는 경우가 있음. -> 이 경우, 캐시 무효화를 요청하여 캐시가 객체 또는 객체 조합을 무시할 수 있도록 가능
- 또한 캐시 데이터와 원본 저장소의 데이터가 일관성을 유지해야 하기에, 캐시 데이터를 유효성 검증을 통해 캐시 데이터를 무효화 시켜야 함.

**캐시 무효화 방법**

- Time-To-Live (TTL) 방식 : 각 캐시 항목에 유효 기간을 설정하여, 정해진 시간이 지나면 자동으로 캐시가 만료되도록 설정. 데이터 변경 여부와 상관없이 일정 시간마다 새로운 데이터를 불러오게 하여, 자주 업데이트되지 않는 데이터에 적합
- 명시적 무효화 (Manual/Explicit Invalidation) : 관리자가 특정 조건이나 이벤트(예: 배포 후 업데이트)에 따라 캐시된 항목을 직접 삭제하는 방식

  - 그 중 하나로 , AWS CloudFront에서는 aws cloudfront create-invalidation 명령어를 통해 지정된 경로의 캐시 무효화.

  ```yaml
   # --distribution-id: 무효화를 적용할 CloudFront 배포의 ID를 지정
   # --paths "/*": 모든 경로(즉, 모든 파일)를 대상으로 무효화 요청
   - name: Invalidate CloudFront cache
          run: |
            aws cloudfront create-invalidation --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} --paths "/*"
  ```

- 캐시 버스팅 (Cache Busting) : 파일 이름이나 URL에 버전 정보나 해시 값을 포함시켜, 리소스가 변경되면 URL 자체가 달라지도록 하는 방법. 이 경우, 브라우저나 CDN은 새로운 URL에 대해 캐시가 없다고 인식하여 항상 최신 파일을 불러오게 됨.
- 조건부 요청 (Conditional Requests) : 클라이언트가 서버에 HTTP 헤더(예: ETag, Last-Modified)를 사용해 리소스의 변경 여부를 확인한 후, 변경되었을 때만 데이터를 새로 다운로드.

---

### Repository secret과 환경변수:

**환경 변수**

- 컴퓨터에서 프로그램이 실행될 때 동작을 제어하거나 설정하기 위해 사용되는 값들의 모임.

**Repository Secret**

- 민감한 정보를 안전하게 저장하기 위한 기능
- GitHub에서 암호화하여 저장되며, workflow 실행 시에만 접근
- GitHub 리포지토리의 Settings > Secrets에서 추가, 수정, 삭제
- ${{ secrets.SECRET_NAME }} 형식으로 사용
- 외부에 노출되어서는 안되는 데이터 (ex. API key, password ..)를 안전하게 관리

<br />

## CDN과 성능최적화

#### CDN 사용 전 :

[S3(CDN 사용 전) 링크](http://hanghae99-week9-bucket.s3-website.eu-north-1.amazonaws.com/)

![](https://velog.velcdn.com/images/jmlee9707/post/c8ed880e-083c-48e1-8cd2-d37e818fc31f/image.png)
![](https://velog.velcdn.com/images/jmlee9707/post/ed42a002-ddb0-4790-b99a-d3523085022c/image.png)

CDN을 사용하기 전, 최초 document를 받아오는데 약 341ms 의 시간이 소요되었습니다.

#### CDN 사용 후 :

[Cloud Front(CDN 사용 후) 링크](https://d3p2prjorrvvlr.cloudfront.net/)

![](https://velog.velcdn.com/images/jmlee9707/post/d22ae4d5-b464-4b54-bb6e-a08d80a8d90b/image.png)

CloudFront를 사용한 후, 최초 document를 받아오는데 56ms 의 시간이 소요되었습니다.

![](https://velog.velcdn.com/images/jmlee9707/post/96fe0ab6-cabe-4ea6-abc6-0d8bfbd5f429/image.png)

또한 Headers>Content-Encoding틑 통해 파일이 압축되어(3.2kB) 있고,
Headers>X-Cache를 통해 CloudFront에서 콘텐츠를 전달받았음을 알 수 있었습니다.

#### 결과

![](https://velog.velcdn.com/images/jmlee9707/post/9f354530-3d93-4cae-96ae-20987b2d9b09/image.png)

CloudFront + S3를 사용 시, S3 만 적용했을 때 보다 파일 크기도 비교적 작고, 시간도 빨리 단축되었음을 확인할 수 있었습니다.
