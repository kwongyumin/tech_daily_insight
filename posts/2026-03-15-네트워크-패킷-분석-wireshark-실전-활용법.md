# 네트워크 패킷 분석 Wireshark 실전 활용법

## 개요

운영 환경에서 서비스 응답이 느려지거나, API 호출이 간헐적으로 실패하거나, 이상한 네트워크 트래픽이 감지될 때 로그만으로는 원인을 찾기 어려운 경우가 많다. 이럴 때 **Wireshark**는 개발자의 손이 닿지 않는 네트워크 레이어를 직접 들여다볼 수 있는 강력한 도구다.

Wireshark는 단순히 패킷을 캡처하는 툴을 넘어서, TCP 핸드셰이크 이상 감지, TLS 복호화, HTTP/2 스트림 추적, 응답 지연 원인 분석 등 실무에서 마주치는 거의 모든 네트워크 문제를 진단할 수 있다. 이 글에서는 중급~시니어 백엔드 개발자가 실무에서 즉시 활용할 수 있는 Wireshark 실전 기법을 다룬다.

---

## 핵심 개념

### 패킷 캡처의 원리

Wireshark는 NIC(Network Interface Card)를 **promiscuous mode**로 설정하여 자신에게 향하지 않은 패킷도 수신한다. 내부적으로는 **libpcap**(Linux/macOS) 또는 **WinPcap/Npcap**(Windows)을 사용하여 커널 수준에서 패킷을 복사해온다.

```
[애플리케이션] ↕ Socket API
[OS Kernel]    ↕ TCP/IP Stack
[libpcap]      ← 패킷 미러링
[Wireshark]    ← 분석 및 디코딩
```

### 핵심 필터 문법

Wireshark의 필터는 크게 두 가지로 나뉜다.

- **Capture Filter** (캡처 시 적용): `tcpdump` 문법과 동일한 BPF(Berkeley Packet Filter) 사용
- **Display Filter** (캡처 후 분석): Wireshark 고유 문법, 더 표현력이 풍부함

```bash
# Capture Filter 예시 (BPF 문법)
host 192.168.1.100 and port 8080
not arp and not icmp
tcp port 443

# Display Filter 예시 (Wireshark 고유 문법)
http.request.method == "POST"
tcp.flags.syn == 1 && tcp.flags.ack == 0
ip.addr == 10.0.0.1 && tcp.port == 3306
```

### 중요 통계 메뉴

`Statistics` 메뉴는 단순 패킷 뷰를 넘어서 거시적인 분석을 가능하게 한다.

- **Statistics > Conversations**: 호스트 간 트래픽 양, 패킷 수 비교
- **Statistics > IO Graphs**: 시간대별 트래픽 시각화
- **Statistics > TCP Stream Graphs > Time-Sequence (tcptrace)**: TCP 재전송, 윈도우 크기 문제 시각화
- **Analyze > Expert Information**: Wireshark가 자동으로 이상 패킷을 분류해서 보여줌

---

## 실전 예제

### 예제 1: TCP 재전송(Retransmission) 분석

애플리케이션 레이어 로그에서는 정상처럼 보이지만 응답 지연이 발생할 때, TCP 재전송이 원인인 경우가 많다.

```
# Display Filter: TCP 재전송 패킷만 필터링
tcp.analysis.retransmission or tcp.analysis.fast_retransmission
```

**분석 절차:**

1. 위 필터 적용 후 재전송 빈도 확인
2. `Statistics > IO Graphs`에서 재전송 패킷을 별도 그래프로 오버레이

```
# IO Graph 설정 예시
Graph 1: tcp.analysis.retransmission  (Color: Red)
Graph 2: all traffic                  (Color: Blue)
```

3. 재전송이 집중된 시간대와 서비스 응답 지연 로그 타임스탬프 비교
4. `Follow > TCP Stream`으로 해당 연결의 전체 흐름 추적

### 예제 2: HTTP API 응답 지연 원인 분석 (Time Delta 활용)

Spring Boot REST API에서 특정 엔드포인트 응답이 느릴 때, Wireshark로 각 단계별 소요 시간을 측정할 수 있다.

```
# Display Filter: 특정 API 엔드포인트만 필터링
http.request.uri contains "/api/v1/orders" or http.response
```

**컬럼 추가로 시간 델타 시각화:**

`Edit > Preferences > Columns`에서 다음 컬럼을 추가한다.

```
Field Name: Time since request
Field Type: Custom
Field: http.time
```

이렇게 하면 HTTP 요청부터 응답까지 걸린 시간이 컬럼으로 표시되어 느린 요청을 즉시 식별할 수 있다.

**tshark를 이용한 배치 분석 (CLI):**

```bash
# pcap 파일에서 HTTP 응답 시간 추출 (1초 이상 걸린 요청만)
tshark -r capture.pcap \
  -Y "http.time > 1" \
  -T fields \
  -e frame.number \
  -e ip.src \
  -e ip.dst \
  -e http.request.uri \
  -e http.time \
  -E header=y \
  -E separator=, > slow_requests.csv
```

### 예제 3: TLS 트래픽 복호화

HTTPS 트래픽을 분석하려면 TLS 복호화가 필요하다. **SSLKEYLOGFILE** 환경변수를 활용하면 개발/테스트 환경에서 TLS 세션 키를 추출할 수 있다.

```bash
# Java 애플리케이션의 TLS 세션 키 덤프 (JVM 옵션)
export SSLKEYLOGFILE=/tmp/ssl_keys.log
java -jar myapp.jar

# Chrome/Firefox도 동일한 환경변수 지원
export SSLKEYLOGFILE=/tmp/ssl_keys.log
google-chrome &
```

Wireshark 설정:
```
Edit > Preferences > Protocols > TLS
> (Pre)-Master-Secret log filename: /tmp/ssl_keys.log
```

이후 `https` 필터를 적용하면 암호화된 TLS 패킷이 HTTP/2 또는 HTTP/1.1로 디코딩되어 보인다.

> ⚠️ **주의**: SSLKEYLOGFILE은 반드시 개발/테스트 환경에서만 사용. 운영 환경 적용 금지.

### 예제 4: 데이터베이스 쿼리 패킷 분석 (MySQL)

애플리케이션과 DB 사이의 실제 쿼리를 패킷 레벨에서 확인할 수 있다.

```
# MySQL 트래픽 필터
tcp.port == 3306 and mysql

# 특정 쿼리 문자열이 포함된 패킷 필터
mysql.query contains "SELECT"
```

**tshark로 DB 쿼리 추출:**

```bash
tshark -r db_traffic.pcap \
  -Y "mysql.query" \
  -T fields \
  -e frame.time \
  -e ip.src \
  -e mysql.query \
  -E header=y
```

쿼리 실행 시간 분석은 MySQL Request와 Response 패킷의 타임스탬프 차이를 계산한다.

```bash
# 쿼리 요청-응답 시간 계산 스크립트
tshark -r db_traffic.pcap \
  -Y "mysql" \
  -T fields \
  -e frame.number \
  -e frame.time_epoch \
  -e mysql.command \
  -e mysql.query \
  | awk 'BEGIN{FS="\t"} 
    /Query/{req_time=$2; query=$4; next} 
    /Response/{printf "%.4f sec | %s\n", $2-req_time, query}'
```

### 예제 5: 자동화 캡처 스크립트 (운영 환경 대응)

운영 환경에서는 GUI를 사용할 수 없으므로 `tcpdump`로 캡처 후 Wireshark로 분석하는 패턴이 일반적이다.

```bash
#!/bin/bash
# 롤링 캡처 스크립트: 문제 발생 시 즉시 실행

CAPTURE_DIR="/tmp/captures"
INTERFACE="eth0"
DURATION=60       # 초
FILESIZE=100      # MB
ROTATE=5          # 파일 수

mkdir -p "$CAPTURE_DIR"

tcpdump -i "$INTERFACE" \
  -w "$CAPTURE_DIR/capture_%Y%m%d_%H%M%S.pcap" \
  -G "$DURATION" \
  -C "$FILESIZE" \
  -W "$ROTATE" \
  -z gzip \
  "port 8080 or port 443 or port 3306" \
  &

echo "캡처 시작 PID: $!"
echo "파일 위치: $CAPTURE_DIR"
```

```bash
# 특정 조건 발생 시 캡처 자동 종료 (HTTP 5xx 에러 감지)
tshark -i eth0 \
  -Y "http.response.code >= 500" \
  -a duration:300 \
  -w /tmp/error_capture.pcap \
  -q 2>&1 | tee /tmp/tshark.log &
```

---

## 주의사항 및 트레이드오프

### 운영 환경 캡처 시 주의사항

**성능 영향**

패킷 캡처는 CPU와 I/O를 소모한다. 트래픽이 많은 환경에서는 캡처 자체가 장애를 유발할 수 있다.

```bash
# 캡처 필터를 최대한 좁혀서 성능 영향 최소화
tcpdump -i eth0 -s 0 -c 10000 \
  "host 10.0.0.100 and port 8080 and tcp[tcpflags] & tcp-syn != 0"
  # -s 0: 전체 패킷 캡처 (기본값은 헤더만)
  # -c 10000: 최대 패킷 수 제한으로 무한 캡처 방지
```

**민감 데이터 처리**

```bash
# 캡처 후 민감 정보 마스킹 (editcap 활용)
editcap --inject-secrets tls,keylog.txt input.pcap output_decrypted.pcap

# 또는 패킷 페이로드 제거 후 공유 (헤더만 유지)
tcpdump -i eth0 -s 64 -w headers_only.pcap
```

### 트레이드오프 정리

| 상황 | Wireshark | tcpdump + 사후분석 |
|------|-----------|-------------------|
| 개발 환경 실시간 디버깅 | ✅ GUI 편의성 높음 | ❌ 불편 |
| 운영 서버 캡처 | ❌ GUI 불가 | ✅ 적합 |
| 대용량 트래픽 분석 | ⚠️ 메모리 이슈 | ✅ 필터 후 분석 |
| TLS 복호화 | ✅ GUI 지원 | ❌ 추가 작업 필요 |
| 자동화/스크립팅 | ✅ tshark 활용 | ✅ 파이프라인 구성 |

### Wireshark 대안 도구

실무에서 Wireshark와 함께 사용하면 효과적인 도구들:

```bash
# ngrep: 패킷 내 문자열 검색 (grep의 네트워크 버전)
ngrep -d eth0 -W byline "HTTP" "port 80"

# ss/netstat: 현재 연결 상태 스냅샷
ss -tnp state established '( dport = :8080 )'

# iperf3: 네트워크 대역폭 측정
iperf3 -s  # 서버
iperf3 -c 10.0.0.1 -t 30 -P 4  # 클라이언트 (4개 병렬 스트림)
```

---

## 정리

Wireshark는 단순한 패킷 덤프 툴이 아니라, 네트워크 문제의 **근본 원인(Root Cause)**을 찾기 위한 필수 진단 도구다. 이 글에서 다룬 핵심 포인트를 정리하면 다음과 같다.

1. **Capture Filter vs Display Filter**를 구분하여 사용하면 분석 효율이 크게 높아진다.
2. **TCP 재전송, 윈도우 크기, RTT** 등 TCP 레이어 지표는 애플리케이션 로그에서 절대 보이지 않는 정보다.
3. **SSLKEYLOGFILE**을 활용한 TLS 복호화는 개발/테스트 환경에서 HTTPS 문제 디버깅을 가능하게 한다.
4. **tshark CLI**를 숙지하면 운영 환경에서도 자동화된 네트워크 진단이 가능하다.
5. 운영 환경에서는 캡처 범위