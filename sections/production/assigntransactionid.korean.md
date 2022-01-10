# 각 로그의 트렌젝션에는 'TransactionId' 할당하기

<br/><br/>

### 한 문단 요약

&nbsp;&nbsp;일반적으로 로그는 모든 컴포넌트와 요청들의 창고와 같다. 
의심스러운(문제가 있다고 생각되는) 라인이나 오류가 감지되면, 특정한 흐름과 (예: John 이 무언가를 구입하려고 시도하는 과정)관련이 있는 다른 로그들도 함께 살필 수 있어야 한다.
특히, 요청/트렌젝션이 여러 컴퓨터에 걸쳐져 있는 마이크로서비스 환경에서는 이 문제가 더 도전적이고 중요해진다.
이 문제를 해결하려면 같은 요청에 대해 고유한 Transaction ID를 부여 하면 된다. 이로써 한 줄을 검사 할 때 그 ID를 이용하여 동일한 흐름에 있는 트렌젝션 로그를 검색 할 수 있게 된다. 
그러나, 싱글 스레드를 사용하여 모든 요청을 처리하는 Node 의 모델로는 이를 달성하기 쉽지 않다. 따라서 요청 수준에서 데이터를 그룹화하는 라이브러리 사용을 검토하자.
다음 슬라이드의 코드 예제를 참고하자. 다른 마이크로서비스에서 호출 할 때, 'x-transaction-id' 라는 HTTP header 를 전달하여 같은 문맥을 유지 할 수 있다.

<br/>

### 예제코드: [async-local-storage](https://nodejs.org/api/async_hooks.html#async_hooks_class_asynclocalstorage) 를 사용하여 요청 기능과 서비스 간에 TransactionId 공유하기

 **async-local-storage가 무엇인가?** 몇몇 사람들은 이것이 쓰레드 별 고유한 저장소를 가지게 하는 노드의 대안이라고 생각 할 수 있다.
 이것은 노드의 비동기 흐름을 위한 저장소다. 더 자세한 내용은 [여기에서](https://www.freecodecamp.org/news/async-local-storage-nodejs/)

```javascript
const express = require('express');
const { AsyncLocalStorage } = require('async_hooks');
const uuid = require('uuid/v4');

const asyncLocalStorage = new AsyncLocalStorage();

// 들어오는 요청에 Transaction Id 설정
const transactionIdMiddleware = (req, res, next) => {
    // asyncLocalStorage.run() 의 첫 번째 인수는 저장소의 상태를 초기화하고, 두 번째 인수는 이 저장소에 엑세스 할 수 있는 함수 입니다.
    asyncLocalStorage.run(new Map(), () => {
        // TransationId 를 요청 해더에서 추출합니다. 만약 없다면 새로 만듭니다.
        const transactionId = req.headers['transactionId'] || uuid();

        // TransactionId 를 스토어에 저장합니다.
        asyncLocalStorage.getStore().set('transactionId', transactionId);
        
        // next() 를 함수 내에서 호출하여, 같은 AsyncLocalStorage 문맥에서 다른 모든 미들웨어가 실행되도록 보장합니다.
        next();
    });
};

const app = express();
app.use(transactionIdMiddleware);

// 나가는 요청에 TransactionId 설정
app.get('/', (req, res) => {
    // 한번 TransactionId 가 미들웨어에서 설정되면, 요청 플로우 어디서든 접근 할 수 있습니다.
    const transactionId = asyncLocalStorage.getStore().get('transactionId');

    try {
        // 다음 서비스에 전달하기 위해 TransactionId 를 해더에 추가합니다.
        const response = await axios.get('https://externalService.com/api/getAllUsers', headers: {
        'x-transaction-id': transactionId
        });
    } catch (err) {
        // 오류는 미들웨어에 전달되기 때문에 TransactionId 를 보낼 필요가 없습니다.
        next(err);
    }

    logger.info('externalService was successfully called with TransactionId header');

    res.send('OK');
});

// Logger 를 호출하는 에러 핸들링 미들웨어
app.use(async (err, req, res, next) => {
    await logger.error(err);
});

// 이제 로거는 동일한 요청 항목이 동일한 값을 갖도록 각 항목에 TransactionId 를 설정 할 수 있습니다.
class logger {
    error(err) {
        console.error(`${err} ${asyncLocalStorage.getStore().get('transactionId')}`);
    }

    info(message) {
        console.log(`${message} ${asyncLocalStorage.getStore().get('transactionId')}`);
    }
}
```
<br/>

<details>
<summary><strong>코드 예제: 문법을 간단하게 하기 위해 Helper 라이브러리 사용</strong></summary>

[cls-rtracer](https://www.npmjs.com/package/cls-rtracer) (async-local-storage 기반으로 작성, Express & Koa middlewares and Fastify & Hapi 플러그인을 위해 추상화 됨) 를 이용하여 요청 기능간에 TransactionId 공유
  
```javascript
const express = require('express');
const rTracer = require('cls-rtracer');

const app = express();

app.use(rTracer.expressMiddleware());

app.get('/getUserData/{id}', async (req, res, next) => {
    try {
        const user = await usersRepo.find(req.params.id);

        // TransactionId 는 로거 내부에 도달 할 수 있기 때문에 또 보낼 필요가 없습니다.
        logger.info(`user ${user.id} data was fetched successfully`);

        res.json(user);
    } catch (err) {
        // 오류는 미들웨어에 전달됩니다.
        next(err);
    }
})

// Error handling middleware calls the logger
app.use(async (err, req, res, next) => {
    await logger.error(err);
});

// The logger can now append the TransactionId to each entry so that entries from the same request will have the same value
class logger {
    error(err) {
        console.error(`${err} ${rTracer.id()}`);
    }
    
    info(message) {
        console.log(`${message} ${rTracer.id()}`);
    }
}
```
<br/>

마이크로서비스 간에 TransactionId 공유

```javascript
// cls-tracer는 기본 미들웨어 구성을 재정의 하여 서비스의 나가는 요청 헤더에 TransactionId 를 저장하고, 들어오는 요청 헤더에서 TransactionId 를 추출 할 수 있습니다.
app.use(rTracer.expressMiddleware({
    // Add TransactionId to the header
    echoHeader: true,
    // Respect TransactionId from header
    useHeader: true,
    // TransactionId header name
    headerName: 'x-transaction-id'
}));

const axios = require('axios');

// 이제 외부 서비스에서도 자동으로 TransactionId 를 헤더에서 가져옵니다.
const response = await axios.get('https://externalService.com/api/getAllUsers');
```
</details>
<br/>

**중요: async-local-storage를 사용하려면 만족해야하는 두 가지 조건**
1. Node v14 이상
2. async_hooks라는 노드의 하위 레벨 구조에 기반을 두고 있고, 이는 여전히 실험적인 기능이므로 성능 문제가 발생 할 수 있다. 물론 그것이 존재해도 미미한 수준이지만 결정에 참고해야한다.
<br/>

<details>
<summary><strong>예제 코드 - async-local-storage의 의존성 없는 전형적인 Express 설정</strong></summary>

```javascript
// when receiving a new request, start a new isolated context and set a transaction id. The following example is using the npm library continuation-local-storage to isolate requests

const { createNamespace } = require('continuation-local-storage');
const session = createNamespace('my session');

router.get('/:id', (req, res, next) => {
    session.set('transactionId', 'some unique GUID');
    someService.getById(req.params.id);
    logger.info('Starting now to get something by id');
});

// Now any other service or components can have access to the contextual, per-request, data
class someService {
    getById(id) {
        logger.info('Starting to get something by id');
        // other logic comes here
    }
}

// The logger can now append the transaction id to each entry so that entries from the same request will have the same value
class logger {
    info (message) {
        console.log(`${message} ${session.get('transactionId')}`);
    }
}
```
</details>

<br/><br/>

### Good: 로그에 TransactionId 가 할당 - 보고 싶은 하나의 흐름에 필터를 사용할 수 있음
![alt text](https://i.ibb.co/YjJwgbN/logs-with-transaction-id.jpg "Logs with transaction id")
<br/><br/>

### Bad: 로그에 TransactionId 가 없음 - 보고 싶은 하나의 흐름에 필터를 적용 할 수 없음, 여기저기 섞여있는 로그 사이에 어떤 로그가 관련이 있는지 스스로 찾아야 한다.
![alt text](https://i.ibb.co/PFgVNfn/logs-withtout-transaction-id.jpg "Logs with transaction id")

<br/><br/>

### 블로그 인용:  "상관관계 ID라는 개념은 간단하다." 지정된 트렌젝션의 모든 요청, 메시지 및 응답에 공통적으로 주워진 값이다. 이 간단한것으로 너는 많은 힘(이점)을 얻을 수 있다.

From [rapid7](https://blog.rapid7.com/2016/12/23/the-value-of-correlation-ids/)

> In the old days when transactional behavior happened in a single domain, in step-by-step procedures, keeping track of request/response behavior was a simple undertaking. However, today one request to a particular domain can involve a myriad of subsequent asynchronous requests from the starting domain to others. For example, you send a request to Expedia, but behind the scenes Expedia is forwarding your request as a message to a message broker. Then that message is consumed by a hotel, airline and car rental agency that responds asynchronously too. So the question comes up, with your one request being passed about to a multitude of processing consumers, how do we keep track of the transaction? The answer is: use a Correlation ID.
