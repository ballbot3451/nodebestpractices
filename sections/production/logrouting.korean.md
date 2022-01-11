# 애플리케이션 코드가 로그 라우팅을 핸들링 해서는 안된다.

<br/><br/>

### 한 문단 요약

애플리케이션의 코드는 로그 라우팅을 처리해서는 안되고, 대신 로거 유틸리티를 사용하여 `stdout/stderr`에 기록해야 한다.
'로그 라우팅'이란 로그를 선택하여 자신의 애플리케이션이나 애플리케이션 프로세스 이외에 다른 위치로 옮기는 것을 의미한다.
예를 들자면, 로그를 파일이나 데이터베이스 등에 쓰는 것이다. 이렇게 하는 이유는 크게 두 가지로 볼 수 있는데, 첫 번째는 관심사의 분리, 두 번째는 [12-최신 애플리케이션에 대한 모범 사례 분석](https://12factor.net/logs)이다.


우리는 종종 '관심사 분리'를 서비스 내부 코드 조각과 서비스 자체의 관점 사이에서 생각하곤 한다. 하지만 이것은 보다 '구조적인' 컴포넌트에도 적용된다.
우리의 애플리케이션 코드는 인프라나 실행 환경(오늘날의 컨테이너)이 핸들해야하는 것들을 대신 핸들해서는 안된다. 
만약 로그의 위치를 애플리케이션에 정의해버리면 어떻게 될까? 그리고 이후에 그 위치를 수정해야한다면? 결론적으로 코드가 변경되고 배포된다.

컨테이너 기반/클라우드 기반 플랫폼 환경에서 작업 하는 경우, 요구사항에 맞는 성능으로 확장 할 때 컨테이너가 재부팅되고 꺼질 수 있다.

so we can't be sure where a logfile will end up. The execution environment (container) should decide where the log files get routed to instead. 
The application should just log what it needs to to `stdout` / `stderr`, and the execution environment should be configured to pick up the log stream 
from there and route it to where it needs to go. 
Also, those on the team who need to specify and/or change the log destinations are often not application developers but are part of DevOps, 
and they might not have familiarity with the application code. This prevents them from easily making changes. 

<br/><br/>

### Code Example – Anti-pattern: Log routing tightly coupled to application

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
Doing it this way, the application now handles both application/business logic AND log routing logic!

<br/><br/>

### Code Example – Better log handling + Docker example
In the application:
```javascript
const logger = new winston.Logger({
  level: 'info',
  transports: [
    new (winston.transports.Console)()
  ]
});

logger.log('info', 'Test Log Message with some parameter %s', 'some parameter', { anything: 'This is metadata' });
```
Then, in the docker container `daemon.json`:
```json5
{
  "log-driver": "splunk", // just using Splunk as an example, it could be another storage type
  "log-opts": {
    "splunk-token": "",
    "splunk-url": "",
    //...
  }
}
```
So this example ends up looking like `log -> stdout -> Docker container -> Splunk`

<br/><br/>

### Blog Quote: "O'Reilly"

From the [O'Reilly blog](https://www.oreilly.com/ideas/a-cloud-native-approach-to-logs),
 > When you have a fixed number of instances on a fixed number of servers, storing logs on disk seems to make sense. However, when your application can dynamically go from 1 running instance to 100, and you have no idea where those instances are running, you need your cloud provider to deal with aggregating those logs on your behalf.

<br/><br/>

### Quote: "12-Factor"

From the [12-Factor best practices for logging](https://12factor.net/logs),
 > A twelve-factor app never concerns itself with routing or storage of its output stream. It should not attempt to write to or manage logfiles. Instead, each running process writes its event stream, unbuffered, to stdout.
 
 > In staging or production deploys, each process’ stream will be captured by the execution environment, collated together with all other streams from the app, and routed to one or more final destinations for viewing and long-term archival. These archival destinations are not visible to or configurable by the app, and instead are completely managed by the execution environment.

<br/><br/>

 ### Example: Architecture overview using Docker and Splunk as an example

![alt text](../../assets/images/logging-overview.png "Log routing overview")

<br/><br/>
