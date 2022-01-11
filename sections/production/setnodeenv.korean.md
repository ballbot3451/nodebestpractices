# NODE_ENV = production 으로 설정하라

<br/><br/>

### 한 문단 요약

&nbsp;&nbsp;프로세스의 환경 변수란, 실행 중인 프로그램이 접근 가능한 키-값의 집합으로 대게 프로그램의 설정을 위해 사용된다. 
어떠한 변수도 사용 가능하지만, Node 는 NODE_ENV 라는 변수로 프로그램이 Production 환경에서 실행되는지 여부를 결정하는것을 권장한다.
이러한 결정을 통해 컴포넌트들이 개발 중에는 더 나은 진단 기능과 캐싱 비활성화, 디버그용 로그 출력 등을 제공 할 수 있다.
요즘 유행하는 배포 도구인 Chef, Puppet, CloudFormation 등 또한 배포 중에 환경 변수를 설정 할 수 있도록 지원한다.

<br/><br/>

### 예제 코드: NODE_ENV 환경 변수를 설정하고 읽어오기

```shell script
// bash 셸에서 node 앱을 실행하기 전에 환경 변수 설정
$ NODE_ENV=development
$ node
```

```javascript
// 코드에서 환경 변수 읽어오기
if (process.env.NODE_ENV === 'production')
    useCaching = true;
```

<br/><br/>

### 다른 블로거의 의견

블로거 [dynatrace](https://www.dynatrace.com/blog/the-drastic-effects-of-omitting-node_env-in-your-express-js-applications/):
> ... Node.js 에는 NODE_ENV 라는 환경 변수를 사용하여 현재 모드를 설정하는 관습이 있습니다. 
> 우리는 실제로 NODE_ENV 읽고, 만약 그 값이 없다면 기본적으로 development 를 사용 한다는 것을 볼 수 있습니다. 
> NODE_ENV 값을 Production 으로 설정하면 CPU 사용량이 심지어 약간 감소하는 동안에도 Node 가 처리하는 요청의 수가 약 2/3 정도 증가하는것을 볼 수 있습니다.
> *다시 한번 강조하자면: NODE_ENV 를 Production 환경에서 설정하여 애플리케이션의 속도를 3배 이상 빠르게 하세요!*
> 
![NODE_ENV=production](https://github.com/goldbergyoni/nodebestpractices/blob/master/assets/images/setnodeenv1.png "NODE_ENV=production")

<br/><br/>
