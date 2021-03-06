# MSW Tutorial

> MSW에 대해서 알아보고, 사용방법을 알아보자


## 0. 프로젝트 준비
이 튜토리얼은 Next.js - TypeScript를 사용합니다.
```linux
$ npx create-next-app msw-test --ts
```

## 1. MSW
> Mock by intercepting requests on the network level. Seamlessly reuse the same mock definition for testing, development, and debugging.

  기존에 우리가 Mock 데이터를 가져오는 방식은 데이터를 service 레이어에서 mock 데이터로 갈아끼우는 형태로 사용했습니다. 그러나 이러한 방식은 실제 API를 호출하는 방식이 아닌 구성된 JSON 데이터를 가져오는 것에 그치지 않습니다. MSW를 사용하는 이유는 실제 사용자가 데이터를 받는 형태를 구성하기 위해서 사용하는 것입니다. MSW는 네트워크 레벨에서 요청을 가로채기 위해서 Service Worker를 사용하는데, 덕분에 모킹 여부와 상관 없이 애플리케이션이 잘 동작하는 것을 보장하고, 특히 코드를 변경할 필요가 없게 해줍니다.

  우리는 기존의 API 호출 코드를 주석 처리하고 임의의 데이터를 반환하도록 수정하는 식의 방식을 해왔지만 그렇게 하면 기존 코드를 확인하는 것이 아닌 임시로 코드를 수정하고 다시 되돌리는 식의 작업을 하게 됩니다. 따라서 서비스 워커를 사용하여 기존 코드 그대로 API 모킹을 통해 작동 여부를 확인하면, 실제 사용자는 API를 요청하는 방식으로, 개발자는 sMock API를 가져오는 형태(Dev 모드에서)로만 사용할 수 있습니다. MSW에서 강조하는 **"실제 사용자가 사용하는 방식을 테스트한다."**를 실천할 수 있습니다.

### 1.1 MSW 설치하기
msw를 devDependency로 설치한다.

```linux
$ npm install msw --save-dev

# or

$ yarn add msw --dev
```

### 1.2 적용 및 설정
project root에 src/mocks 디렉터리를 구성하고 `brower.ts`와 `server.ts`를 생성합니다.

`src/mocks/browser.ts`
```ts
// src/mocks/browser.ts
import { setupWorker, SetupWorkerApi } from 'msw';
import { handlers } from './handlers';

export const worker: SetupWorkerApi = setupWorker(...handlers);
```

`src/mocks/server.ts`
```ts
// src/mocks/server.ts
import { setupServer, SetupServerApi } from 'msw/node';
import { handlers } from './handlers';

export const server: SetupServerApi = setupServer(...handlers);
```

handler를 만들어 Mock API를 구성합니다.

`src/mocks/handler/index.ts`
```ts
// src/mocks/handler/index.ts
import { rest } from "msw";
import { users } from './data/users';

export const handlers = [
  rest.get("https://backend.dev/users", (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.delay(200),
      ctx.json(users),
      );
  }),
];
```

`src/mocks/handler/data/users.ts`
```ts
// src/mocks/handler/data/users.ts
export const users = [
  {
    id: 1,
    name: "hanpy",
    username: "King",
    email: "hanpy@gmail.com",
    phone: "1-770-736-8031 x56442",
  },
  {
    id: 2,
    name: "Ervin Howell",
    username: "Antonette",
    email: "Shanna@melissa.tv",
    phone: "010-692-6593 x09125",
  },
];
```

그리고 아래 명령어를 실행합니다. 이 명령어는 Service Worker를 생성, 등록합니다. `/public/mockServiceWorker.js` 파일이 생성될 것입니다.

```linux
$ npx msw init public/ --save
```

App의 Development mode에서만 실행될 수 있도록 `/pages/_app.tsx`에서 아래 코드를 작성합니다.
그전에 Service Worker가 `development` 모드에서만 실행될 수 있도록 세팅합니다.

`src/mocks/index.js`
```js
// src/mocks/index.js
if (typeof window === 'undefined') {
  const { server } = require('./server');
  server.listen();
} else {
  const { worker } = require('./browser');
  worker.start();
}
```

`pages/_app.tsx`
```ts
// pages/_app.tsx
import '../styles/globals.css';
import type { AppProps } from 'next/app';

if (process.env.NODE_ENV === 'development') {
  require('../src/mocks');
}

function MyApp({ Component, pageProps }: AppProps) {
  return <Component {...pageProps} />;
}

export default MyApp;
```

여기까지 MSW의 기본 설정이 되었습니다. 다음은 Next.js에서 `getServersideProps`와 Client-side에서 fetching하는 방법을 알아봅시다.
