# docker-trial
Docker와 Docker Compose 써보기

## docker 운영에 필요한 CLI 명령어들

### Container Image 빌드

`Dockerfile`이 있는 상태에서 아래 명령어를 실행.

```bash
$ docker build -t (image 이름) .
$ docker build -t getting-started .
```

### App Container 실행

```bash
$ docker run -dp (host의 포트 번호):(container의 포트 번호) (image 이름)
$ docker run -dp 3000:3000 getting-started
```

### 실행 중인 Container 보기

```bash
$ docker ps
```

### 실행 중인 Container 중단하기

`docker ps`를 통해 container ID를 확인할 수 있음.

```bash
$ docker stop (container ID)
```

### 중단된 Container 제거하기

```bash
$ docker rm (container ID)
```

### Container에서 명령어 실행하기

다음 두 가지 방법 중 하나를 사용한다.

1. docker desktop의 대시보드에서 container의 CLI 아이콘 클릭 &rarr; 생성되는 터미널 창에서 명령어 실행
2. container ID를 사용해서 로컬 터미널 창에서 명령어 실행

```bash
$ docker exec (container ID) (명령어)
```

### Container의 파일 시스템

모든 container는 독립된 파일 시스템을 가진다. 같은 image로부터 실행되었다 할지라도.

### Container Volume

volume을 사용하면, host의 디스크 공간을 container의 특정 디렉토리에 mount할 수 있다. container에서 해당 디렉토리에 파일을 추가하거나 삭제하면 host에서도 확인할 수 있고, 새로운 container를 실행할 때 같은 volume을 사용하면, 파일 시스템과 별개로 해당 디렉토리는 공유할 수 있다.

### 이름있는 Volume 생성하기

```bash
$ docker volume create (volume 이름)
```

### Volume과 함께 App Container 실행

```bash
$ docker run -dp (host의 포트 번호):(container의 포트 번호) -v (volume 이름):(volume을 mount할 container의 절대경로) (image 이름)
$ docker run -dp 3000:3000 -v todo-db:/etc/todos getting-started
```

### Volume이 생성되는 host 경로 확인하기

다음 명령어를 통해, container에 mount되는 host의 디스크 공간(`Mountpoint`)을 확인할 수 있다.\
단, Docker Desktop을 사용 중인 경우, docker 명령어는 작은 VM에서 실행되고, `Mountpoint` 역시 그 VM에서의 경로를 나타낸다.

```bash
$ docker volume inspect (volume 이름)
```

### Bind mount

bind mount는 volume을 다루는 또다른 방식이다. bind mount를 사용하면 host의 특정 디렉토리를 container에 mount할 수 있다.

### Bind mount와 함께 App Container 실행

PowerShell에서 명령어 개행 시, `\` 대신 `` ` `` 를 사용해야 한다.

```bash
$ docker run -dp (host의 포트 번호):(container의 포트 번호) \
    -w (명령이 실행될 container의 절대경로) \
    -v (host의 디렉토리 경로):(volume을 mount할 container의 절대경로) \
    (image 이름) \
    (working directory에서 실행할 명령어)
$ docker run -dp 3000:3000 \
    -w /app \
    -v "$(pwd):/app" \
    node:12-alpine \
    sh -c "pnpm install && pnpm run dev"
```

### Container Networking

컨테이너는 기본적으로 격리되어 실행되며, 같은 시스템의 다른 프로세스나 컨테이너와 통신할 수 없다. 다른 프로세스나 컨테이너와 통신하려면, 같은 네트워크 안에 소속되게 해야 한다.

### Network 생성하기

```bash
$ docker network create (network 이름)
```

### Network와 함께 MySQL Container 실행

```bash
$ docker run -d \
    --network (network 이름) \
    --network-alias (network 안에서 사용될, container의 hostname) \
    -v (volume 설정) \
    -e MYSQL_ROOT_PASSWORD=(mysql root 계정의 비밀번호) \
    -e MYSQL_DATABASE=(앱에서 사용할 database의 이름) \
    mysql:5.7
$ docker run -d \
    --network todo-app \
    --network-alias mysql \
    -v todo-mysql-data:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=secret \
    -e MYSQL_DATABASE=todos \
    mysql:5.7
```

### MySQL Container 접속하기

```bash
$ docker exec -it (mysql container의 ID) mysql -u root -p
```

### Container의 내부 IP 확인하는 방법

```bash
$ docker run -it --network (내부 IP를 확인하려는 container가 소속된 network의 이름) nicolaka/netshoot
```

`nicolaka/netshoot` container를 실행한 후, `dig mysql`을 입력하면 다음과 같은 형태의 출력을 확인할 수 있다.

```bash
; <<>> DiG 9.16.22 <<>> mysql
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 7876
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;mysql.                         IN      A

;; ANSWER SECTION:
mysql.                  600     IN      A       (mysql container의 내부 IP)

;; Query time: 0 msec
;; SERVER: 127.0.0.11#53(127.0.0.11)
;; WHEN: Sun Mar 13 04:05:36 UTC 2022
;; MSG SIZE  rcvd: 44
```

위에서 mysql container를 실행할 때 `network alias`를 사용했기 때문에, 튜토리얼에서는 내부 IP를 몰라도 상관없다.

### MySQL과 함께 App Container 실행

```bash
$ docker run -dp (host의 포트 번호):(container의 포트 번호) \
    -w (명령이 실행될 container의 절대경로) \
    -v (host의 디렉토리 경로):(volume을 mount할 container의 절대경로) \
    --network (network 이름) \
    -e MYSQL_HOST=(mysql container의 network alias) \
    -e MYSQL_USER=(mysql 계정) \
    -e MYSQL_PASSWORD=(mysql 계정의 비밀번호) \
    -e MYSQL_DB=(앱에서 사용할 database의 이름) \
    (image 이름) \
    (working directory에서 실행할 명령어)
$ docker run -dp 3000:3000 \
    -w /app \
    -v "$(pwd):/app" \
    --network todo-app \
    -e MYSQL_HOST=mysql \
    -e MYSQL_USER=root \
    -e MYSQL_PASSWORD=secret \
    -e MYSQL_DB=todos \
    node:12-alpine \
    sh -c "pnpm install && pnpm run dev"
```

### Compose file을 기반으로 container를 생성하고 실행하기

compose file의 경로가 `./docker-compose.yml`이 아닐 경우, `-f` 옵션으로 경로를 명시해야 한다.

```bash
$ docker-compose up -d
```

### Compose file을 기반으로 실행한 container의 로그 확인

```bash
# 모든 container의 로그 확인
$ docker-compose logs -f

# 개별 container의 로그 확인
$ docker-compose logs -f (compose file에 정의된 service의 이름)
```

### Compose file을 기반으로 실행한 container와 network를 중단하고 제거하기

기본적으로 이름있는 volume은 제거되지 않는다. `--volumes` 옵션을 추가해야 제거된다.

```bash
$ docker-compose down
```

### Dockerfile의 Layer caching 원리(?)

- [Dockerfile cache 및 작성 Tip](https://4orty.tistory.com/21)

### GitHub Action에서 제공하는 `Deploy to Amazon ECS` 템플릿을 사용해서 배포하는 방법

ECS의 월 요금(약 38.5 USD)이 생각보다 비싸서, 별로 좋은 방법은 아닌 듯.

1. docker image를 저장할 ECR repository(저장소)를 생성한다.
    - 일반 설정
        - 표시 여부 설정: private
        - 리포지토리 이름: 적당히 입력
        - 태그 변경 불가능: 비활성화
    - 다른 설정들은 건드리지 않고 생성 완료한다.
    - 생성된 저장소의 이름과 지역(예: ap-northeast-2)은 환경변수 `ECR_REPOSITORY`, `AWS_REGION`에 각각 할당한다.

2. ECS task definition(작업 정의), ECS cluster(클러스터), ECS service(서비스)를 생성한다.
    - 생성 예시: [Getting Started guide](https://ap-northeast-2.console.aws.amazon.com/ecs/home?region=ap-northeast-2#/firstRun) 참고
    - 생성된 서비스 이름과 클러스터 이름은 환경변수 `ECS_SERVICE`, `ECS_CLUSTER`에 각각 할당한다.

3. JSON 포맷의 ECS task definition(작업 정의)을 이 저장소에 추가한다.
    - JSON 파일의 경로와 컨테이너 이름(JSON 파일의 `containerDefinitions` 부분 참고)은 환경변수 `ECS_TASK_DEFINITION`, `CONTAINER_NAME`에 각각 할당한다.

4. IAM user를 생성하고, `AWS_ACCESS_KEY_ID`와 `AWS_SECRET_ACCESS_KEY`를 GitHub Action secret으로 추가한다.
    - 기본적으로 다음 두 가지 권한이 필요하다.
        1. `AmazonEC2ContainerRegistryPowerUser` 또는 `AmazonEC2ContainerRegistryFullAccess`
        2. `AmazonECS_FullAccess`
