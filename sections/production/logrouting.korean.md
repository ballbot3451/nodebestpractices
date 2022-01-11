# 애플리케이션 코드가 로그 라우팅을 핸들링 해서는 안된다.

<br/><br/>

### 한 문단 요약

애플리케이션의 코드는 로그 라우팅을 처리해서는 안되고, 대신 로거 유틸리티를 사용하여 `stdout/stderr`에 기록해야 한다.
'로그 라우팅'이란 로그를 선택하여 자신의 애플리케이션이나 애플리케이션 프로세스 이외에 다른 위치로 옮기는 것을 의미한다.
예를 들자면, 로그를 파일이나 데이터베이스 등에 쓰는 것이다. 이렇게 하는 이유는 크게 두 가지로 볼 수 있는데, 첫 번째는 관심사의 분리, 두 번째는 [12-최신 애플리케이션에 대한 모범 사례 분석](https://12factor.net/logs)이다.


우리는 종종 '관심사 분리'를 서비스 내부 코드 조각과 서비스 자체의 관점 사이에서 생각하곤 한다. 하지만 이것은 보다 '구조적인' 컴포넌트에도 적용된다.
우리의 애플리케이션 코드는 인프라나 실행 환경(오늘날의 컨테이너)이 핸들해야하는 것들을 대신 핸들해서는 안된다. 
만약 로그의 위치를 애플리케이션에 정의해버리면 어떻게 될까? 그리고 이후에 그 위치를 수정해야한다면? 결론적으로 코드가 변경되고 배포된다.

컨테이너 기반/클라우드 기반 플랫폼 환경에서 작업 하는 경우, 요구사항에 맞는 성능으로 확장 할 때 컨테이너가 재부팅되고 꺼질 수 있으므로 로그파일이 어디서 끝날지를 확신 할 수 없다.
따라서 실행 환경(컨테이너)이 로그 파일이 라우팅 될 위치를 결정해야만한다. 애플리케이션은 단순히 로깅 해야할 것들을 `stdout` / `stderr`에 기록하고, 실행 환경은 그것들을 로그 스트림으로 가져와서 저장(이동) 해아하는 위치에 라우팅 하도록 구성해야 한다. 또한, 로그파일의 위치를 지정 및 변경하는 팀원은 애플리케이션 개발자가 아니라 DevOps 엔지니어의 일이기도 하다. 그들은 애플리케이션 코드와 친하지 않을 수 있지만 오히려 그들이 쉽게 변화를 만드는것을 막을 수 있다.

<br/><br/>

### 예제 코드 - 안티 패턴: 로그 라우팅이 밀접하게 애플리케이션과 붙어 있음

```javascript
const { createLogger, transports, winston } = require('winston');
/**
   * Requiring `winston-mongodb` will expose
   * `winston.transports.MongoDB`
   */
require('winston-mongodb');
 
// log to two different files, which the application now must be concerned with
const logger = createLogger({
  transports: [
    new transports.File({ filename: 'combined.log' }),
  ],
  exceptionHandlers: [
    new transports.File({ filename: 'exceptions.log' })
  ]
});
 
// log to MongoDB, which the application now must be concerned with
winston.add(winston.transports.MongoDB, options);
```
이렇게 구성하면, 애플리케이션은 비지니스 로직화 로그 라우팅 로직을 함께 핸들링한다!

<br/><br/>

### 예제 코드 - 더 나은 핸들링 + 도커 예제

애플리케이션 코드:
```javascript
const logger = new winston.Logger({
  level: 'info',
  transports: [
    new (winston.transports.Console)()
  ]
});

logger.log('info', 'Test Log Message with some parameter %s', 'some parameter', { anything: 'This is metadata' });
```
도커 컨테이너: `daemon.json`:
```json5
{
  "log-driver": "splunk", // Splunk를 예제로 사용했지만, 어떤 스토리지 타입이던 상관 없다.
  "log-opts": {
    "splunk-token": "",
    "splunk-url": "",
    //...
  }
}
```

이 예제는 다음과 같게 된다 `log -> stdout -> Docker container -> Splunk` 

<br/><br/>

### 블로그 인용: "O'Reilly"

[O'Reilly blog](https://www.oreilly.com/ideas/a-cloud-native-approach-to-logs)

> 만약 고정된 수의 서버와 고정된 수의 인스턴스가 있다면, 로그를 디스크에 저장하는게 좋습니다. 애플리케이션의 인스턴스 수가 유동적이고 그 인스턴스가 어디서 실행되는지 모르는 경우에는 클라우드 공급자가 대신 이러한 로그들을 집계해야 합니다.

<br/><br/>

### 인용: "12-Factor"

[12-Factor 로깅 모범 사례](https://12factor.net/logs)
 > 12-Factor App 은 출력 스트림의 라우팅이나 저장소와는 전혀 관련이 없습니다. 또한 로그 파일을 관리하려고 하거나 쓰려고 시도해서는 안됩니다. 대신 실행중인 각각의 프로세스는 버퍼링 되지 않은, 이벤트 스트림을 stdout 에 기록합니다.
 
> 스테이징 또는 프로덕션 배포에서, 각가의 프로세스의 스트림은 실행 환경에 의해 캡쳐되어 앱의 다른 모든 스트림과 함께 수집되고, 로그를 보거나 장기간 보관을 위해 하나 이상의 최종 목적지로 라우팅 됩니다. 이러한 저장소적 성격의 목적지는 앱에 표시되거나 앱에 의해 구성 될 수 없으며, 대신 실행 환경에 의해 완전히 관리됩니다.

<br/><br/>

 ### 예제: 도커와 Splunk를 사용한 아키텍처 개요

![alt text](../../assets/images/logging-overview.png "Log routing overview")

<br/><br/>
