# templates-nodejs

## Node.js 웹 앱의 도커라이징

이 예제에서는 Node.js 애플리케이션을 Docker 컨테이너에 넣는 방법을 보여줍니다. 이 가이드는 개발 목적이지 프로덕션 배포용이 아닙니다. Docker가 설치되어 있고 Node.js 애플리케이션의 구조에 대한 기본적인 지식이 있어야 합니다.

먼저 간단한 Node.js 웹 애플리케이션을 만든 후에 이 애플리케이션을 위한 Docker 이미지를 만들어서 컨테이너로 실행할 것입니다.

Docker를 사용하면 애플리케이션과 모든 의존성을 소프트웨어 개발에서 컨테이너라고 부르는 표준화된 단위로 패키징할 수 있습니다. 컨테이너는 리눅스 운영체제의 간단 버전입니다. 이미지는 컨테이너에 로드하는 소프트웨어를 말합니다.

Node.js 앱 생성

우선, 모든 파일을 넣은 새로운 디렉터리를 만들겠습니다. 이 디렉터리 안에 애플리케이션과 의존성을 알려주는 package.json 파일을 생성하겠습니다.

{
  "name": "docker_web_app",
  "version": "1.0.0",
  "description": "Node.js on Docker",
  "author": "First Last <first.last@example.com>",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "dependencies": {
    "express": "^4.16.1"
  }
}
package.json 파일을 만든 후, npm install을 실행하세요. 버전 5 이상의 npm을 사용한다면, Docker 이미지에 복사할 package-lock.json 파일을 npm에서 생성할 것입니다.

이제 Express.js 프레임워크로 웹앱을 정의하는 server.js를 만들겠습니다.

'use strict';

const express = require('express');

// 상수
const PORT = 8080;
const HOST = '0.0.0.0';

// 앱
const app = express();
app.get('/', (req, res) => {
  res.send('Hello World');
});

app.listen(PORT, HOST, () => {
  console.log(`Running on http://${HOST}:${PORT}`);
});
다음 단계에서 공식 Docker 이미지를 사용해서 Docker 컨테이너 안에서 이 앱을 실행하는 방법을 살펴보겠습니다. 먼저 앱의 Docker 이미지를 만들어야 합니다.

Dockerfile 생성

Dockerfile이라는 빈 파일을 생성합니다.

touch Dockerfile
선호하는 텍스트 에디터로 Dockerfile을 엽니다.

가장 먼저 해야 할 것은 어떤 이미지를 사용해서 빌드할 것인지를 정의하는 것입니다. 여기서는 Docker Hub에 있는 node의 최신 LTS(장기 지원) 버전인 16을 사용할 것입니다.

FROM node:16
다음으로 이미지 안에 애플리케이션 코드를 넣기 위해 디렉터리를 생성할 것입니다. 이 디렉터리가 애플리케이션의 작업 디렉터리가 됩니다.

# 앱 디렉터리 생성
WORKDIR /usr/src/app
이 이미지에는 이미 Node.js와 NPM이 설치되어 있으므로 npm 바이너리로 앱의 의존성을 설치하기만 하면 됩니다. 버전 4 이하의 npm은 package-lock.json 파일을 생성하지 않을 것입니다.

# 앱 의존성 설치
# 가능한 경우(npm@5+) package.json과 package-lock.json을 모두 복사하기 위해
# 와일드카드를 사용
COPY package*.json ./

RUN npm install
# 프로덕션을 위한 코드를 빌드하는 경우
# RUN npm ci --omit=dev
작업 디렉터리 전체가 아닌 package.json 파일만을 복사하고 있는데, 이는 캐시된 Docker 레이어의 장점을 활용하기 위함입니다. bitJudo가 이에 대해 여기에 잘 설명해 두었습니다. 추가로, 주석에 언급된 npm ci 커맨드는 프로덕션 환경을 위한 더 빠르고, 신뢰할 수 있고, 재현 가능한 빌드를 제공합니다. 이에 대해 여기에서 자세히 알아볼 수 있습니다.

Docker 이미지 안에 앱의 소스코드를 넣기 위해 COPY 지시어를 사용합니다.

# 앱 소스 추가
COPY . .
앱이 8080포트에 바인딩 되어 있으므로 EXPOSE 지시어를 사용해서 docker 데몬에 매핑합니다.

EXPOSE 8080
마지막으로 런타임을 정의하는 CMD로 앱을 실행하는 중요 명령어를 정의해야 합니다. 여기서는 서버를 구동하도록 node server.js을 실행하는 기본 npm start을 사용할 것입니다.

CMD [ "node", "server.js" ]
Dockerfile은 다음과 같아야 합니다.

FROM node:16

# 앱 디렉터리 생성
WORKDIR /usr/src/app

# 앱 의존성 설치
# 가능한 경우(npm@5+) package.json과 package-lock.json을 모두 복사하기 위해
# 와일드카드를 사용
COPY package*.json ./

RUN npm install
# 프로덕션을 위한 코드를 빌드하는 경우
# RUN npm ci --omit=dev

# 앱 소스 추가
COPY . .

EXPOSE 8080
CMD [ "node", "server.js" ]
.dockerignore 파일

Dockerfile과 같은 디렉터리에 .dockerignore 파일을 다음 내용으로 만드세요.

node_modules
npm-debug.log
이는 Docker 이미지에 로컬 모듈과 디버깅 로그를 복사하는 것을 막아서 이미지 내에서 설치한 모듈을 덮어쓰지 않게 합니다.

이미지 빌드

작성한 Dockerfile이 있는 디렉터리로 가서 Docker 이미지를 빌드하는 다음 명령어를 실행하세요. -t 플래그로 이미지에 태그를 추가할 수 있어 나중에 docker images 명령어로 쉽게 찾을 수 있습니다.

docker build . -t <your username>/node-web-app
Docker가 당신이 빌드한 이미지를 보여줄 것입니다.

$ docker images

# 예시
REPOSITORY                      TAG        ID              CREATED
node                            16         1934b0b038d1    5 days ago
<your username>/node-web-app    latest     d64d3505b0d2    1 minute ago
이미지 실행

-d로 이미지를 실행하면 분리 모드로 컨테이너를 실행해서 백그라운드에서 컨테이너가 돌아가도록 합니다. -p 플래그는 공개 포트를 컨테이너 내의 비공개 포트로 리다이렉트합니다. 앞에서 만든 이미지를 실행하세요.

docker run -p 49160:8080 -d <your username>/node-web-app
앱의 로그를 출력하세요.

# 컨테이너 아이디를 확인합니다
$ docker ps

# 앱 로그를 출력합니다
$ docker logs <container id>

# 예시
Running on http://localhost:8080
컨테이너 안에 들어가 봐야 한다면 exec 명령어를 사용할 수 있습니다.

# 컨테이너에 들어갑니다
$ docker exec -it <container id> /bin/bash
테스트

앱을 테스트하려면 Docker 매핑된 앱 포트를 확인합니다.

$ docker ps

# 예시
ID            IMAGE                                COMMAND    ...   PORTS
ecce33b30ebf  <your username>/node-web-app:latest  npm start  ...   49160->8080
위 예시에서 Docker가 컨테이너 내의 8080 포트를 머신의 49160 포트로 매핑했습니다.

이제 curl로 앱을 호출할 수 있습니다.(필요하다면 sudo apt-get install curl로 설치하세요.)

$ curl -i localhost:49160

HTTP/1.1 200 OK
X-Powered-By: Express
Content-Type: text/html; charset=utf-8
Content-Length: 12
ETag: W/"c-M6tWOb/Y57lesdjQuHeB1P/qTV0"
Date: Mon, 13 Nov 2017 20:53:59 GMT
Connection: keep-alive

Hello world
간단한 Node.js 애플리케이션을 Docker로 실행하는데 이 튜토리얼이 도움되었길 바랍니다.

다음 링크에서 Docker와 Docker에서의 Node.js에 대한 정보를 더 자세히 볼 수 있습니다.

공식 Node.js Docker 이미지
Node.js Docker 사용사례 문서
공식 Docker 문서
Stack Overflow에 Docker 태그로 올라온 질문
Docker 레딧
