# npm ci 로 패키지를 설치하라

<br/><br/>

### 한 문단 요약

[**의존성 잠금**](./lockdependencies.md) 에 따라 의존성을 잠궜지만, 이러한 패키지들이 실제 Production 환경에서도 똑같이 사용되는지 확실히 해야 합니다.

`npm ci` 사용하여 패키지를 설치하면 그 이상의 효과를 볼 수 있다.
* `package-lock.json` 가 없거나 `package.json`와 다른 경우 패키지 설치는 실패합니다.
* `node_modules`폴더가 이미 있다면 자동으로 제거되고 설치됩니다.
* 거의 2배 이상 빠릅니다. [the release blog post](https://blog.npmjs.org/post/171556855892/introducing-npm-ci-for-faster-more-reliable)

### 언제 이것이 유용한가?
CI 환경 또는 QA 가 소프트웨어를 테스트 할 때 나중에 운영 환경으로 보낼 패키지 버전과 정확히 동일한 버전으로 테스트 될 것임이 보장된다. 
게다가 어떤 이유로 누군가 cli 를 사용하지 않고 수동으로 package.json 을 변경하는 경우, package.json 과 package-lock.json 사이에 차이가 생기고 오류가 발생 될것이다.

### npm의 의견

[npm ci documentation](https://docs.npmjs.com/cli/ci.html)
> 이 명령어는 CI, 테스트 플랫폼, 배포 환경과 같은 자동화된 환경 또는 의존성을 완전히 새로 설치하려는 모든 상황에서 사용 된다는 점을 제외하면, npm-install 명령어와 닮았습니다.

[Blog post announcing the release of `ci` command](https://blog.npmjs.org/post/171556855892/introducing-npm-ci-for-faster-more-reliable)
> 이 명령어는 CI/CD프로세스를 위한 빌드 성능과 안정성을 크게 개선하여 워크플로우에서 CI/CD를 사용하는 개발자에게 일관되고 빠른 경험을 제공합니다.

[npmjs: dependencies and devDepencies](https://docs.npmjs.com/specifying-dependencies-and-devdependencies-in-a-package-json-file)
>    "dependencies": 애플리케이션이 Production 에서 요구하는 패키지
>    "devDependencies": 로컬 개발 또는 테스팅에서만 필요로 하는 패키지

<br/><br/>
