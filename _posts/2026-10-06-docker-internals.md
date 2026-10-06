---
title: "Docker 내부 동작 원리: 컨테이너는 결국 프로세스다"
date: 2026-10-06 21:00:00 +0900
categories: [DevOps, Docker]
tags: [docker, container, linux, namespace, cgroup, overlayfs, jvm]
---

Docker를 쓰면서 `docker run`이 실제로 무슨 일을 하는지는 잘 몰랐다. 오늘은 이론을 정리하고, 리눅스 머신에서 직접 확인해 봤다. 결론부터 말하면 컨테이너는 가상 머신이 아니라 **리눅스 커널 기능으로 격리·제한된 호스트의 프로세스**다.

> **실습 환경**: Windows + WSL2 Ubuntu에 Docker Engine을 직접 설치했다. Docker Desktop은 숨은 VM 안에서 컨테이너를 돌려서 호스트 쪽에서 프로세스와 `/proc`이 보이지 않기 때문에, WSL Integration을 끄고 진행했다.
{: .prompt-info }

## 1. VM과 컨테이너의 차이

핵심은 **커널을 누가 갖고 있는가**다.

| 구분 | VM | 컨테이너 |
|---|---|---|
| 가상화 대상 | 하드웨어 (CPU, 메모리, 디스크) | 없음. 프로세스의 시야와 자원만 제한 |
| 커널 | VM마다 게스트 커널을 따로 부팅 | 호스트 커널 하나를 모두 공유 |
| 구조 | 하드웨어 → 호스트 OS → 하이퍼바이저 → 게스트 커널 → 앱 | 하드웨어 → 리눅스 커널 → 컨테이너 런타임 → 앱(프로세스) |
| 시작 속도 | 초~분 (OS 부팅) | ms 단위 (프로세스 생성) |
| 격리 경계 | 하이퍼바이저 (강함) | 커널 (커널 취약점 시 탈출 가능) |

커널을 공유하기 때문에 생기는 현상들이 있다.

- 컨테이너 안의 프로세스는 호스트에서 `ps -ef`로 그대로 보인다. 컨테이너 안에서는 PID 1, 호스트에서는 일반 PID다.
- 컨테이너 안에서 `uname -r`을 치면 호스트와 같은 커널 버전이 나온다.
- 같은 이유로 리눅스 호스트에서 Windows 컨테이너는 실행할 수 없다.

### 하이퍼바이저

VM을 만들고 관리하는 소프트웨어다. 물리 자원을 쪼개 각 VM에게 가짜 하드웨어를 보여주고, 게스트 OS의 하드웨어 접근을 중재한다. Intel VT-x, AMD-V 같은 CPU 가상화 기능을 사용한다.

- **Type 1 (bare-metal)**: 하드웨어 위에 직접 설치. VMware ESXi, Xen, KVM, Hyper-V. 서버·클라우드용.
- **Type 2 (hosted)**: 일반 OS 위에서 앱처럼 실행. VirtualBox, VMware Workstation. 개인 PC용.

AWS EC2 인스턴스 자체도 Nitro 하이퍼바이저(KVM 기반) 위의 VM이다. 그래서 EC2에서 Docker를 쓰면 `물리 서버 → 하이퍼바이저 → EC2 VM → 컨테이너` 구조가 된다. VM과 컨테이너는 대체 관계가 아니라 겹쳐 쓰는 경우가 대부분이다.

### 맥·윈도우의 Docker Desktop

macOS 커널(XNU)에는 namespace도 cgroup도 없다. 그래서 Docker Desktop은 **경량 리눅스 VM을 자동으로 만들고** 그 안에서 dockerd와 컨테이너를 돌린다.

- 맥: Apple Virtualization.framework로 VM 생성
- 윈도우: WSL2 사용. WSL2 자체가 Hyper-V 기반 경량 VM에서 실제 리눅스 커널을 돌리는 구조
- 리눅스: VM 없이 호스트 커널을 바로 사용

이 구조를 알면 평소에 겪던 현상이 설명된다.

- Docker Desktop 설정의 CPU/메모리 값은 숨은 VM의 사양이다. 컨테이너는 이 한도 안에서만 자원을 쓴다.
- 바인드 마운트(`-v ./src:/app`)가 리눅스보다 느린 건 파일 접근이 VM 경계를 넘기 때문이다.
- `localhost:8080`으로 접속되는 건 Docker Desktop이 맥 ↔ VM 사이 포트를 포워딩해 주는 덕분이다.

## 2. 컨테이너를 구성하는 세 가지 커널 기능

Docker 고유 기술이 아니라 리눅스 커널에 원래 있던 기능의 조합이다.

| 기능 | 역할 | 질문 |
|---|---|---|
| Namespace | 격리 | 무엇이 보이는가 |
| Cgroups | 제한 | 얼마나 쓸 수 있는가 |
| Union FS (OverlayFS) | 파일시스템 구성 | 파일을 어떻게 겹쳐 보여주는가 |

### Namespace — 무엇이 보이는가

프로세스가 볼 수 있는 시스템 자원의 범위를 분리한다.

| Namespace | 분리 대상 |
|---|---|
| PID | 프로세스 번호 체계 (컨테이너 안 첫 프로세스가 PID 1) |
| NET | 네트워크 인터페이스, IP, 라우팅 테이블, 포트 공간, loopback |
| MNT | 파일시스템 마운트 트리 |
| UTS | 호스트네임 |
| IPC | 공유 메모리, 메시지 큐 |
| USER | UID/GID 매핑 |

**NET namespace와 포트.** 포트 번호만 분리되는 게 아니라 네트워크 스택 전체가 컨테이너별로 따로 생긴다.

- redis 컨테이너 두 개가 각자 내부에서 6379를 써도 충돌하지 않는다.
- 충돌은 호스트 쪽에서만 일어난다. 그래서 `-p 6379:6379`, `-p 6380:6379`처럼 호스트 포트만 다르게 주면 된다.
- `--network host`를 주면 호스트의 NET namespace를 그대로 쓰므로 포트가 실제로 공유되고 충돌한다.

자주 겪는 함정도 여기서 나온다.

- 컨테이너 안의 `localhost`는 컨테이너 자기 자신이다. Spring 컨테이너에서 `localhost:3306`으로 MySQL 컨테이너에 붙을 수 없고, Compose 서비스 이름(`mysql:3306`)을 써야 한다.
- 앱이 `127.0.0.1`에만 바인딩하면 `-p`로 열어도 외부에서 접속이 안 된다. 외부 패킷은 컨테이너의 `eth0`으로 들어오므로 `0.0.0.0`으로 바인딩해야 한다.

### Cgroups — 얼마나 쓸 수 있는가

CPU, 메모리, I/O, 프로세스 수를 제한하고 측정한다.

- `docker run -m 512m`은 결국 `/sys/fs/cgroup/.../memory.max` 파일에 512MB를 써넣는 것이다.
- 한도를 넘으면 커널 OOM killer가 프로세스를 종료하고, 종료 코드는 **137**(128 + SIGKILL 9)이 된다.

### Union Filesystem (OverlayFS) — 파일을 어떻게 겹쳐 보여주는가

- **이미지**: 읽기 전용 레이어 여러 장의 스택.
- **컨테이너**: 이미지 레이어 위에 **쓰기 가능 레이어(container layer)** 한 장을 얹은 것.
- OverlayFS가 `lowerdir`(이미지 레이어)와 `upperdir`(쓰기 레이어)를 겹쳐 `merged` 디렉터리 하나로 보여준다.
- **Copy-on-Write**: 이미지에 있던 파일을 수정하면 그 파일을 upperdir로 복사한 뒤 수정한다.

레이어는 공유된다.

- 같은 이미지로 컨테이너를 여러 개 띄워도 이미지 레이어는 디스크에 한 벌만 있다.
- 다른 이미지라도 같은 베이스 레이어(예: debian)는 공유된다. 레이어가 sha256 해시로 식별되기 때문이다.
- 공유되는 건 읽기 전용 원본뿐이다. 각 컨테이너가 쓴 데이터는 자기 쓰기 레이어에만 있다.
- `docker rm`을 하면 쓰기 레이어도 지워진다. 데이터를 남기려면 **볼륨**으로 경로를 쓰기 레이어 밖(호스트 디스크)으로 빼야 한다.

## 3. 이미지 빌드부터 컨테이너 실행까지

자바의 `.java → javac → .class → JVM 로딩 → 실행` 흐름에 대응시켜 보면 이해가 쉽다.

```text
Dockerfile ──(docker build)──> Image ──(push/pull)──> Registry
                                  │
                           (docker run)
                                  ▼
docker CLI → dockerd → containerd → containerd-shim → runc
                                                       │
                     namespace 생성 + cgroup 설정 + overlay 마운트
                                                       ▼
                                          ENTRYPOINT 프로세스 실행
```

| 단계 | 자바 대응 | 내용 |
|---|---|---|
| 빌드 | `javac` | Dockerfile 각 명령의 결과(파일시스템 변경분)를 레이어로 저장. 이미지 = 레이어 목록 + 설정 JSON(ENV, ENTRYPOINT 등) |
| 이미지 포맷 | `.class` | OCI 표준. Docker로 만든 이미지를 Podman, Kubernetes가 그대로 사용 |
| 실행 요청 | JVM 기동 | CLI → dockerd → containerd → shim → runc |
| 컨테이너 생성 | 클래스 로딩·메모리 할당 | runc가 namespace·cgroup·루트 파일시스템을 준비하고 프로세스 실행 |
| 실행 중 | 힙/스택에서 실행 | 커널 입장에선 평범한 프로세스. 시스템콜을 호스트 커널이 직접 처리하므로 CPU 성능 손실이 거의 없음 |

### 각 구성 요소의 역할

- **docker CLI**: 클라이언트. 명령을 HTTP 요청으로 바꿔 `/var/run/docker.sock`으로 보낸다.
- **dockerd**: 항상 떠 있는 데몬. 이미지, 네트워크, 볼륨을 관리하고 컨테이너 생명주기는 containerd에 맡긴다.
- **containerd**: 컨테이너 생명주기 관리. 컨테이너마다 shim을 하나씩 띄운다.
- **containerd-shim**: runc가 끝난 뒤 컨테이너 프로세스의 부모로 남아 종료 코드와 stdout을 관리한다. 덕분에 dockerd를 재시작해도 컨테이너가 살아 있을 수 있다.
- **runc**: 저수준 런타임. 컨테이너를 실제로 만들고 종료된다.

### runc가 하는 일

1. `clone()` 시스템콜에 `CLONE_NEWPID | CLONE_NEWNET | CLONE_NEWNS ...` 플래그를 줘서 새 namespace 안에 자식 프로세스를 만든다.
2. 그 프로세스를 cgroup에 등록한다.
3. OverlayFS의 merged 디렉터리를 `pivot_root`로 새 루트(`/`)로 바꾼다.
4. `exec()`로 ENTRYPOINT(예: `java -jar app.jar`)를 실행한다.
5. runc는 종료되고 shim이 부모로 남는다.

### `docker run`은 API 호출 세 번이다

1. (이미지가 없으면) `POST /images/create` — pull
2. `POST /containers/create` — 생성
3. `POST /containers/{id}/start` — 시작

그래서 CLI 없이도 dockerd를 조작할 수 있다.

```bash
curl --unix-socket /var/run/docker.sock http://localhost/containers/json
```

### docker.sock과 보안

- Jenkins 컨테이너에 `-v /var/run/docker.sock:/var/run/docker.sock`을 마운트하면 Jenkins 안의 docker CLI가 호스트 dockerd에 요청을 보내 호스트에 컨테이너를 띄운다.
- dockerd는 root로 실행된다. 따라서 **docker.sock 접근 권한은 사실상 호스트 root 권한**이다. 사용자를 `docker` 그룹에 넣는 건 신중해야 한다.

## 4. 네트워크 연결 구조

- 컨테이너의 NET namespace는 처음엔 완전히 고립되어 있다.
- Docker가 **veth pair**(가상 랜선)를 만들어 한쪽은 컨테이너 안 `eth0`, 다른 쪽은 호스트의 `docker0` 브리지에 꽂는다.
- 컨테이너끼리는 이 브리지를 통해 통신한다.
- `-p 8080:8080`은 호스트 iptables에 DNAT 규칙을 추가하는 것이다. 호스트 8080으로 들어온 패킷을 컨테이너 IP로 넘긴다.
- Compose에서 서비스 이름으로 접속되는 이유는 사용자 정의 네트워크에서 Docker가 `127.0.0.11`에 내장 DNS 서버를 띄워 서비스 이름을 컨테이너 IP로 바꿔주기 때문이다.

## 5. 데몬

터미널과 연결되지 않은 채 백그라운드에 상주하면서 요청이나 이벤트를 기다렸다가 처리하는 프로세스다.

- 일반 프로그램(`ls`): 시작 → 작업 → 종료
- 데몬: 시작 → 대기 → 요청 처리 → 대기 (로그아웃해도 유지)
- 이름 끝에 `d`를 붙이는 관례가 있다: `sshd`, `crond`, `mysqld`, `dockerd`, `systemd`
- 보통 부팅 시 systemd가 띄우고 관리한다. `ps -ef`에서 TTY가 `?`인 프로세스들이다.
- 8080 포트에서 요청을 기다리는 Spring Boot 서버도 본질적으로 데몬이다.

### Gradle 데몬과 `--no-daemon`

- Gradle은 빌드가 끝나도 JVM을 종료하지 않고 데몬으로 남긴다. 다음 빌드에서 JVM 기동·클래스 로딩 비용 없이 JIT로 데워진 코드와 캐시를 재사용해서 빨라진다.
- `./gradlew`(클라이언트)와 Gradle 데몬(서버)의 관계는 `docker` CLI와 dockerd의 관계와 같다.
- `--no-daemon`은 이번 빌드 전용 JVM을 띄우고 끝나면 종료한다.
- CI나 Dockerfile에서 쓰는 이유는 일회성 환경이라 데몬이 재사용될 일이 없고 메모리만 차지하기 때문이다.

```dockerfile
RUN ./gradlew bootJar --no-daemon
```

## 6. 원리에서 나오는 실무 포인트

### JVM과 cgroup

- JDK 8u191 이전 JVM은 cgroup을 인식하지 못해서 호스트 전체 메모리 기준으로 힙을 잡았고, 그 결과 OOM kill을 당했다.
- 지금 JVM은 컨테이너를 인식하지만 기본 최대 힙이 메모리 한도의 **25%**밖에 안 된다.
- 보통 `-XX:MaxRAMPercentage=75` 정도로 명시한다.
- 힙 말고도 메타스페이스, 스레드 스택, 다이렉트 버퍼가 같은 cgroup 한도를 나눠 쓴다는 점을 기억해야 한다.

### PID 1과 시그널

- `docker stop`은 PID 1에 SIGTERM을 보내고, 10초 안에 안 끝나면 SIGKILL을 보낸다.
- PID 1은 커널이 특별 취급해서, 핸들러를 등록하지 않은 시그널은 무시된다.
- shell form(`ENTRYPOINT java -jar app.jar`)은 PID 1이 `/bin/sh`가 되고, sh는 시그널을 자바로 넘기지 않아서 graceful shutdown이 안 된다.
- exec form(`ENTRYPOINT ["java", "-jar", "app.jar"]`)을 써야 자바가 PID 1이 되어 SIGTERM을 받는다.
- `--init` 옵션을 주면 시그널을 전달해 주는 작은 init 프로세스가 PID 1로 들어간다.

### 레이어 캐시 순서

- 한 레이어가 바뀌면 그 아래 레이어 캐시는 모두 무효화된다.
- 잘 안 바뀌는 것(`build.gradle`, 의존성 다운로드)을 먼저, 자주 바뀌는 것(소스 코드 `COPY`)을 나중에 둔다.
- Spring Boot의 layered jar도 같은 원리를 이용한다.

## 7. 실습

리눅스 머신에서 진행했다. Docker Desktop 환경에서는 `/proc`, `/sys/fs/cgroup`을 호스트에서 직접 볼 수 없어서 아래 실습 대부분이 안 된다.

### 7-1. 컨테이너는 호스트의 프로세스다

```bash
docker run -d --name redis1 redis
ps -ef | grep redis-server
docker inspect -f '{{.State.Pid}}' redis1
ps -ef --forest | grep -B2 redis-server
```

확인할 것: 호스트에서 redis-server가 일반 프로세스로 보이는지, `docker inspect`의 PID와 같은지, 부모가 `containerd-shim`인지.

```text
$ ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
root           1       0  0 10:13 ?        00:00:00 /sbin/init
root         221       1  3 10:13 ?        00:00:05 /usr/bin/containerd
root         308       1  0 10:13 ?        00:00:01 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
root         939       1  0 10:16 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id c78ecae...
999          964     939  3 10:16 ?        00:00:00 redis-server *:6379
...

$ docker inspect -f '{{.State.Pid}}' redis1
964

$ ps -ef --forest | grep -B2 redis-server
root         939       1  0 10:16 ?        00:00:00 /usr/bin/containerd-shim-runc-v2 -namespace moby -id c78ecaef2c56...
999          964     939  0 10:16 ?        00:00:00  \_ redis-server *:6379
```

- 컨테이너 안의 redis-server가 호스트의 `ps -ef`에 **PID 964**로 그대로 보이고, `docker inspect`가 알려준 PID와 정확히 같다. 컨테이너는 정말 호스트의 프로세스였다.
- 부모(PPID 939)는 `containerd-shim-runc-v2`다. runc는 컨테이너를 만들고 이미 종료되어 목록에 없다.
- shim의 부모는 containerd가 아니라 **PID 1(init)**이다. shim이 containerd에서 떨어져 나와 독립적으로 살아 있기 때문에 dockerd나 containerd를 재시작해도 컨테이너가 죽지 않는다.
- `dockerd`, `containerd`의 TTY가 `?`다. 터미널과 연결되지 않은 데몬이라는 뜻이다.
- redis-server의 UID가 `999`로 보인다. redis 이미지가 컨테이너 안에서 만든 `redis` 사용자의 UID인데, 호스트에는 그 이름의 사용자가 없어서 숫자로만 표시된다. USER namespace를 따로 쓰지 않으면 UID는 호스트와 그대로 공유된다.

### 7-2. 커널 공유

```bash
uname -r
docker run --rm alpine uname -r
```

확인할 것: 두 값이 같은지.

```text
$ uname -r
6.18.33.2-microsoft-standard-WSL2

$ docker run --rm alpine uname -r
6.18.33.2-microsoft-standard-WSL2
```

- Ubuntu(WSL2)와 alpine 컨테이너가 **같은 커널 버전**을 출력한다. alpine 이미지에는 커널이 들어 있지 않고 `uname` 같은 사용자 영역 파일만 있다. `uname`은 시스템콜로 커널에 버전을 묻는데, 그 질문을 받는 건 호스트 커널 하나뿐이다.
- 배포판이 Ubuntu든 alpine이든 결국 같은 커널 위에서 도는 프로세스다. 그래서 리눅스 호스트에서 Windows 컨테이너를 돌릴 수 없다.
- 커널 이름의 `microsoft-standard-WSL2`가 재밌다. 이 커널은 Microsoft가 빌드한 리눅스 커널이고, Hyper-V 기반 경량 VM 안에서 돈다. 즉 내 환경은 `Windows → Hyper-V → WSL2 VM(리눅스 커널) → 컨테이너` 구조다. 1장에서 말한 "VM과 컨테이너는 겹쳐 쓴다"가 내 PC에서도 그대로였다.

### 7-3. Namespace 확인

```bash
PID=$(docker inspect -f '{{.State.Pid}}' redis1)
sudo ls -l /proc/self/ns
sudo ls -l /proc/$PID/ns
sudo nsenter -t $PID -n ip addr
sudo nsenter -t $PID -n ss -lntp
```

확인할 것: 호스트와 컨테이너의 namespace ID(`net:[4026...]`)가 다른지, 컨테이너 NET namespace에 `lo`와 `eth0`만 있고 6379가 열려 있는지.

| namespace | 호스트(WSL Ubuntu) | redis1 컨테이너 |
|---|---|---|
| cgroup | 4026531835 | 4026532230 |
| ipc | 4026532208 | 4026532228 |
| mnt | 4026532219 | 4026532222 |
| net | 4026531833 | 4026532231 |
| pid | 4026532221 | 4026532229 |
| time | 4026531834 | 4026532293 |
| **user** | **4026531837** | **4026531837** |
| uts | 4026532220 | 4026532226 |

- `user`를 빼고 전부 다르다. 프로세스마다 namespace 링크가 있고, 두 프로세스의 ID가 같으면 같은 namespace를 공유한다는 뜻이다.
- `user`만 같은 이유는 Docker가 기본적으로 USER namespace를 분리하지 않기 때문이다. 7-1에서 redis-server의 UID `999`가 호스트에서 그대로 보였던 것과 같은 이야기다.
- 이 숫자는 namespace 전용 가상 파일시스템(nsfs)의 **inode 번호**다. 커널은 이런 가상 inode를 `0xF0000000`(= 4026531840)부터 동적으로 나눠 주고, 부팅 때 만들어지는 초기 namespace에는 그 바로 아래 고정 번호(user `...837`, cgroup `...835`, time `...834` 등)를 쓴다. 그래서 대부분 `4026...`으로 시작한다.
- WSL 쪽의 pid·mnt·ipc·uts가 고정 번호가 아닌 것도 흥미롭다. WSL2는 배포판(Ubuntu)마다 따로 namespace를 만들어 돌리기 때문에, 내가 "호스트"라고 부른 Ubuntu도 사실 VM 안에서 부분적으로 격리된 환경이었다.

```text
$ sudo nsenter -t $PID -n ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 ...
    inet 127.0.0.1/8 scope host lo
2: eth0@if6: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 ...
    inet 172.17.0.2/16 brd 172.17.255.255 scope global eth0

$ sudo nsenter -t $PID -n ss -lntp
State   Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
LISTEN  0       511           0.0.0.0:6379       0.0.0.0:*      users:(("redis-server",pid=964,fd=18))
LISTEN  0       511              [::]:6379          [::]:*      users:(("redis-server",pid=964,fd=19))
```

`nsenter -t <PID> -n`은 그 프로세스의 NET namespace에만 들어가서 명령을 실행한다. 컨테이너 이미지에 `ip`, `ss`가 없어도 호스트의 도구로 컨테이너 네트워크를 볼 수 있다.

- 인터페이스는 `lo`와 `eth0` 두 개뿐이다. 호스트의 인터페이스는 하나도 보이지 않는다.
- **`lo` (loopback)**: 자기 자신과 통신하는 가상 인터페이스로, `127.0.0.1`(= `localhost`)이 여기 붙어 있다. NET namespace마다 `lo`가 따로 생기기 때문에 **컨테이너 안의 `localhost`는 컨테이너 자기 자신**이다.
- **`eth0@if6`**: 외부와 통신하는 인터페이스. 정체는 4장에서 말한 **veth pair**의 컨테이너 쪽 끝이다. `@if6`은 짝이 되는 반대쪽 끝이 호스트의 6번 인터페이스(`vethXXXX`)라는 뜻이고, 그쪽은 `docker0` 브리지에 꽂혀 있다. IP `172.17.0.2/16`은 `docker0` 브리지의 기본 대역에서 받은 주소다.
veth의 반대쪽 끝은 호스트에서 확인할 수 있다.

```text
$ ip addr | grep -A1 "^6:"
6: vethec7c5cc@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether ea:a6:e7:28:6e:87 brd ff:ff:ff:ff:ff:ff link-netnsid 0

$ ip link show master docker0
6: vethec7c5cc@if2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP mode DEFAULT group default
```

- 컨테이너 쪽은 `2: eth0@if6`, 호스트 쪽은 `6: vethec7c5cc@if2`다. 서로가 서로의 번호를 가리키고 있다. 랜선 한 가닥의 양 끝이다.
- 호스트 쪽 끝에 `master docker0`이 붙어 있다. 이 veth가 `docker0` 브리지에 꽂혀 있다는 뜻이다.
- `docker0`에 꽂힌 인터페이스는 이것 하나뿐이다. 지금 떠 있는 컨테이너가 redis1 하나라서 그렇다. 컨테이너를 하나 더 띄우면 veth도 하나 더 생긴다.

```text
[redis1 NET namespace]                  [호스트 NET namespace]
  eth0 (172.17.0.2) ── veth pair ── vethec7c5cc ── docker0 브리지
```

- redis가 `0.0.0.0:6379`로 열려 있다. 모든 인터페이스에서 받겠다는 뜻이라 `eth0`으로 들어오는 외부 패킷도 받을 수 있다. `127.0.0.1:6379`로만 열려 있었다면 `-p`로 포트를 열어도 밖에서 접속할 수 없다.

### 7-4. 호스트 포트 충돌

```bash
docker run -d --name r1 -p 6379:6379 redis
docker run -d --name r2 -p 6379:6379 redis   # 실패
docker run -d --name r3 -p 6380:6379 redis   # 성공
```

확인할 것: 두 번째 명령에서 `port is already allocated` 에러가 나는지. 컨테이너 내부 포트는 둘 다 6379인데 호스트 포트만 다르면 문제없다.

```text
$ docker run -d --name r1 -p 6379:6379 redis
5f417fbc2bffb3787df3fccff5a9c2f9d3900699ea8fa2cdc66785462463673f

$ docker run -d --name r2 -p 6379:6379 redis
6461b3efa3cca17383d039bcffadaf8bbd81df78ac0c3769a74964ee8589e019
docker: Error response from daemon: failed to set up container networking: driver failed programming
external connectivity on endpoint r2 (...): Bind for 0.0.0.0:6379 failed: port is already allocated

$ docker run -d --name r3 -p 6380:6379 redis
b9e1ad62da83eea09b2effe1b6708ab8665ef70463ef7e3dc0b6885955799885
```

- r1, r2, r3 모두 **컨테이너 안에서는 6379**를 쓴다. 각자 NET namespace가 따로 있으니 내부 포트끼리는 충돌할 수 없다.
- 실패한 건 r2의 **호스트 쪽 6379**다. 에러 메시지도 컨테이너가 아니라 `0.0.0.0:6379`(호스트) 바인딩에 실패했다고 말한다. `-p`는 호스트 포트를 컨테이너로 연결하는 설정이라, 호스트 포트는 하나의 컨테이너만 가질 수 있다.
- r3처럼 호스트 포트만 6380으로 바꾸면 내부 포트가 똑같이 6379여도 문제없다.
- 실패한 r2도 **컨테이너 ID를 먼저 출력**했다. 3장에서 정리한 대로 `docker run`은 `create` → `start` 두 단계로 나뉘는데, `create`는 성공하고 `start`에서 네트워크를 연결하다 실패한 것이다. 그래서 r2는 `Created` 상태로 남아 있고 `docker ps -a`로 확인할 수 있다.

```text
$ docker ps -a --filter name=r2
CONTAINER ID   IMAGE     COMMAND                  CREATED         STATUS    PORTS     NAMES
6461b3efa3cc   redis     "docker-entrypoint.s…"   3 minutes ago   Created             r2

$ ip link show master docker0
6: vethec7c5cc@if2: <...> master docker0 state UP ... link-netnsid 0
9: veth4f27cea@if2: <...> master docker0 state UP ... link-netnsid 1
12: veth346fa8e@if2: <...> master docker0 state UP ... link-netnsid 2
```

- `docker0`에 꽂힌 veth가 1개에서 **3개**로 늘었다. 실행 중인 redis1, r1, r3 하나씩이다. 시작에 실패한 r2는 네트워크 연결 단계까지 가지 못해서 veth가 없다.
- 세 veth 모두 짝이 `@if2`다. 각 컨테이너의 NET namespace 안에서는 `lo`가 1번, `eth0`이 2번이라 번호가 같을 수밖에 없다. 번호는 namespace마다 따로 매겨진다.
- 그래서 짝이 어느 namespace에 있는지는 `link-netnsid`(0, 1, 2)로 구분한다. 컨테이너마다 NET namespace가 하나씩 따로 있다는 걸 여기서도 볼 수 있다.

```text
redis1 eth0 ── vethec7c5cc ─┐
r1     eth0 ── veth4f27cea ─┼── docker0 브리지 ── (iptables DNAT) ── 호스트 6379, 6380
r3     eth0 ── veth346fa8e ─┘
```

### 7-5. Namespace 직접 만들어 보기

```bash
sudo unshare --pid --fork --mount-proc bash
ps aux
```

확인할 것: Docker 없이도 bash가 PID 1이 되고 프로세스가 몇 개만 보이는지.

```text
root@DESKTOP-ABG527P:/home/bong# ps aux
USER         PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root           1  0.0  0.0   5212  4572 pts/3    S    10:43   0:00 bash
root           7  0.0  0.0   7160  4292 pts/3    R+   10:43   0:00 ps aux
```

`ps aux`는 시스템의 모든 프로세스(`a`: 모든 사용자, `x`: 터미널 없는 데몬 포함)를 사용자 이름과 함께(`u`) 보여주는 명령이다. 7-1에서 같은 머신에서 `ps -ef`를 쳤을 때는 `/sbin/init`, `containerd`, `dockerd`, `redis-server` 등 수십 개가 나왔다.

- 그런데 `unshare`로 만든 셸 안에서는 **딱 두 개**만 보인다. 방금 띄운 `bash`와 지금 실행한 `ps aux`뿐이다.
- `bash`가 **PID 1**이다. 원래 PID 1은 `/sbin/init`이었는데, 새 PID namespace에서는 처음 들어온 bash가 1번이 된다. 컨테이너 안의 첫 프로세스가 PID 1이 되는 것과 정확히 같다.
- `ps`가 PID 2가 아니라 7인 이유는 bash가 시작하면서 `.bashrc`의 명령들을 잠깐 실행하느라 2~6번을 이미 썼기 때문이다.
- `--mount-proc`은 `/proc`을 새로 마운트하는 옵션이다. `ps`는 `/proc`을 읽어서 프로세스 목록을 만드는데, 이 옵션이 없으면 호스트의 `/proc`을 그대로 읽어서 다른 프로세스가 다 보인다.

Docker, 이미지, runc 없이 **명령 한 줄로 컨테이너의 PID 격리를 재현**한 셈이다. Docker가 하는 일도 결국 이런 namespace를 PID, NET, MNT, UTS, IPC 전부에 대해 한꺼번에 만들어 주는 것이다.

### 7-6. Cgroup 제한과 OOM kill

```bash
docker run -d --name mem -m 256m redis
CID=$(docker inspect -f '{{.Id}}' mem)
cat /sys/fs/cgroup/system.slice/docker-$CID.scope/memory.max

docker run --name oom -m 50m python:3-slim \
  python -c "a = ' ' * (200 * 1024 * 1024)"
docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' oom
```

확인할 것: `memory.max`가 `268435456`(256MB)인지, OOM 컨테이너의 종료 코드가 `137`, `OOMKilled`가 `true`인지.

```text
$ docker run -d --name mem -m 256m redis
85c19f4525348febfd7b28599b5b53305bcfd10ae955d5bfc8314060279cf35b

$ cat /sys/fs/cgroup/system.slice/docker-$CID.scope/memory.max
268435456

$ docker run --name oom -m 50m python:3-slim python -c "a = ' ' * (200 * 1024 * 1024)"
$ docker inspect -f '{{.State.ExitCode}} {{.State.OOMKilled}}' oom
137 true
```

- `memory.max`의 `268435456`은 256 × 1024 × 1024, 정확히 **256MB**다. `-m 256m` 옵션은 Docker만의 마법이 아니라 커널의 cgroup 파일에 숫자 하나를 써넣는 일이었다. 파일 경로의 `docker-<컨테이너ID>.scope`에서 컨테이너마다 cgroup이 따로 있다는 것도 보인다.
- 두 번째 컨테이너는 한도가 50MB인데 200MB짜리 문자열을 만들려고 했다. 한도를 넘는 순간 커널의 OOM killer가 프로세스를 SIGKILL로 강제 종료했다. 에러 메시지조차 없이 조용히 끝난 게 그 증거다. 예외를 던질 틈도 없이 죽은 것이다.
- 종료 코드 **137** = 128 + 9(SIGKILL). 리눅스에서 시그널로 죽은 프로세스의 종료 코드는 `128 + 시그널 번호`다. `OOMKilled: true`로 원인이 메모리 초과였다는 것도 Docker가 기록해 준다.
- 운영 중인 컨테이너가 이유 없이 137로 죽어 있다면 가장 먼저 메모리 한도를 의심하면 된다. 이게 6장의 JVM 힙 설정이 중요한 이유다.

### 7-7. JVM 힙과 컨테이너 메모리

```bash
docker run --rm -m 1g eclipse-temurin:21 \
  java -XX:+PrintFlagsFinal -version | grep MaxHeapSize
docker run --rm -m 1g eclipse-temurin:21 \
  java -XX:MaxRAMPercentage=75 -XX:+PrintFlagsFinal -version | grep MaxHeapSize
```

확인할 것: 기본값은 약 256MB(1GB의 25%), `MaxRAMPercentage=75`는 약 768MB인지.

```text
# 기본값
   size_t MaxHeapSize      = 268435456     {product} {ergonomic}

# -XX:MaxRAMPercentage=75
   size_t MaxHeapSize      = 805306368     {product} {ergonomic}
```

| 설정 | MaxHeapSize | 컨테이너 한도(1GB) 대비 |
|---|---|---|
| 기본값 | 268,435,456 = **256MB** | 25% |
| `MaxRAMPercentage=75` | 805,306,368 = **768MB** | 75% |

- JVM이 호스트(WSL VM) 전체 메모리가 아니라 **컨테이너 cgroup 한도 1GB를 기준**으로 힙을 계산했다. 7-6에서 본 `memory.max` 파일을 JVM이 직접 읽은 것이다. JDK 8u191 이전 JVM이었다면 호스트 메모리 기준으로 힙을 잡았을 것이다.
- `{ergonomic}`은 내가 `-Xmx`로 직접 정한 값이 아니라 JVM이 환경을 보고 자동으로 계산한 값이라는 표시다.
- 기본값 그대로면 1GB를 받고도 힙은 256MB밖에 못 쓴다. 나머지 768MB는 거의 놀게 된다. 그래서 `MaxRAMPercentage`를 명시한다.
- 그렇다고 100%로 주면 안 된다. 메타스페이스, 스레드 스택, 다이렉트 버퍼, JIT 코드 캐시도 같은 1GB 한도를 나눠 쓴다. 힙이 한도를 다 차지하면 이것들이 들어갈 자리가 없어 7-6처럼 **137로 OOM kill**된다. 75% 정도로 두고 나머지 25%를 힙 외 영역에 남기는 이유다.

### 7-8. 이미지 레이어와 쓰기 레이어

```bash
docker history redis
docker image inspect -f '{{json .RootFS.Layers}}' redis

docker run -d --name ca alpine sleep 3600
docker run -d --name cb alpine sleep 3600
docker exec ca sh -c 'echo hello > /tmp/hello.txt'
docker exec cb ls /tmp
docker diff ca

mount | grep overlay
docker inspect -f '{{json .GraphDriver.Data}}' ca
docker system df -v
```

확인할 것: ca에서 만든 파일이 cb에서 안 보이는지, `docker diff`에 `A /tmp/hello.txt`가 나오는지, overlay 마운트의 `lowerdir`/`upperdir`, `SHARED SIZE`로 레이어 공유.

> 처음에 컨테이너 이름을 `a`, `b`로 줬다가 `Invalid container name (a), only [a-zA-Z0-9][a-zA-Z0-9_.-] are allowed` 에러가 났다. 정규식을 보면 첫 글자 뒤에 한 글자 이상이 더 와야 해서, 컨테이너 이름은 **최소 두 글자**여야 한다.
{: .prompt-warning }

**이미지 레이어**

```text
$ docker history redis
IMAGE          CREATED       CREATED BY                                      SIZE
6f81e8915c60   2 weeks ago   CMD ["redis-server"]                            0B
<missing>      2 weeks ago   EXPOSE map[6379/tcp:{}]                         0B
<missing>      2 weeks ago   ENTRYPOINT ["docker-entrypoint.sh"]             0B
<missing>      2 weeks ago   COPY docker-entrypoint.sh /usr/local/bin/ # …   24.6kB
<missing>      2 weeks ago   WORKDIR /data                                   4.1kB
<missing>      2 weeks ago   RUN |2 REDIS_DOWNLOAD_URL=https://github.com…   8.19kB
<missing>      2 weeks ago   RUN |2 REDIS_DOWNLOAD_URL=https://github.com…   67.3MB
<missing>      2 weeks ago   ARG REDIS_DOWNLOAD_SHA=ae6973de9b6ad8e6cf21b…   0B
<missing>      2 weeks ago   ARG REDIS_DOWNLOAD_URL=https://github.com/re…   0B
<missing>      2 weeks ago   ENV REDIS_VERSION=8.10.2                        0B
<missing>      2 weeks ago   RUN /bin/sh -c set -eux;  apt-get update;  a…   41kB
<missing>      2 weeks ago   RUN /bin/sh -c set -eux;  groupadd -r -g 999…   41kB
<missing>      2 weeks ago   # debian.sh --arch 'amd64' out/ 'trixie' '@1…   87.6MB

$ docker image inspect -f '{{json .RootFS.Layers}}' redis
["sha256:a6dc7651...","sha256:c380752d...","sha256:08040909...","sha256:3013a9ae...",
 "sha256:1f7ec197...","sha256:5f70bf18...","sha256:e43a8d81..."]
```

- `docker history`는 Dockerfile 명령 13줄을 보여주지만 실제 레이어(`RootFS.Layers`)는 **7개**다.
- 크기가 0B인 `CMD`, `EXPOSE`, `ENTRYPOINT`, `ENV`, `ARG`는 파일시스템을 바꾸지 않는다. 이미지 설정 JSON에만 기록되고 레이어를 만들지 않는다. 크기가 있는 7줄(debian 베이스, RUN 4개, WORKDIR, COPY)이 정확히 7개 레이어다.
- 맨 아래 87.6MB 레이어가 debian(trixie) 베이스, 67.3MB 레이어가 redis를 내려받아 빌드한 `RUN`이다.
- 맨 위 한 줄 빼고 IMAGE가 `<missing>`인 건 오류가 아니다. 레지스트리에서 받은 이미지는 중간 단계의 이미지 ID를 갖고 있지 않아서 그렇다.

**overlay 마운트 — 레이어 공유를 눈으로 보기**

```text
$ mount | grep overlay
overlay on /var/lib/docker/rootfs/overlayfs/c78ecaef2c56... type overlay (rw,relatime,
  lowerdir=.../snapshots/11/fs:.../snapshots/10/fs:.../snapshots/9/fs:...:.../snapshots/4/fs,
  upperdir=.../snapshots/12/fs, workdir=.../snapshots/12/work)
overlay on /var/lib/docker/rootfs/overlayfs/5f417fbc2bff... type overlay (rw,relatime,
  lowerdir=.../snapshots/16/fs:.../snapshots/10/fs:.../snapshots/9/fs:...:.../snapshots/4/fs,
  upperdir=.../snapshots/17/fs, ...)
...
```

실행 중인 redis 컨테이너 4개(redis1, r1, r3, mem)마다 overlay 마운트가 하나씩 있다. `lowerdir`, `upperdir`를 정리하면 이렇다.

| 컨테이너 | lowerdir (읽기 전용) | upperdir (쓰기 레이어) |
|---|---|---|
| redis1 (`c78ecaef`) | 11 + **10, 9, 8, 7, 6, 5, 4** | 12 |
| r1 (`5f417fbc`) | 16 + **10, 9, 8, 7, 6, 5, 4** | 17 |
| r3 (`b9e1ad62`) | 20 + **10, 9, 8, 7, 6, 5, 4** | 21 |
| mem (`85c19f45`) | 22 + **10, 9, 8, 7, 6, 5, 4** | 23 |

- 스냅샷 **4~10번 7개**가 네 컨테이너 모두의 `lowerdir`에 똑같이 들어 있다. 위에서 본 redis 이미지 레이어 7개다. 컨테이너를 4개 띄워도 이미지 레이어는 디스크에 **한 벌**만 있고 다 같이 읽는다는 걸 마운트 옵션으로 직접 확인했다.
- `upperdir`(12, 17, 21, 23)는 컨테이너마다 다르다. 각자의 **쓰기 레이어**다.
- `lowerdir` 맨 앞의 11, 16, 20, 22도 컨테이너마다 다르다. Docker가 컨테이너마다 `/etc/hosts`, `/etc/resolv.conf` 같은 파일을 넣기 위해 이미지 위에 한 장 더 얹는 init 레이어로 보인다.
- 경로가 `/var/lib/containerd/io.containerd.snapshotter.v1.overlayfs/...`인 건 최신 Docker가 이미지 저장을 containerd의 snapshotter에 맡기기 때문이다.
- 그 위의 `/usr/lib/modules`, `/mnt/wslg` 같은 overlay 마운트는 Docker가 아니라 WSL 자체가 쓰는 것이다. OverlayFS가 Docker 전용 기술이 아니라 커널 기능이라는 증거이기도 하다.

**디스크 사용량과 레이어 공유**

```text
$ docker system df -v
REPOSITORY        TAG       IMAGE ID       SIZE      SHARED SIZE   UNIQUE SIZE   CONTAINERS
eclipse-temurin   21        3e3c176ffed1   738MB     0B            738.4MB       0
python            3-slim    c3e521df8b2b   192MB     117.5MB       74.07MB       1
redis             latest    6f81e8915c60   213MB     117.5MB       95.2MB        5
alpine            latest    294b683cb724   13MB      0B            13.02MB       0

CONTAINER ID   IMAGE           SIZE      STATUS                       NAMES
48c119e86c9d   python:3-slim   94.2kB    Exited (137) 3 minutes ago   oom
85c19f452534   redis           4.1kB     Up 4 minutes                 mem
b9e1ad62da83   redis           4.1kB     Up 14 minutes                r3
6461b3efa3cc   redis           4.1kB     Created                      r2
5f417fbc2bff   redis           4.1kB     Up 14 minutes                r1
c78ecaef2c56   redis           4.1kB     Up 35 minutes                redis1
```

- python과 redis가 **117.5MB를 공유**한다. 둘 다 같은 debian(trixie) 베이스 레이어 위에 만들어졌기 때문이다. 레이어가 sha256 해시로 식별되니 서로 다른 이미지라도 같은 레이어는 한 번만 저장된다.
- redis 이미지는 213MB인데, 그걸로 만든 컨테이너 5개는 각각 **4.1kB**밖에 안 쓴다. 컨테이너 크기는 쓰기 레이어 크기일 뿐이고 이미지 레이어는 공유하기 때문이다.

**쓰기 레이어 분리**

```text
$ docker run -d --name ca alpine sleep 3600
$ docker run -d --name cb alpine sleep 3600
$ docker exec ca sh -c 'echo hello > /tmp/hello.txt'
$ docker exec cb ls /tmp
                              # 아무것도 출력되지 않음
$ docker diff ca
C /tmp
A /tmp/hello.txt
```

- ca와 cb는 같은 alpine 이미지에서 만들었지만, ca에서 만든 `hello.txt`가 cb에서는 **보이지 않는다**. 공유하는 건 읽기 전용 이미지 레이어뿐이고, 새로 쓴 파일은 ca 자신의 쓰기 레이어(upperdir)에만 들어가기 때문이다.
- `docker diff`는 이미지와 비교해 컨테이너의 쓰기 레이어에 생긴 변화를 보여준다. `A`(Added)는 새로 생긴 파일, `C`(Changed)는 바뀐 디렉터리다. `/tmp`는 이미지에 원래 있던 디렉터리인데, 안에 파일이 생기면서 Copy-on-Write로 쓰기 레이어에 복사되어 `C`로 표시됐다.
- 이 쓰기 레이어는 `docker rm ca`를 하면 같이 지워진다. 컨테이너가 쓴 데이터를 남기려면 볼륨을 써야 하는 이유다.

### 7-9. dockerd는 HTTP 서버다

```bash
systemctl status docker
curl -s --unix-socket /var/run/docker.sock http://localhost/containers/json | head -c 500
```

확인할 것: docker CLI 없이 컨테이너 목록 JSON을 받을 수 있는지.

```text
$ systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: enabled)
     Active: active (running) since Tue 2026-10-06 10:13:42 KST; 41min ago
TriggeredBy: ● docker.socket
   Main PID: 308 (dockerd)
     CGroup: /system.slice/docker.service
             ├─ 308 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock
             ├─1826 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 6379 -container-ip 172.17.0.3 -container-port 6379
             ├─1832 /usr/bin/docker-proxy -proto tcp -host-ip :: -host-port 6379 -container-ip 172.17.0.3 -container-port 6379
             ├─2026 /usr/bin/docker-proxy -proto tcp -host-ip 0.0.0.0 -host-port 6380 -container-ip 172.17.0.4 -container-port 6379
             └─2034 /usr/bin/docker-proxy -proto tcp -host-ip :: -host-port 6380 -container-ip 172.17.0.4 -container-port 6379

... dockerd[308]: level=info msg="image pulled" digest="sha256:c3e521df..."
... dockerd[308]: level=info msg="received task-delete event from containerd" ...
```

- dockerd는 **systemd가 관리하는 데몬**이다. `enabled`라서 부팅할 때 자동으로 뜨고, `Main PID: 308`은 7-1의 `ps -ef`에서 본 dockerd PID와 같다.
- `TriggeredBy: docker.socket`: systemd가 `/var/run/docker.sock` 소켓을 먼저 열어 두고, 그 소켓으로 요청이 오면 dockerd를 띄우는 구조다. dockerd의 실행 옵션 `-H fd://`도 "systemd가 넘겨준 소켓으로 요청을 받겠다"는 뜻이다.
- `CGroup: /system.slice/docker.service`: dockerd 자신도 cgroup 안에 있다. cgroup은 컨테이너 전용 기능이 아니라 systemd가 모든 서비스를 관리하는 데 쓰는 커널 기능이다. 7-6의 컨테이너 cgroup이 `system.slice/docker-<ID>.scope`에 있었던 것도 같은 계층 구조다.
- `docker-proxy` 4개는 7-4에서 `-p`로 연 포트들이다. r1(`172.17.0.3`)에 호스트 6379, r3(`172.17.0.4`)에 호스트 6380이 연결되어 있고, IPv4(`0.0.0.0`)와 IPv6(`::`)용으로 하나씩이다. `-p`는 iptables DNAT 규칙과 함께 이 프록시 프로세스가 호스트 포트를 실제로 붙잡는다. 7-4에서 r2가 `port is already allocated`로 실패한 것도 r1의 docker-proxy가 6379를 이미 잡고 있었기 때문이다.
- 로그의 `received task-delete event from containerd`는 `--rm` 컨테이너가 끝났다는 알림을 containerd가 dockerd에게 보낸 것이다. dockerd가 컨테이너 생명주기를 containerd에 맡긴다는 3장의 내용이 로그에 그대로 남아 있다.

```text
$ curl -s --unix-socket /var/run/docker.sock http://localhost/containers/json | head -c 500
[{"Id":"8bfb11839e57f2670f8082d4a44699994b68cb472d084dec0edd179517f6521f","Names":["/cb"],
"Image":"alpine","ImageID":"sha256:294b683cb724...",
"ImageManifestDescriptor":{"mediaType":"application/vnd.oci.image.manifest.v1+json", ...
"annotations":{..., "org.opencontainers.image.base.name":"scratch", ...
```

- `docker` CLI 없이 **curl만으로** 실행 중인 컨테이너 목록을 받았다. `docker ps`가 하는 일은 결국 `GET /containers/json` HTTP 요청 한 번이다. dockerd는 TCP 포트 대신 유닉스 소켓 파일로 요청을 받는 HTTP 서버였다.
- 목록은 최근에 만든 순서라서 7-8에서 마지막으로 띄운 `cb`가 맨 앞에 나왔다. `Id`도 `docker run`이 출력한 ID(`8bfb1183...`)와 같다.
- `mediaType`이 `application/vnd.oci.image.manifest.v1+json`이다. 3장에서 말한 **OCI 표준 이미지 포맷**이 응답에 그대로 찍혀 있다. Docker 전용 포맷이 아니라서 Podman이나 Kubernetes도 같은 이미지를 쓸 수 있다.
- `org.opencontainers.image.base.name: scratch`는 alpine이 다른 이미지 위가 아니라 **빈 이미지(scratch)에서 시작한** 베이스 이미지라는 뜻이다.
- 이 소켓에 요청을 보낼 수 있으면 컨테이너를 만들고 지우는 것까지 전부 할 수 있다. 그래서 docker.sock 접근 권한이 사실상 호스트 root 권한이라는 것이다.

### 7-10. PID 1과 시그널

```bash
docker run -d --name noinit alpine sleep 1000
time docker stop noinit

docker run -d --init --name withinit alpine sleep 1000
time docker stop withinit
```

확인할 것: `--init` 없이는 sleep이 SIGTERM을 무시해서 약 10초 뒤에야 SIGKILL로 종료되고, `--init`을 주면 바로 종료되는지.

```text
$ docker run -d --name noinit alpine sleep 1000
$ time docker stop noinit
noinit

real    0m10.317s

$ docker run -d --init --name withinit alpine sleep 1000
$ time docker stop withinit
withinit

real    0m0.319s
```

| | PID 1 | `docker stop` 시간 | 종료 방식 |
|---|---|---|---|
| noinit | `sleep` | **10.3초** | SIGTERM 무시 → 10초 뒤 SIGKILL |
| withinit (`--init`) | `docker-init` (tini) | **0.3초** | SIGTERM을 sleep에 전달 → 바로 종료 |

- `--init` 없이는 `sleep`이 PID 1이 된다. `sleep`은 SIGTERM 핸들러를 등록하지 않는데, 커널은 PID 1이 핸들러 없는 시그널을 받으면 **무시**한다. 그래서 SIGTERM은 아무 효과가 없었고, Docker가 기본 유예 시간 10초를 기다린 뒤 SIGKILL로 강제 종료했다. 10.3초라는 숫자가 정확히 그 10초다.
- `--init`을 주면 Docker가 작은 init 프로세스를 PID 1로 넣고, `sleep`은 그 자식이 된다. init이 받은 SIGTERM을 `sleep`에 넘겨주고, PID 1이 아닌 `sleep`은 SIGTERM의 기본 동작대로 바로 종료된다.
- Spring Boot 컨테이너라면 이 차이가 곧 **graceful shutdown이 되느냐 마느냐**다. 10초 뒤 SIGKILL로 죽으면 처리 중이던 요청과 DB 커넥션 정리가 모두 끊긴다. Dockerfile에서 exec form(`ENTRYPOINT ["java", "-jar", "app.jar"]`)을 써서 자바가 직접 PID 1로 SIGTERM을 받게 하거나 `--init`을 쓰는 이유다. 자바는 JVM이 SIGTERM 핸들러(셧다운 훅)를 등록하기 때문에 PID 1이어도 신호를 받는다.

> 처음에는 같은 이름의 컨테이너가 남아 있어서 `docker run`이 `Conflict. The container name "/noinit" is already in use` 에러로 실패했다. 그 상태로 `docker stop`이 기존 컨테이너에 실행되어 시간이 엉뚱하게(5.2초, 0.1초) 나왔다. `docker rm -f noinit withinit`로 지우고 다시 하니 예상대로 나왔다.
{: .prompt-tip }

### 정리

```bash
docker rm -f redis1 r1 r2 r3 mem oom ca cb noinit withinit
```

## 마치며

이론으로만 알던 내용을 실습으로 하나씩 확인하면서 "진짜 그렇구나" 하고 납득하는 과정이 재밌었다. 컨테이너 안의 redis가 호스트의 `ps`에 그대로 찍히고, 50MB 한도를 넘긴 컨테이너가 정말 137로 죽는 걸 눈으로 보니 글로 읽을 때와는 다르게 와닿았다.

JVM과 Docker를 비교하면서 공부한 것도 좋았다. 빌드와 실행 흐름을 `javac` → `.class` → JVM에 대응시키고, Gradle 데몬과 dockerd를 클라이언트-서버 구조로 같이 놓고 보니 처음 보는 개념도 금방 이해됐다. 내가 이미 알고 있는 지식이 전혀 다른 기술을 이해하는 데 그대로 쓰일 수 있다는 걸 느꼈고, 그래서 기초가 되는 CS 지식을 더 열심히 공부해야겠다는 생각이 들었다.

반대로 부족한 부분도 많이 보였다. Docker를 제대로 이해하려면 결국 리눅스 명령어, 운영체제(프로세스, 시그널, 파일시스템), 네트워크(인터페이스, 브리지, 포트) 지식이 필요했는데, 실습 중간중간 막히는 지점이 대부분 여기였다. Docker는 새로운 기술이라기보다 운영체제와 네트워크 기능을 잘 엮어 놓은 도구였고, 그 바탕이 되는 CS 지식을 탄탄하게 쌓아야겠다고 느꼈다.
