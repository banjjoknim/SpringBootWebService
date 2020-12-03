# Chapter9. 코드가 푸시되면 자동으로 배포해 보자 - Travis CI 배포 자동화
24시간 365일 운영되는 서비스에서 배포 환경 구축은 필수 과제 중 하나입니다. 여러 개발자의 코드가 **실시간으로** 병합되고, 테스트가 수행되는 환경, `master` 브랜치가 푸시되면 배포가 자동으로 이루어지는 환경을 구착하지 않으면 실수할 여지가 너무나 많습니다. 이번에는 이러한 배포 환경을 구성해 보겠습니다.

---

## 9.1 CI & CD 소개
앞에서 스크립트를 개발자가 직접 실행함으로써 발생하는 불편을 경험했습니다. 그래서 `CI`, `CD` 환경을 구축하여 이 과정을 개선하려고 합니다.

`CI`와 `CD`란 무엇일까? 코드 버전 관리를 하는 `VCS 시스템(Git, SVN 등)`에 `PUSH`가 되면 자동으로 테스트와 빌드가 수행되어 **안정적인 배포 파일을 만드는 과정**을 `CI(Continuous Integration - 지속적 통합)`라고 하며, 이 빌드 결과를 자동으로 운영 서버에 무중단 배포까지 진행되는 과정을 `CD(Continuous Deployment - 지속적인 배포)`라고 합니다.

여기서 주의할 점은 단순히 **CI 도구를 도입했다고 해서 CI를 하고 있는 것은 아닙니다.** [마틴 파울러의 블로그](http://bit.ly/2Yv0vFp)를 참고해 보면 `CI`에 대해 다음과 같은 4가지 규칙을 이야기합니다.

- 모든 소스 코드가 살아 있고(현재 실행되고) 누구든 현재의 소스에 접근할 수 있는 단일 지점을 유지할 것
- 빌드 프로세스를 자동화해서 누구든 소스로부터 시스템을 빌드하는 단일 명령어를 사용할 수 있게 할 것
- 테스팅을 자동화해서 단일 명령어로 언제든지 시스템에 대한 건전한 테스트 수트를 실핼할 수 있게 할 것
- 누구나 현재 실행 파일을 얻으면 지금까지 가장 완전한 실행 파일을 얻었다는 확신을 하게 할 것

여기서 특히나 중요한 것은 **테스팅 자동화**입니다. 지속적으로 통합하기 위해서는 무엇보다 이 프로젝트가 **완전한 상태임을 보장**하기 위해 테스트 코드가 구현되어 있어야만 합니다.

- 추천 강의 : [백명석님의 클린코더스 - TDD편](http://bit.ly/2xtKinX)

---

## 9.2 Travis CI 연동하기
`Travis CI`는 깃허브에서 제공하는 무료 `CI` 서비스입니다. 젠킨스와 같은 `CI` 도구도 있지만, 젠킨스는 설치형이기 때문에 이를 위한 `EC2` 인스턴스가 하나 더 필요합니다. 이제 시작하는 서비스에서 배포를 위한 `EC2` 인스턴스는 부담스럽기 때문에 오픈소스 웹 서비스인 `Travis CI`를 사용하겠습니다.

>`AWS`에서 `Travis CI`와 같은 `CI` 도구로 `CodeBuild`를 제공합니다. 하지만 빌드 시간만큼 요금이 부과되는 구조라 초기에 사용하기는 부담스럽습니다. 실제 서비스되는 `EC2`, `RDS`, `S3` 외에는 비용 부분을 최소화하는 것이 좋습니다.

### Travis CI 웹 서비스 설정
`https://travis-ci.org/`에서 깃허브 계정으로 로그인을 한 뒤, 오른쪽 위에 `[계정명 -> Settings]`를 클릭합니다.

설정 페이지 아래쪽을 보면 깃허브 저장소 검색창이 있습니다. 여기서 저장소 이름을 입력해서 찾은 다음, 오른쪽의 상태바를 활성화시킵니다(`Legacy Services integration 메뉴`).

활성화한 저장소를 클릭하면 다음과 같이 저장소 빌드 히스토리 페이지로 이동합니다.

![Chapter9_Travis_CI1](https://user-images.githubusercontent.com/68052095/100978996-89e04c00-3586-11eb-811a-1bd8f61d38d8.PNG)

`Travis CI` 웹 사이트에서 설정은 이것이 끝입니다. 상세한 설정은 **프로젝트의 yml 파일로** 진행해야 합니다.

### 프로젝트 설정
`Travis CI`의 상세한 설정은 프로젝트에 존재하는 `.travis.yml` 파일로 할 수 있습니다. `yml` 파일 확장자를 `YAML(야믈)`이라고 합니다. `YAML`은 쉽게 말해서 **JSON에서 괄호를 제거한** 것입니다.

프로젝트의 `build.gradle`과 같은 위치에서 `.travis.yml`을 생성한 후 다음의 코드를 추가합니다.

**.travis.yml**
```java
language: java
jdk:
  - openjdk8

branches: // 1.
  only:
    - master

# Travis CI 서버의 Home
cache: // 2.
  directories:
    - '$HOME/.m2/repository'
    - '$HOME/.gradle'

script: "./gradlew clean build" // 3.

# 실행 완료 시 메일로 알람
notifications: // 4.
  email:
    - recipients: 본인 메일 주소
```
##### 코드설명
**1. branches**
- `Travis CI`를 어느 브랜치가 푸시될 때 수행할지 지정합니다.
- 현재 옵션은 오직 **master 브랜치에 push될 때만** 수행합니다.

**2. cache**
- 그레이들을 통해 의존성을 받게 되면 이를 해당 디렉토리에 캐시하여, **같은 의존성은 다음 배포 때부터 다시 받지 않도록** 설정합니다.

**3. script**
- `master` 브랜치에 푸시되었을 때 수행하는 명령어입니다.
- 여기서는 프로젝트 내부에 둔 `gradlew`을 통해 `clean & build`를 수행합니다.

**4. notifications**
- `Travis CI` 실행 완료 시 자동으로 알람이 가도록 설정합니다.

자 그럼 여기까지 마친 뒤, `master` 브랜치에 커밋과 푸시를 하고, 좀 전의 `Travis CI` 저장소 페이지를 확인합니다.

>###### 학습중 발생 오류 추가
>계속해서 아무 변화가 없길래 여기저기 다 찾아봤지만 해결책을 찾을 수 없었다. 
>그러다가 [Travis](https://travis-ci.org/getting_started)의  2번 항목에서 해결책을 찾을 수 있었다.
>
>**Add a .travis.yml file to your repository**
>
>In order for Travis CI to build your project, you'll need to add a .travis.yml
> configuration file to the root directory of your repository.
>If a .travis.yml is not in your repository, or is not valid YAML, Travis CI will ignore it.
>Here you can find some of our basic language examples.
>
>그러니까 쉽게 말하면, `repository`의 `root` 디렉토리에 `.travis.yml` 파일을 만들어야만 한다는 것이다. 나의 경우에는 프로젝트의 상위에 `TIL`이라는 `root` 디렉토리가 있었기 때문에 계속해서 안된 것이었다...

>###### 학습중 발생 오류 추가
>![Chapter9_Travis_CI_queued](https://user-images.githubusercontent.com/68052095/100988702-57891b80-3593-11eb-919b-3264e5f39f5e.PNG)
>빌드가 되는줄 알았더니... Queued에서 멈춰버렸다.
>[Jobs stuck on "Queued"](https://travis-ci.community/t/jobs-stuck-on-queued/5768)를 참고해보니, 존재하지 않는 환경을 요청하는 경우 발생한다고 한다.
>

빌드가 성공한 것이 확인되면 `.travis.yml`에 등록한 이메일을 확인합니다.

---

## 9.3 Travis CI와 AWS S3 연동하기
