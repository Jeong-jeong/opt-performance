<p align="center">
<a href="https://www.inflearn.com/course/웹-성능-최적화-리액트-1/dashboard">프론트엔드 개발자를 위한, 실전 웹 성능 최적화(feat. React) - Part. 1' 강의</a>를 수강하며 정리한 내용이다.</p>

# 블로그 성능 최적화

## LightHouse

LightHouse `데스크톱` 분석 결과 **70점**이 나왔고 추천과 진단 탭에서는 아래와 같은 문제점이 발견되었다.

<img width="564" alt="image" src="https://user-images.githubusercontent.com/68528752/164903818-9ebee786-bf01-4458-b80d-60b2ba0a14ec.png">

### 이미지 크기 적절히 설정하기

<img width="649" alt="image" src="https://user-images.githubusercontent.com/68528752/164908905-de0636bb-6ceb-4922-bdd2-3717b8ce2e94.png">

현재 이미지는 api를 통해 불러오는데 사용되는 이미지 크기(120 x 120)에 비해 원본 픽셀값(1200 x 1200)이 너무 크다.
레티나 디스플레이의 경우 보여지는 크기에 비해 좀더 많은 픽셀값을 가질 수 있기 때문에 보통은 **2배**인 240 \* 240 으로 크기를 설정하면 적당하다고 한다.

api로 불러오는 이미지를 어떻게 최적화할 수 있을까?

#### CDN(Contents Delivery Network)

- Image CDN(image processing CDN)
  이미지를 중간에 변환해서 사용자에게 전달할 수 있음.

```js
// 형태 예시
 http://cdn.image.com?src=img.src&width=200&height=100
```

CDN을 직접 구축하지 않고 아래와 같은 솔루션을 사용할 수도 있음.
ex) [imgIX](https://imgix.com/)

다만 현재는 unsplash 에서 img를 받아오므로 자체적으로 크기를 줄이고 LightHouse를 돌려보았다.

전체 점수가 70점에서 **82점**으로 크게 올랐고 추천탭에 있던 이미지 크기 관련 경고가 사라졌다.

<img width="568" alt="image" src="https://user-images.githubusercontent.com/68528752/164909498-64eb4317-4827-4a69-a298-6e3557fad1b3.png">

### (사용하지 않는) 자바스크립트 줄이기

 <img width="597" alt="image" src="https://user-images.githubusercontent.com/68528752/165757985-b21a000d-4bfd-44e1-8317-6b5f7d91fea6.png">

[Lighthouse JS 줄이기] 항목에서 `...0.chunk.js` 파일의 전송크기가 확연히 큰 것을 확인했지만 내부의 어떤 요소가 원인인지는 정확히 나오지 않았다.

<img width="883" alt="image" src="https://user-images.githubusercontent.com/68528752/165764360-4b061e09-0ade-4fd4-927d-7a4241eedd19.png">

[성능] 측정 결과 화면에 첫 렌더링 시점인 FCP & LCP 이후 aritcles 네트워크 요청이 이루어지고 아래쪽에서 `Aritcle 함수`가 오랫동안 실행이 된 것이 보인다.

   <img width="879" alt="image" src="https://user-images.githubusercontent.com/68528752/165764957-bb06e3ca-a71c-4c98-9107-3b34c8e3a576.png">

좀더 확대해보면 Aritcle 컴포넌트 내에서 removeSpecialCharacter 함수가 오랫동안 지속적으로 실행되었고, CG에 의해 중간중간 끊겨있었다.

`removeSpecialCharacter 함수`는 api로 받아온 마크다운 문자열에서 특수 문자를 제거해주는 역할을 한다. 하나의 마크다운 파일은 몇만자까지도 길어지는데 내부에서 **이중 반복문과 substring, concat을 난발하여 성능을 악화**시키는 것이었다.

이를 해결하기 위해 1. replace 함수로 정규표현식을 사용하여 함수를 한번에 실행시키고, 2. 받아온 마크다운 문자열을 다 검사하지 않고 LCP에서 보이는 200 ~ 300자 정도의 글자만 잘라주기로 했다.

<img width="685" alt="image" src="https://user-images.githubusercontent.com/68528752/165767731-996db5df-24a5-4916-8cb7-b03ea0627dbe.png">

리팩터링 후 성능 재측정 결과 articles 네트워크 요청 후 Aritcle 실행 부분이 너무 작아서 보이지 않게 되었다. 클릭해서 테두리가 빨간 부분인데 확연이 줄어든 것을 확인할 수 있다.
<img width="883" alt="image" src="https://user-images.githubusercontent.com/68528752/165764360-4b061e09-0ade-4fd4-927d-7a4241eedd19.png">

이전과 비슷한 조건으로 축소시켜보면 함수 실행 부분의 폭이 크게 줄었다.

<img width="343" alt="image" src="https://user-images.githubusercontent.com/68528752/165768212-6282f2a0-e61a-4fe1-bff9-f35cb22612c8.png">

LightHouse 측정 결과도 82점에서 **91점**으로 증가하였다.

<img width="597" alt="image" src="https://user-images.githubusercontent.com/68528752/165768585-efb9decd-8177-45bc-ac04-ab4224bdce56.png">

### Code Spliting & lazy loading

[Lighthouse]

<img width="597" alt="image" src="https://user-images.githubusercontent.com/68528752/165757985-b21a000d-4bfd-44e1-8317-6b5f7d91fea6.png">

[성능]
<img width="789" alt="image" src="https://user-images.githubusercontent.com/68528752/165772639-f8c32797-bf68-4000-b8f5-4bbc2018111b.png">

두 개의 결과를 종합해보면 0.chunk.js 파일의 크기가 995.4KiB로 상당히 큰 게 보인다. main.chunk.js 파일도 크지만 이런 JS 파일의 다운로드 크기가 클 수록 HTML 파싱 결과가 늦어지게 되므로 그만큼 화면이 늦게 보일 수 있다.

번들링된 JS chunk 파일들의 크기를 상세히 파악하기 위해 [webpack-bundle-analyzer](https://www.npmjs.com/package/webpack-bundle-analyzer) 라이브러리를 사용할 수 있지만 CRA로 webpack 설정이 필요하기 때문에 귀찮고 복잡하다. [cra-bundle-analyzer](https://www.npmjs.com/package/cra-bundle-analyzer)를 사용하면 eject나 craco 등의 추가 설정 없이도 CRA에서 webpack-bundle-analyzer를 사용할 수 있다.

```shell
# 설치
yarn add -D cra-bundle-analyzer

# 실행
npx cra-bundle-analyzer
```

<img width="1090" alt="image" src="https://user-images.githubusercontent.com/68528752/165774432-9c6efb97-d3dd-4416-8ad3-5eb3de8c0ba5.png">

static/~~.chunk.js 파일이 번들된 크기의 상당 부분을 차지한다 ㄷㄷ. 이름은 번들될 때마다 바뀌기 때문에 0.chunk.js 파일인지 파악할 수 없지만 chunk.js 내부에 상당부분을 `refractor` 라는 게 큰 부분을 차지한다.
<br />

[yarn.lock]
<img width="477" alt="image" src="https://user-images.githubusercontent.com/68528752/165775171-16184faa-3df0-4590-bd44-77198f75df22.png">

yarn.lock (or package.lock.json) 파일 내부에서 refactor를 찾아본 결과 `react-sysntax-highlighter` 라이브러리에서 사용되고 있었다. 해당 라이브러리는 **맨 처음 페이지 로딩 시에는 사용되지 않고** 각 아티클 내부에서만 사용해도 무방한 라이브러리이다. 이를 위해 필요한 페이지에서만 필요한 파일을 다운받을 수 있도록 code-spliting을 진행할 것이다.

#### Code Spliting

[React - 코드 분할](https://ko.reactjs.org/docs/code-splitting.html)에서 다양한 방법들을 공부할 수 있다. 코드 분할에는 `페이지 단위로 분할`, `모듈 단위로 분할` 등 여러 방안들이 존재하지만 적재적소에 분할하기란 쉽지 않다. 따라서 간편하게 Route 되는 단위로 균등하게 분할해볼 것이다.

```jsx
import React, { Suspense, lazy } from "react";
// import ListPage from './pages/ListPage/index'
// import ViewPage from './pages/ViewPage/index'

const ListPage = lazy(() => import("./pages/ListPage/index"));
const ViewPage = lazy(() => import("./pages/ViewPage/index"));
```

기존 코드를 lazy문으로 import 하여 Route 시에 해당 페이지로 진입했을 때 필요한 파일을 다운받도록 한다.

이렇게만 하면 아래와 같이 fallback이 없다는 오류가 발생한다.

```jsx
Error: A React component suspended while rendering, but no fallback UI was specified.
```

Suspense를 이용해 필요한 페이지가 보이지 않을 때 활용할 UI를 fallback 시켜준다.

```jsx
<Suspense fallback={<div>loading...</div>}>
  <Switch>
    <Route path="/" component={ListPage} exact />
    <Route path="/view/:id" component={ViewPage} exact />
  </Switch>
</Suspense>
```

(⚠️ 이제는 Switch 문대신 Routes + Route 문이 사용되기 때문에 위 코드는 참고만 바란다.)
<br />

<img width="1440" alt="image" src="https://user-images.githubusercontent.com/68528752/165777747-cbf79110-daae-494d-8745-ed928f65dd40.png">

재측정 결과 chunk.js 파일이 크게 두개로 분리되었다. 여전히 refractor가 사용되는 파일의 번들 크기가 크다 ㅎㅎ;.

<br />
[Lighthouse]
<img width="597" alt="image" src="https://user-images.githubusercontent.com/68528752/165768585-efb9decd-8177-45bc-ac04-ab4224bdce56.png">
<img width="612" alt="image" src="https://user-images.githubusercontent.com/68528752/165778396-ce80b65f-a1f2-48b1-b3fb-d95a9f4323ae.png">

와오 😮 Lighthouse 성능 측정 결과가 91점에서 **97점**으로 대폭 상승하고 진단 결과에서 JS 파일 줄이기 항목이 사라졌다. <br/>
Code spliting을 통해 `FCP`를 1.3초에서 0.8초로, `TTI(유저가 처음 interect 할 수 있는 시간)`도 1.3초 -> 0.8초로, `LCP(첫 렌더링 시 가장 큰 컨텐츠가 보이는 시간)`도 1.4초에서 1.2초로 많이 개선했다!!! 뿌듯 ㅎㅅㅎ

### 텍스트 압축하기

Lighthouse에서 텍스트 관련 경고는 없었지만, 간단히 텍스트 압축을 어떻게 하고 어떤 파일에서 텍스트 압축해야 되는지 알아보겠다.

우선 성능 측정은 **로컬 환경이 아닌 배포 환경**에서 진행해야 더 정확하다. production 배포 환경에서 적용되지 않았던 minify나 난독화 등에 따라 결과가 달라질 수 있기 때문이다.

serve.js 라이브러리를 통해 배포 환경을 간단히 구성하고 성능을 재측정한 결과 측정 항목들이 대략 절반 정도로 줄어서 **99점**으로 올랐다.

```json
// package.json
 "serve": "npm run build && node ./node_modules/serve/bin/serve.js -u -s build",
```

<img width="612" alt="image" src="https://user-images.githubusercontent.com/68528752/165778396-ce80b65f-a1f2-48b1-b3fb-d95a9f4323ae.png">

   <img width="568" alt="image" src="https://user-images.githubusercontent.com/68528752/165885916-cb261f6b-ec2c-42dc-8925-d7555b0de8d4.png">

<br />

serve.js에선 간단히 `-u 옵션(no-compression)`을 제거하여 텍스트를 압축할 수 있다. 다만 실무에선 매 서버마다 해당 옵션을 줄 수 없으므로 **공통적으로 사용하는 라우터 서버**에서 텍스트 압축을 적용하거나 한다고 한다. 이 역할은 번들 파일을 내려주는 쪽에서 보통 하고 프론트엔드에서 번들 파일을 직접 관리한다면 프론트에서 아니면 백엔드에서 처리한다.

압축을 적용해야 하는 기준은 보통 **2KB**이며, 2KB를 넘으면 적용, 넘지 않으면 적용하지 않는다. 프론트엔드 서버에서 압축을 해제해야 하는데 이 또한 비용이 들기 때문이다.

```json
// package.json
 "serve": "npm run build && node ./node_modules/serve/bin/serve.js -s build",
```

-u 옵션 삭제 후 네트워크 articles 를 살펴보면 인코딩 형식이 `gzip` 으로 변화한다. gzip으로 압축될 경우 적게는 1/5 ~ 1/10까지도 줄어든다고 한다 와아!!!!

<img width="313" alt="image" src="https://user-images.githubusercontent.com/68528752/165886171-6e06bed4-242d-4b9c-87da-67d09b16e4e7.png">
