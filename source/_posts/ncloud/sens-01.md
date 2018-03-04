---
title: Ncloud SENS를 이용하여 나만의 이벤트 통보 시스템 만들기
categories: ['ncloud']
date: 2018-03-04 20:00:00
comments: true
---

## 시작하기 전에 

서비스를 운영하는 것은 많은 어려움이 따르는데요.

특히 안정적으로 지속적인 서비스 운영을 위해서는 많은 신경이 쓰일 수 밖에 없습니다.

운영자가 개입이 요구되는 예상치 못한 이벤트 발생 시, 이를 빨리 캐치 후 대응하여 서비스 중단을 최소화 하는 것이 필수적입니다.

이번 기고에서는 Ncloud SENS를 이용하여 나만의 이벤트 통보 시스템 만들기를 기술해보겠습니다.

<!-- more --> 

### Ncloud SENS 간단 설명

SENS는 Simple & Easy Notification Service의 약자로서, 이를 이용하면 우리들의 서비스에 메시지 및 알람 전송 기능을 간편하게 구현할 수 있습니다.

이번 기고에서 다루어볼 예제는 SMS 발송 기능을 이용할 예정인데요, NCLOUD SENS는 SMS발송 기능을 OPEN API로 제공하고 있어 우리들이 구현한 로직 속에서 알람 전송을 구현할 수 있습니다. 


## 구현목표

**[그림. 나만의 이벤트 통보 시스템 구조]**

![](/assets/ncloud-sens-01.png)

구현에 앞서 먼저 해당 시스템에 대한 스펙을 정의해보면 다음과 같습니다.

> **OS**: Linux 계열 운영체제
> **Platform**: Nodejs
> **이벤트 감지 대상**: 로그파일
> **이벤트 통보 방법**: SMS(SENS OPEN API 이용)
 

SENS는 이를 우리들이 구현한 로직 속에서 메시지를 전송할 수 있도록 OPEN API를 제공하고 있습니다. 

이를 이용하여 특정 이벤트 발생시 SMS를 전송 받도록 구현하면 되겠습니다 ^^

## SENS API 분석 

SENS에 대한 OPEN API 문서(https://sens.ncloud.com/assets/html/docs/index.html?url=https://api-sens.ncloud.com/docs/openapi/ko)를 확인해보도록 하겠습니다.

![](/assets/ncloud-sens-02.png)


SMS에 관련된  발신자, 수신자, 메시지외에도 인증을 위한 X-NCP-auth-key, X-NCP-service-secret, serviceId 세가지 값이 필요합니다.

### X-NCP-auth-key

 X-NCP-auth-key값은 Ncloud 포털에서 얻을 수 있습니다.

마이페이지 > 계정관리 > 인증 관리 탭에 확인 할 수 있는 API 인증키 관리 부분에서 확인할 수 있는 Access Key ID가 이에 해당합니다.

인증키가 하나도 없는 경우에는 추가로 하나 생성하여 해당 키를 획득해야합니다.

![](/assets/ncloud-sens-03.png)

### X-NCP-service-secret & serviceId


 X-NCP-service-secret, serviceId 이 두 값은 SENS(https://sens.ncloud.com)의 프로젝트 메뉴에서 획득할 수 있습니다.

생성한 프로젝트에서 열쇠 모양을 클릭하면 아래와 같은 창이 팝업됩니다. 여기에 ID와 Secret Key에 해당하는 부분이 우리가 원하는 값이 됩니다.

![](/assets/ncloud-sens-04.png)


자 이제 OEPN API를 이용하여 SENS를 사용하기위한 인증키 값들을 모두 획득하였습니다.

이제 우리가 구축할 시스템에서 특정 이벤트가 발생시 SENS에 RESTful Message를 전송하여, 우리에게 통보가 되도록 하는 로직만 구성하면 됩니다.

 
## Ncloud SENS를 이용한 이벤트 통보 시스템 구현


앞서 정의한 스펙에 따라 로그파일에서 특정 메시지가 발견 시, 이벤트를 통보하도록 구성하겠습니다.

이를 위해서 Nodejs패키지인 tail package(https://github.com/lucagrulla/node-tail) 를 사용하겠습니다. tail package는 마지막 행을 기준으로

새로운 메시지가 기록될 시, 이를 우리가 알 수 있도록 event listener 에 이벤트를 발생시켜줍니다.

**[코드 예제: tail 사용 예제]**
```javascript
const Tail = require('tail').Tail;

tail = new Tail("fileToTail");

tail.on("line", function(data) {
  console.log(data);
});

tail.on("error", function(error) {
  console.log('ERROR: ', error);
});
```

새로운 로그가 추가될때 마다, 이를 “line” 이벤트 리스너에서 확인 할 수 있습니다.

우리는 이 data값에서 특정 메시지가 들어 있는 경우, OPEN API를 사용하면 되겠습니다.

 
SENS OPEN API를 사용하기 위한 RESTful 메시지를 생성하기 위한 모듈로 axios(https://github.com/axios/axios)를 사용하였습니다.

**[코드 예제: SENS OPEN API를 위한 클라이언트]**

```javascript
const axios = require('axios');

const { ['X-NCP-auth-key']: ncpAuthKey, ['X-NCP-service-secret']: sensServiceSecret, serviceId } = process.env;

module.exports = (function() {
  return {
    sendSms: ({ from, to, content }, callback) => {
      const message = {
        type: "sms",
        from,
        to, // array
        content
      };

      axios({
        method: 'POST',
        url: `https://api-sens.ncloud.com/v1/sms/services/${ encodeURI( serviceId ) }/messages`,
        headers: {
          'X-NCP-auth-key': ncpAuthKey,
          'X-NCP-service-secret': sensServiceSecret
        },
        data: message
      })
        .then( res => {
          callback( null, res.data );
        })
        .catch( err => {
          callback( err );
        }) // end axios Promise
    }
  }
})();
```

인증 값들로 필요한 X-NCP-auth-key, X-NCP-service-secret, serviceId 이 세가지 값들은 앞서 획득한 값들을 이용합니다.

serviceId의 경우 URI의 Path로 이용되므로 URI 인코딩을 해줘야함에 유의합니다. 해당 값들을 헤더와 URI에 입력하고,

message부분에 수신자, 발신자, 메시지 내용을 입력하면 OPEN API를 활용할 수 있습니다 ^^  

 
이제 “hello world” 로그 메시지가 발생했을 때 “An Event Occurred”라는 SMS를 통보받는 로직을 구현해보겠습니다.

**[코드 예제: hello world 시 SENS OPEN API로 메시지 전송]**

```javascript
require('dotenv').config();
const path = require('path');
const tracker = require('./helpers/tracker');
const sensApi = require('./helpers/ncloudSens');

const filePath = path.join( __dirname, 'log.txt' );

const tail = tracker.createTracker( filePath );

tail.on("line", function(data) {
  if ( data.includes("hello world") ) {
    sensApi.sendSms({
      from: "01028640911",
      to: ["01028640911"],
      content: "An Event Occurred"
    }, (error, result) => {
      if ( error ) {
        return console.log( error.message );
      }
      console.log( result );
    })
  }
});

tail.on("error", function(error) {
  console.log('ERROR: ', error);
});
```

line이벤트를 통해 새로운 로그 메시지가 발생하면 이를 확인 할 수 있습니다. “hello world”메시지가 있다면, 방금전 구현했던 SENS 클라이언트를 통해

SMS통보를 위한 RESTful메시지를 생성하면 되겠습니다

 
테스트를 위해서 작성한 시스템을 실행하고,  아래와 같이 log.txt에 “hello world” 메시지를 기록합니다.

```bash
$ echo “hello world” >> log.txt
```
 

사전에 정의 했던 “hello world” 메시지를 입력했네요. 이제 핸드폰을 통해 이벤트 발생 메시지를 통보 받을 수 있습니다.


![](/assets/ncloud-sens-05.png)


지금까지 기술한 예제는 아주 간단한 로직을 다루었으며, 추가 요구 사항에 따라 세부 로직을 추가하여

나만의 통보시스템을 보강할 수 있겠습니다.


지금까지 설명한 예제는 https://github.com/mosfet1kg/ncloudLog 에서 확인할 수 있습니다.

