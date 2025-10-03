# 키클락 개인 IAM 서버 구축기
제 게인용으로 사용하기 위해 IAM 서버를 구성하고 있습니다.

## podman으로 구성하기
기존에 docker로 구성하던 중, rootless로 구성된 container를 사용해보고 싶어, docker 대신 podman으로 변경함. 

### podman with docker-compose
podman은 사실상 docker랑 같다고 봐도 될 정도로 유사함. docker를 어느정도 써봤다면 사실 podman에 적응하는 것은 어렵지않음.

다만, podman-compose는 아직 좀 불편하다고 느낌. 일단은 podman을 구성엔진으로 하는 docker-compose를 사용하기로 마음먹음.이렇게 사용하니 podman에서 docker-compose 형식을 그대로 사용할 수 있음.

1. podman, docker-compose 설치
```bash
sudo apt install podman docker-compose
```
1. 루트리스 podman 소켓 실행
podman 역시 root권한을 가지고 소켓을 실행시킬 수 있지만, 아무래도 podman을 쓰는 큰 이유 중 하나가 rootless 모드인 만큼 rootless 모드로 실행시킨다.
```bash
systemctl --user enable --now podman.socket
```
2. 자동 실행
podman의 소켓은 현재 재부팅 되면 수동으로 실행시켜야 하지만, 해당 과정을 일일히 하긴 귀찮기 때문에 아래 명령어를 통해 자동 실행을 보장시켜 놓는다.
```bash
loginctl enable-linger $USER
```
3. podman을 메인 엔진으로 구성
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
4. 아래 내용으로 docker-compose with podman 검증
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


