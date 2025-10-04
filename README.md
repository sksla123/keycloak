# 키클락 개인 IAM 서버 구축기
제 개인용으로 사용하기 위해 IAM 서버를 구성하고 있습니다.

## podman으로 구성하기
기존에 docker로 구성하던 중, rootless로 구성된 container를 사용해보고 싶어, docker 대신 podman으로 변경함. 

### podman with docker-compose
podman은 사실상 docker랑 같다고 봐도 될 정도로 유사함. docker를 어느정도 써봤다면 사실 podman에 적응하는 것은 어렵지않음.

다만, podman-compose는 아직 좀 불편하다고 느낌. 일단은 podman을 구성엔진으로 하는 docker-compose를 사용하기로 마음먹음.이렇게 사용하니 podman에서 docker-compose 형식을 그대로 사용할 수 있음.

1. podman, docker-compose 설치
```bash
sudo apt install podman docker-compose
```
2. 루트리스 podman 소켓 실행
podman 역시 root권한을 가지고 소켓을 실행시킬 수 있지만, 아무래도 podman을 쓰는 큰 이유 중 하나가 rootless 모드인 만큼 rootless 모드로 실행시킨다.
```bash
systemctl --user enable --now podman.socket
```
3. 자동 실행
podman의 소켓은 현재 재부팅 되면 수동으로 실행시켜야 하지만, 해당 과정을 일일히 하긴 귀찮기 때문에 아래 명령어를 통해 자동 실행을 보장시켜 놓는다.
```bash
loginctl enable-linger $USER
```
4. podman을 메인 엔진으로 구성
아래 명령어를 통해 환경 변수(podman 소켓 위치)를 bash 실행 시 자동으로 세션에 등록되도록 유지시킨다.
*각 쉘 종류에 맞는 rc 파일을 사용해주세요. (zsh -> ~/.zshrc)
```bash
nano ~/.bashrc
```
```txt
export DOCKER_HOST="unix:$(podman info --format '{{.Host.RemoteSocket.Path}}')"
```

```bash
source ~/.bashrc
```
6. 아래 내용으로 docker-compose with podman 검증
```yaml
#docker-compose.yaml
services:
  test-alpine:
    image: alpine:latest
    command: sh -c "echo 'Podman + Docker Compose Test Successful!' && sleep 3600"
    restart: "no"
```
```bash
docker-compose up
//ctrl+x(종료)
podman ps -a // alpine 컨테이너가 존재하는지 확인
docker-compose down
```

### 키클락 docker-compose 파일 구성
여기 있는 docker-compose.yaml.example 참조
```yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: keycloak-db
    volumes:
      - ./pgdata:/var/lib/postgresql/data ## data 영구 유지를 위함
    environment:
      POSTGRES_DB: KC-table
      POSTGRES_USER: KC-user
      POSTGRES_PASSWORD: 비밀번호는 알아서 만들어주세요!
    restart: always
    networks:
      - keycloak-net

  keycloak:
    image: quay.io/keycloak/keycloak:26.4.0 # 보안/인증과 관련된 모듈은 항상 최신 stable 버전을 사용하도록 노력하자.
    container_name: keycloak-server
    depends_on:
      - postgres
    environment:
      # --- 데이터베이스 연결 설정 ---
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: KC-table
      KC_DB_USERNAME: KC-user
      KC_DB_PASSWORD: DB에 설정된 비밀번호와 일치시켜주세요!

      # --- 키클락 관리자 설정 ---
      KC_BOOTSTRAP_ADMIN_USERNAME: 임시 어드민 계정 이름
      KC_BOOTSTRAP_ADMIN_PASSWORD: 임시 어드민 비밀번호

      # --- 리버스 프록시 및 HTTP 설정 ---
      ## 저는 리버스 프록시 뒤에서 동작시키고 있기 때문에, 리버스 프록시 설정을 해두었습니다만, 해당사항이 없으신 분들은 아래 내용을 없애주세요!
      KC_PROXY_HEADERS: "xforwarded"
      KC_HTTP_ENABLED: "true" # http 모드 실행 (false 시 https 모드로 실행됨.) ## 저는 https로 동작되는 리버스 프록시 뒤에서 http로 통신하는 형태로 만들었습니다.
      KC_HOSTNAME: "your.example.com" # 본인 도메인 입력
      KC_HTTP_RELATIVE_PATH: "/your-private-relative-path" #본인이 리버스프록시에서 어떤 path에서 받는지 적으시면 됩니다.

    ports:
      - "8080:8080"
    restart: always
    networks:
      - keycloak-net
    command:
      - "start"

networks:
  keycloak-net:
    driver: bridge
```
## admin 등록/임시 admin 삭제
키클록에서는 보안을 위해 임시 admin를 삭제할 것을 권고 하고 있다. 임시 마스터를 삭제하기 위해 현재 masters realm에서 우선 새로운 admin을 등록한다.
> *realm이 뭔가요?
> > realm이란 조직에 가깝다. 한 realm에 등록된 유저들은 realm에 등록된 모든 client에 로그인할 수 있다.
> >
> > 이를 조직에 확대시켜 예를 들어보면, 카카오 조직(kakao realm)에 들어간 모든 회원에 대해 카카오에서 하는 모든 서비스에 로그인 할 수 있도록 하는 것이다.
> >
> > (물론 실제로 이렇게 운영되진 않는다.) 
> >
> > 다만, 유저는 어떤 그룹에 속해있는지, 어떤 role을 부여받았는지, 해당 유저가 가지고 있는 세부적인 메타 정보에 따라 조직 내에서도 다른 서비스를 제공하게 할 수도 있다.

1. 먼저 realm이 master인지 확인한다.
