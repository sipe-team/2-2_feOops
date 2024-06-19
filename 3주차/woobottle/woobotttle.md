데브옵스

1. 왜 Docker?
   1. 기존엔 vercel을 쓰고 있었음
      1. amplify도 쓰고 있었음
   2. 예전에 ec2에 직접 세팅을 해본적 있었음
      1. 상당히 불편
      2. rails / postgresql 세팅등 전부 커맨드로 일일히 설치 및 세팅
      3. 도커로 가보자
   3. 경험삼아 세팅해보자

      1. 회사에서는 도커로 배포 중

      ![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/359809db-cb83-47c2-9cce-0c45f96418ab/2bf76baf-8566-4623-9e7f-4e7c92a58b7c/Untitled.png)

      ecr(elastic container repository) ⇒ private docker hub 라고 이해하면 됨, Docker Container들이 관리되어 있음

      ecs(elastic container service) ⇒ task 단위로 도커 컨테이너가 이미지로 실행되어 관리 되는 곳

처음 시도

1. image 크기가 1.x gb 후반대 크기
   1. node 사이즈를 작은걸 씀
   2. nextJs의 standalone option을 사용
      1. nexstjs에서 제공하는 옵션
      2. 어플리케이션을 실행하는데 필요한 최소한의 코드만 추출
      3. .next/standalone/server.js에 실행파일이 생김
   3. 아래 코드

```docker
# Node.js 18-alpine 이미지를 기본으로 사용함
FROM node:18-alpine AS base

# working directory를 /app으로 사용
WORKDIR /app

# yarn을 설치
RUN npm install -g corepack && corepack enable

# Install Yarn Berry (v2 or v3)
RUN yarn set version berry

# 필요한 파일들을 복사
COPY .yarn .yarn
COPY .pnp.cjs .pnp.cjs
COPY .pnp.loader.mjs .pnp.loader.mjs
COPY package.json yarn.lock .yarnrc.yml ./

# berry를 지원하지 않는 라이브러리들도 있음 이 라이브러리 들은 /unplugged에 위치
# .yarn/unplugged에 있는 파일들을 설치하기 위한 코드
RUN yarn install --immutable

# 전체 프로젝트 파일들 복사
COPY . .

# yarn build => next.config.js에서 standalone 옵션 사용
RUN yarn build

FROM base AS runner

WORKDIR /app

\# base stage의 빌드 프로젝트를 복사해옴
# 도커 빌드가 될때 build context 외부의
# 다른 디렉터리의 위치한 파일에 접근이 불가능 하므로 새로 FROM~/WORKDIR~
# 을 선언하여 하위 디렉터리에서 복사해옴
COPY --from=base /app/.next/standalone ./
COPY --from=base /app/public ./public
COPY --from=base /app/.next/static ./.next/static

# 3000포트로 동작하도록 세팅
EXPOSE 3000

# Next.js 앱 실행
CMD ["node", "server.js"]

```

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/359809db-cb83-47c2-9cce-0c45f96418ab/8552b491-4732-41b0-bf9e-5225d2b19b78/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/359809db-cb83-47c2-9cce-0c45f96418ab/1849c15f-65d0-4af7-8d39-ccd5e0139d2a/Untitled.png)

1. 현재 겪고 있는 문제 (해결해야함…)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/359809db-cb83-47c2-9cce-0c45f96418ab/63afa700-0975-4179-b215-543121a4fe1e/Untitled.png)
