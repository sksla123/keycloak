# 키클락 개인 IAM 서버 구축기
제 개인용으로 사용하기 위해 IAM 서버를 구성하고 있으며, 개인 깃허브에 기록을 위해 해당 내용을 정리하고 있습니다.

## podman으로 구성하기
기존에 docker로 구성하던 중, rootless로 구성된 container를 사용해보고 싶어 docker 대신 podman을 사용해보기로 했습니다.

### podman with docker-compose
podman은 사실상 docker랑 같다고 봐도 될 정도로 유사함. docker를 어느정도 써봤다면 사실 podman에 적응하는 것은 어렵지 않습니다.

다만, podman-compose는 아직 설치나 실행과정이 불편해 제 맘에 들진 않았습니다. 이에 일단 podman을 구성엔진으로 하는 docker-compose를 사용하기로 했고, 이렇게 사용해보니 podman에서 docker-compose 형식을 그대로 사용할 수 있어 굉장히 맘에 들었습니다.

1. podman, docker-compose 설치
```bash
sudo apt install podman docker-compose
```
2. 루트리스 podman 소켓 실행
podman 역시 root권한으로 데몬을 통해 소켓을 실행시킬 수 있지만, 아무래도 podman을 사용하기로 마음먹은 가장 큰 이유 중 하나가 rootless 모드로 구성해보자이니 만큼 여기서는 rootless 모드로 실행시켰습니다.
```bash
systemctl --user enable --now podman.socket
```
3. podman 소켓 자동 실행 등록 (생략 가능)
현재 podman의 소켓은 재부팅 되면, 다시 실행시켜야하는 상태입니다. 해당 과정을 매순간 반복하기 보단, 아래 명령어를 통해 부팅과 함께 자동 실행되도록 설정할 수 있습니다.
```bash
loginctl enable-linger $USER
```
4. podman을 메인 엔진으로 구성
아래 명령어를 통해 환경 변수(podman 소켓 위치)를 bash 실행 시 자동으로 세션에 등록되도록 유지시킬 수 있습니다. 이 과정을 통해 docker-compose가 컨테이너 구성엔진으로 podman을 사용하도록 설정할 수 있습니다.

(저는 bash 쉘을 사용하기 때문에 ~/.bashrc를 사용했지만, 다른 쉘 사용자분들은 쉘에 맞는 rc 파일을 사용해주세요.)
```bash
nano ~/.bashrc
```
```txt
# Docker host 환경변수 등록
export DOCKER_HOST="unix:$(podman info --format '{{.Host.RemoteSocket.Path}}')"
```
```bash
source ~/.bashrc
```
5. 아래 내용으로 docker-compose with podman 검증
```yaml
#docker-compose.yaml
services:
  test-alpine:
    image: alpine:latest
    command: sh -c "echo 'Podman + Docker Compose Test Successful!' && sleep 3600"
    restart: "no"
```
```bash
docker-compose up -d
podman ps -a // alpine 컨테이너가 존재하는지 확인
docker-compose down -v
```
> 만약, podman을 완전히 docker처럼 사용하고 싶으시다면, alias 등록을 통해 docker 처럼 사용할 수 있습니다.

## 키클락 컨테이너 설치 및 실행
1. docker-compose 파일 구성
> docker-compose.example.yaml 참조
```yaml
#docker-compose.yaml
services:
  postgres:
    image: postgres:15-alpine
    container_name: keycloak-db
    volumes:
      - ./pgdata:/var/lib/postgresql/data ## 데이터 영구 유지 및 관리를 위한 폴더 마운트 (가상 볼륨 마운트도 괜찮습니다.)
    environment:
      POSTGRES_DB: DB 테이블 명 입력해주세요.
      POSTGRES_USER: DB 사용자 명 입력해주세요.
      POSTGRES_PASSWORD: DB와 통신할 때 사용되는 비밀번호를 입력해주세요.
    restart: always
    networks:
      - keycloak-net

  keycloak:
    image: quay.io/keycloak/keycloak:26.4.0 # 보안/인증과 관련된 모듈은 항상 최신 stable 버전을 사용하는 것이 좋습니다.
    container_name: keycloak-server
    depends_on:
      - postgres
    environment:
      # --- 데이터베이스 연결 설정 ---
      KC_DB: postgres
      KC_DB_URL_HOST: postgres
      KC_DB_URL_DATABASE: 입력한 DB 테이블 명 입력해주세요.
      KC_DB_USERNAME: 입력한 DB 사용자 명 입력해주세요.
      KC_DB_PASSWORD: 입력한 DB 통신에 사용되는 비밀번호를 입력해주세요.

      # --- 키클락 관리자 설정 ---
      KC_BOOTSTRAP_ADMIN_USERNAME: 임시 어드민 계정 이름을 입력해주세요.
      KC_BOOTSTRAP_ADMIN_PASSWORD: 임시 어드민 비밀번호를 입력해주세요.

      # --- http 설정 ---
      KC_HTTP_ENABLED: "true" # http 모드 실행 여부 (false 시 https 모드로 실행됩니다.)

      # --- 도메인 설정 ---
      KC_HOSTNAME: "your.example.com" # 도메인을 가지신 분들은 도메인을 작성해주세요. 

      # --- 리버스 프록시 설정 ---
      ## 해당사항이 없으신 분들은 아래 내용을 없애도 괜찮습니다.
      KC_PROXY_HEADERS: "xforwarded"
      KC_HTTP_RELATIVE_PATH: "/your/private/relative/path" # 리버스 프록시 상대 경로 입력. (루트에서 받는다면 지워도 괜찮습니다.)

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
2. 컨테이너 실행
```
docker-compose up -d
```
> 컨테이너 종료
```bash
docker-compose down
```

## admin 등록/임시 admin 삭제
키클록에서는 보안을 위해 임시 admin를 삭제할 것을 권고하고 있습니다. 임시 마스터를 삭제하기 위해서는 현재 masters realm에서 우선 새로운 admin을 등록해야 합니다.
> *realm이 뭔가요?
> > realm이란 조직에 가깝다. 한 realm에 등록된 유저들은 realm에 등록된 모든 client에 로그인할 수 있다.
> >
> > 이를 조직에 확대시켜 예를 들어보면, 카카오 조직(kakao realm)에 들어간 모든 회원에 대해 카카오에서 하는 모든 서비스에 로그인 할 수 있도록 하는 것이다.
> >
> > (물론 실제로 이렇게 운영되진 않는다.) 
> >
> > 다만, 유저는 어떤 그룹에 속해있는지, 어떤 role을 부여받았는지, 해당 유저가 가지고 있는 세부적인 메타 정보에 따라 조직 내에서도 다른 서비스를 제공하게 할 수도 있다.

1. 먼저 realm이 master(=keycloak)인지 확인합니다.
<img width="557" height="332" alt="Image" src="https://github.com/user-attachments/assets/7ca067aa-0e80-4d03-a28c-6878ec46a551" />

(사이트 왼쪽 상단을 확인하면, 본인의 현재 realm(current realm)이 나온다.)
> 만약 realm이 master가 아니라면, 왼쪽 상단의 Manage realms를 누른 뒤, master realm을 선택해 눌러줍니다.
2. users 탭으로 이동 후, add user를 눌러줍니다.
<img width="899" height="267" alt="image" src="https://github.com/user-attachments/assets/509170d7-3552-447d-b537-2acd4f9ca75d" />

3. 새로운 사용자 이름을 작성 후 하단의 create 버튼을 눌러줍니다.
<img width="797" height="889" alt="image" src="https://github.com/user-attachments/assets/abcfb324-eb48-4918-8998-9bf3fec685c4" />

4. 새롭게 뜬 유저 관리 창에서, credentials 탭으로 이동 후 비밀번호를 설정해줍니다.
<img width="1056" height="503" alt="image" src="https://github.com/user-attachments/assets/0ca21c32-3bc5-470e-8221-456c5243f01a" />

> admin 계정이니 만큼 전 temporaly는 꺼둔 상태입니다. (개인 호불호에 따라 결정하시면 될 듯 합니다.)
5. 유저 관리 창에서, Role Mapping 탭으로 이동 후 assign role을 눌러 realm roles - admin을 부여해줍니다.
<img width="1163" height="632" alt="image" src="https://github.com/user-attachments/assets/5c2a8767-8cf9-4d6e-bd7a-56faff5a14bd" />

6. sign out 후 새롭게 만든 id로 로그인합니다.
7. 다시 현재 realm이 master인지 확인하고 user 탭으로 이동 후, 기존 임시 admin 계정을 삭제합니다.
<img width="1688" height="301" alt="image" src="https://github.com/user-attachments/assets/705c1937-fecd-432d-9c13-0d7331acf6fd" />

## realm 생성
(힘이 빠져서 내일 적겠습니다.)
