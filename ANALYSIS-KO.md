# 📘 zapret-discord-youtube 분석 정리 (한국어)

> 이 문서는 `zapret-discord-youtube` 프로젝트를 분석하고, 설치·사용법과
> 파생 사업 아이디어까지 정리한 한국어 문서입니다.

---

## 🔗 관련 GitHub 주소

| 구분 | 주소 |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/zapret-discord-youtube |
| **원본 프로젝트** | https://github.com/Flowseal/zapret-discord-youtube |
| **최신 릴리스 (다운로드)** | https://github.com/Flowseal/zapret-discord-youtube/releases/latest |
| **핵심 엔진 (zapret)** | https://github.com/bol-van/zapret |
| **엔진 파라미터 문서** | https://github.com/bol-van/zapret/blob/master/docs/readme.md#nfqws |
| **윈도우 바이너리 번들** | https://github.com/bol-van/zapret-win-bundle |
| **WinDivert 은닉 가이드** | https://github.com/bol-van/zapret-win-bundle/tree/master/windivert-hide |
| **Windows 7용 드라이버** | https://github.com/bol-van/zapret-win-bundle/tree/master/win7 |
| **텔레그램용 대안 도구** | https://github.com/Flowseal/tg-ws-proxy |
| **이슈 / 토론** | https://github.com/Flowseal/zapret-discord-youtube/issues · [Discussions](https://github.com/Flowseal/zapret-discord-youtube/discussions) |

- **라이선스**: MIT
- **현재 로컬 버전**: `1.10.2` (`.service/version.txt` 기준)

---

## 1. 한 줄 요약

**인터넷 검열(차단)을 우회하는 윈도우용 도구.**

러시아에서 국가 차원으로 막힌 Discord·YouTube에 접속하기 위해 만들어졌습니다.
원본 엔진은 `bol-van/zapret`("zapret"은 러시아어로 *금지/차단*)이며,
이 저장소는 그것을 **초보자도 더블클릭만 하면 쓸 수 있게** 배치 파일로 포장한 배포판입니다.

---

## 2. 동작 원리

### 2-1. 쉬운 비유 — "검문소 아저씨"

인터넷 통신은 택배와 비슷합니다. 중간에 검문소(통신사의 DPI 장비)가 있어서,
상자를 열어 받는 사람 이름을 확인하고 차단합니다.

```
검문소: "받는 사람이... 유튜브? ❌ 반송!"
```

이 프로그램은 두 가지 방법으로 검문소를 속입니다.

**① 이름을 잘라서 보내기**

```
원래:  "유튜브"        → 검문소: 걸렸어! ❌
변형:  "유" + "튜브"   → 검문소: 이게 뭐지... 통과 ✅
```

검문소는 대충 보고 못 알아채지만, 실제 서버는 조각을 다시 붙여서 정상 처리합니다.

**② 가짜 택배를 먼저 보내기**

```
가짜 택배: "받는 사람: 구글"   (검문소까지만 도달하고 소멸)
진짜 택배: "받는 사람: 유튜브" (뒤따라 통과)
```

검문소는 앞의 "구글"만 보고 통과시킵니다.

### 2-2. 기술적 설명

통신사의 **DPI(Deep Packet Inspection)** 장비는 TLS 핸드셰이크의
**ClientHello 안 SNI 필드**에서 도메인명을 읽어 차단 여부를 판단합니다.
zapret은 이 판독 과정을 교란합니다.

| 기법 | 동작 |
|---|---|
| `multisplit` / `multidisorder` | 패킷을 여러 조각으로 분할(및 순서 뒤섞기)해 SNI를 못 읽게 함 |
| `fake` | 진짜 요청 앞에 낮은 TTL의 **가짜 패킷**을 삽입 → DPI까지만 도달, 실제 서버엔 미도달 |
| `seqovl` (시퀀스 오버랩) | TCP 시퀀스 번호를 겹치게 조작 → DPI와 실제 서버가 **서로 다른 내용으로 조립** |
| `fakedsplit` / `hostfakesplit` | fake와 split을 조합한 변형 전략 |

`bin/` 폴더의 `tls_clienthello_www_google_com.bin`, `quic_initial_www_google_com.bin` 등이
바로 이때 사용되는 **미끼용 가짜 패킷 원본**입니다.

### 2-3. VPN과의 차이 ⚠️

| | VPN | zapret |
|---|---|---|
| 방식 | 트래픽 전체를 암호화해 외부 서버로 우회 | 나가는 첫 패킷만 변형 (직통) |
| 속도 | 서버를 경유해 느려짐 | 거의 손실 없음 |
| IP 숨김 | O | **X** |
| 익명성 / 보안 | 있음 | **없음** |
| 비용 | 대체로 유료 | 무료 |

> **중요**: zapret은 VPN이 아닙니다. 개인정보 보호·익명성 목적에는 적합하지 않습니다.

---

## 3. 폴더 구조

```
📁 bin/                     핵심 엔진
   winws.exe                실행 파일 (패킷 조작 담당)
   WinDivert.dll / .sys     커널 드라이버 (패킷 가로채기)
   *.bin                    가짜 패킷 재료 (TLS ClientHello, QUIC Initial 등)

📄 general*.bat  (25개)     우회 전략 모음
   general.bat, (ALT)~(ALT13), (SIMPLE FAKE)*, (FAKE TLS AUTO)*, (EXP)
   → 통신사마다 통하는 전략이 달라 하나씩 시도하는 용도

📄 service.bat   (35KB)     관리 도구 (자동실행 등록·진단·테스트)

📁 lists/                   대상 명단
   list-general.txt         우회 대상: discord.*, cloudflare, 7tv, betterttv 등
   list-google.txt          우회 대상: youtube.com, googlevideo.com, ytimg 등
   list-exclude.txt         제외: steam, twitch, yandex, nvidia 등
   ipset-all.txt            IP / 대역 목록 (백업본에 약 32,000줄)
   ipset-exclude.txt        제외 IP

📁 utils/
   test zapret.ps1          전략 자동 테스트 스크립트 (PowerShell)
   targets.txt              테스트 대상 엔드포인트 목록

📁 .service/                버전 정보, hosts 템플릿, ipset 서비스 설정
```

`general.bat`는 `winws.exe`에 **9개의 규칙을 한 번에** 전달합니다.
"443 포트 QUIC은 이렇게, Discord 음성(19294~19344 포트)은 저렇게,
구글 도메인은 요렇게" 식의 **서비스별 맞춤 처방** 구조입니다.

---

## 4. 설치 및 사용법

### ⚠️ 시작 전 체크

| 항목 | 내용 |
|---|---|
| OS | **윈도우 전용** (macOS / Linux 불가) |
| 권한 | **관리자 권한 필수** (커널 드라이버 로드) |
| 백신 | `WinDivert`를 위험도구로 탐지 → **예외 등록 필요** |
| 게임 | 안티치트가 커널 드라이버를 문제 삼을 수 있음 → 주의 |

### STEP 1. Secure DNS 활성화 (필수)

차단은 DNS 단계에서도 걸리기 때문에 이 단계를 건너뛰면 나머지가 무의미합니다.

- **Chrome**: 설정 → 개인정보 및 보안 → 보안 → "보안 DNS 사용" → 제공업체 `Google (dns.google)`
  (Cloudflare는 러시아에서 차단될 수 있어 비권장)
- **Firefox**: 설정 → 개인정보 → "HTTPS를 통한 DNS" → 최대 보호 → 직접 입력
  `https://dns.google/dns-query`
- **Windows 11**: 설정 → 네트워크 → 어댑터 속성 → DNS 서버 할당 편집 → 암호화됨(HTTPS만)
- **Keenetic 공유기 사용 시**: 라우터 설정에서 "요청 전송(Транзит запросов)" 옵션 활성화

### STEP 2. 다운로드 & 압축 해제

1. [릴리스 페이지](https://github.com/Flowseal/zapret-discord-youtube/releases/latest)에서 최신 zip 다운로드
2. **⭐ 중요**: zip 파일 **우클릭 → 속성 → "차단 해제(Unblock)" 체크 → 적용**
   (7-Zip·PeaZip으로 풀면 생략 가능. 이 단계를 놓치면 배치 파일이 조용히 실패합니다.)
3. **한글·공백·특수문자가 없는 경로**에 압축 해제

```
✅ C:\zapret
❌ C:\Users\민수\바탕 화면\새 폴더\zapret
```

### STEP 3. 전략 찾기 (핵심 단계)

`general`로 시작하는 `.bat` 파일 25개를 하나씩 더블클릭하며 통하는 전략을 찾습니다.

**추천 시도 순서**

```
1. general.bat                      ← 기본
2. general (ALT).bat ~ (ALT13).bat
3. general (SIMPLE FAKE)* 계열
4. general (FAKE TLS AUTO)* 계열
5. general (EXP).bat                ← 실험적, 마지막에
```

**정상 동작 확인**
- 콘솔 창이 뜨고 최소화되면서 작업표시줄에 🔒 자물쇠 아이콘(`winws.exe`)이 생김
- 그 상태로 브라우저에서 YouTube / Discord 접속 테스트

**안 될 때**
1. 트레이 자물쇠 아이콘 우클릭 → 닫기 (또는 작업관리자에서 `winws.exe` 종료)
2. 다음 `general` 파일 실행
3. 반복

> 유튜브는 되는데 Discord는 안 되는 경우가 있습니다. 이때는 유튜브가 되는 전략을
> 기준으로 잡고 아래 문제 해결 항목을 참고하세요.

### STEP 4. 자동 실행 등록

`service.bat`을 **관리자 권한으로 실행**하면 나오는 메뉴:

```
  ZAPRET SERVICE MANAGER v1.10.2
  ----------------------------------------
  :: SERVICE
     1. Install Service         ← 자동실행 등록 ⭐
     2. Remove Services         ← 완전 삭제
     3. Check Status            ← 동작 상태 확인

  :: SETTINGS
     4. Game Filter      [disabled]   ← 게임용 (기본 꺼짐)
     5. IPSet Filter     [none]       ← IP 기반 우회 (기본 꺼짐)
     6. Auto-Update Check[enabled]
     7. Replace active fakes

  :: UPDATES
     8. Update IPSet List       ← IP 목록 최신화
     9. Update Hosts File       ← Discord 음성채팅 수정 ⭐
     10. Check for Updates

  :: TOOLS
     11. Run Diagnostics        ← 원인 진단 ⭐
     12. Run Tests              ← 전략 자동 테스트 ⭐
  ----------------------------------------
     0. Exit
```

`1` 입력 후 성공한 전략을 선택하면 `services.msc`에 등록되어 부팅 시 자동 실행됩니다.

> ⚠️ 서비스 등록 후 `general*.bat`을 다시 실행하면 **충돌**합니다. 둘 중 하나만 사용하세요.

### STEP 5. 대상 사이트 추가

`lists/` 폴더에 아래 파일을 만들어 한 줄에 하나씩 추가합니다.
(`*-user.txt` 파일들은 최초 실행 시 자동 생성됩니다.)

| 파일 | 용도 |
|---|---|
| `list-general-user.txt` | 우회할 도메인 추가 (서브도메인 자동 포함) |
| `list-exclude-user.txt` | 우회에서 제외할 도메인 |
| `ipset-all.txt` | IP / 서브넷 추가 |
| `ipset-exclude-user.txt` | 제외할 IP / 서브넷 |

---

## 5. 문제 해결

| 증상 | 해결법 |
|---|---|
| 실행해도 아무 반응 없음 | zip "차단 해제" 여부 + 경로에 한글/공백 없는지 확인 |
| Discord 음성채팅 무한 "연결 중" | `service.bat` → `9` (Update Hosts File) |
| YouTube만 안 됨 | 광고 차단 확장 비활성화 + Secure DNS 재확인 |
| 어떤 전략도 안 됨 | `service.bat` → `11` (Run Diagnostics) |
| 게임이 안 됨 | `8`(IPSet 업데이트) → `4`(Game Filter 켜기) → 전략 재시작 |
| 게임/앱이 오히려 고장남 | `4` Game Filter를 `disabled`, `5` IPSet을 `none`으로 |
| 어떤 전략이 좋은지 모르겠음 | `service.bat` → `12` (Run Tests) |
| 텔레그램이 안 됨 | [tg-ws-proxy](https://github.com/Flowseal/tg-ws-proxy) 사용 |

### 네트워크 초기화 (최후의 수단)

관리자 권한 CMD에서 순서대로 실행 후 재부팅:

```cmd
netsh winsock reset
netsh int ip reset all
netsh winhttp reset proxy
ipconfig /flushdns
```

### 완전 삭제

```
1. service.bat → 2 (Remove Services)
2. service.bat → 11 (Run Diagnostics) → 마지막에 Y
3. 폴더 삭제
```

WinDivert가 서비스에 남아있는 경우, 관리자 CMD에서:

```cmd
driverquery | find "Divert"
sc stop <조회된_서비스명>
sc delete <조회된_서비스명>
```

---

## 6. 한국 사용자에게 실용적인가?

**결론: 실용성은 낮고, 학습 가치는 높습니다.**

### 실용성이 낮은 이유
- 한국에서는 YouTube·Discord가 차단되어 있지 않음
- `lists/` 의 도메인·IP 명단이 전부 러시아 상황 기준
- Windows 전용 + 관리자 권한 필수

### 리스크
- 백신이 `WinDivert`를 `RiskTool.Multi.WinDivert` 등으로 탐지 (실제 악성코드는 아님)
- 게임 안티치트가 커널 드라이버를 문제 삼을 수 있음
- VPN이 아니므로 익명성·보안상의 이득은 전혀 없음

### 그래도 가치 있는 점
- TCP 시퀀스, TLS 핸드셰이크, QUIC, TTL, 커널 드라이버를 한 번에 다루는 **최고급 학습 교재**
- 35,000자 규모의 `service.bat` 배치 스크립트는 그 자체로 참고 가치가 있음
- 해외(러시아·중동 등) 대상 서비스 설계 시 차단 환경에 대한 감각을 얻을 수 있음

---

## 7. 수익화 아이디어

### ⛔ 피해야 할 것

| 항목 | 이유 |
|---|---|
| **이 프로그램 자체를 유료 판매** | MIT라 법적으론 가능하지만, 원본 README가 "가짜 배포 주의"를 경고할 만큼 커뮤니티가 예민함. 평판 손실이 큼 |
| **러시아 타겟 마케팅** | 러시아는 우회수단 홍보 자체가 불법(2024년 법 개정). Stripe·PayPal 등 결제사도 해당 카테고리를 차단 |
| **설치파일에 광고/번들 삽입** | 백신이 이미 민감한 상태 → 실제 멀웨어로 등재될 위험 |

> 이 프로젝트 **자체**의 수익성은 사실상 0입니다. 타겟 유저층의 지불 능력이 제한적이고,
> 무료 대체재가 다수 존재합니다.

### ✅ 기술을 옆으로 옮기는 방향

| 순위 | 아이디어 | 난이도 | 수익성 | 리스크 |
|---|---|---|---|---|
| 🥇 | **네트워크 진단 SaaS (B2B)** | 중 | 높음 | 낮음 |
| 🥈 | **기술 콘텐츠 / 교육** | 하 | 중 | 없음 |
| 🥉 | **글로벌 접속성 모니터링 (지오블로킹 QA)** | 중 | 중~높음 | 낮음 |
| 4 | 게임 네트워크 최적화 툴 | 상 | 높음 | 중 (안티치트) |

**1. 네트워크 진단 SaaS** — 이 저장소에서 가장 상품성 있는 부분은 사실
`service.bat`의 `Run Diagnostics` / `Run Tests`와 `utils/test zapret.ps1` 구조입니다.
"우리 회사 사무실에서만 이 SaaS가 느린 이유"를 진단해주는 도구는 기업이 실제로 비용을 지불하는 영역입니다.

**2. 기술 콘텐츠 / 교육** — "인터넷 차단은 어떻게 뚫리는가" 주제는 수요가 큽니다.
리스크 없이 즉시 시작 가능하며, 애드센스 → 뉴스레터 → 유료 강의로 확장할 수 있습니다.

**3. 지오블로킹 QA** — "우리 서비스가 각국에서 정상 접속되는가" 모니터링.
합법적이고 B2B이며 결제 문제도 없습니다.

**4. 게임 네트워크 최적화** — WinDivert 기반 패킷 제어라는 스택이 이 프로젝트와 동일하며
한국에 실존하는 시장이지만, 안티치트 호환성이 큰 장벽입니다.

---

## 8. React / PHP로 만들 수 있는가?

### 결론: 절반은 가능, 절반은 불가능

```
🔴 패킷 조작 엔진 (winws.exe 역할)
   → C / C++ / Rust + 커널 드라이버 필요
   → React·PHP 로는 불가능

🟢 진단 · 대시보드 · SaaS 레이어
   → React + PHP 로 충분히 구현 가능
```

React는 브라우저 샌드박스 안에서, PHP는 서버 위에서 동작합니다.
패킷 조작은 OS 커널 수준의 작업이라 계층 자체가 다릅니다.

**다만 1순위 아이템인 네트워크 진단 SaaS는 패킷 조작이 필요 없습니다.**

### 제안 아이템: "설치 없이 브라우저로 하는 네트워크 진단"

```
경쟁사: "에이전트를 설치하세요" → 기업 보안팀이 차단
우리:   "이 링크를 열어보세요"   → 즉시 진단 완료
```

기업 고객은 설치형 프로그램에 거부감이 크기 때문에, **무설치**가 최대 차별점이 됩니다.

### 아키텍처

```
┌──────────────┐   진단 실행     ┌────────────────┐
│   React SPA  │ ─────────────> │  대상 엔드포인트 │
│  (브라우저)   │ <───────────── │   (수십 개)     │
└──────┬───────┘  성공/실패/지연  └────────────────┘
       │
       │ 결과 전송 (JSON)
       ▼
┌──────────────┐        ┌──────────────┐
│  PHP/Laravel │ ─────> │  MySQL       │
│   API 서버    │        │  (이력 저장)  │
└──────┬───────┘        └──────────────┘
       │
       │ 서버측 교차검증 (핵심 차별점)
       ▼
  curl / fsockopen / TLS 핸드셰이크 검사
```

**핵심 트릭 — 브라우저 결과와 서버 결과의 대조**

```
서버는 되는데 브라우저는 안 됨  →  "고객사 사내망이 차단 중"
둘 다 안 됨                    →  "대상 서비스 자체의 장애"
```

이 diff가 실제 상품 가치입니다.

### 코드 스케치

**React (브라우저 진단)**

```jsx
const TARGETS = [
  { name: 'Google', url: 'https://www.google.com/favicon.ico' },
  { name: 'Slack',  url: 'https://slack.com/favicon.ico' },
  { name: 'GitHub', url: 'https://github.com/favicon.ico' },
];

async function probe({ name, url }) {
  const t0 = performance.now();
  try {
    // no-cors: 응답 본문은 못 읽어도 "도달 여부"는 알 수 있음
    await fetch(`${url}?t=${Date.now()}`, {
      mode: 'no-cors',
      cache: 'no-store',
      signal: AbortSignal.timeout(5000),
    });
    return { name, ok: true, ms: Math.round(performance.now() - t0) };
  } catch {
    return { name, ok: false, ms: null };
  }
}

const results = await Promise.all(TARGETS.map(probe));
```

**PHP / Laravel (서버 교차검증)**

```php
public function crossCheck(Request $req)
{
    $results = [];
    foreach ($req->input('targets') as $url) {
        $start = microtime(true);
        $ch = curl_init($url);
        curl_setopt_array($ch, [
            CURLOPT_NOBODY         => true,
            CURLOPT_TIMEOUT        => 5,
            CURLOPT_RETURNTRANSFER => true,
        ]);
        curl_exec($ch);
        $results[$url] = [
            'code' => curl_getinfo($ch, CURLINFO_HTTP_CODE),
            'ms'   => round((microtime(true) - $start) * 1000),
            'tls'  => curl_getinfo($ch, CURLINFO_APPCONNECT_TIME),
        ];
        curl_close($ch);
    }
    return response()->json($results);
}
```

PHP는 TCP 포트 점검과 TLS 인증서 검사도 가능합니다.

```php
$sock = @fsockopen('example.com', 443, $errno, $errstr, 3);   // 포트 개방 여부
$ctx  = stream_context_create(['ssl' => ['capture_peer_cert' => true]]);
// → 중간자(MITM) 인증서 감지까지 가능
```

### 추천 스택

| 레이어 | 기술 | 이유 |
|---|---|---|
| 프론트 | React + Vite + TypeScript | 익숙한 스택 |
| 스타일 | Tailwind CSS | 대시보드 구현 속도 |
| 차트 | Recharts | 지연시간 그래프 |
| 백엔드 | Laravel 11 (PHP 8.3) | Queue·Scheduler·Auth 내장 |
| DB | MySQL / PostgreSQL | 이력 저장 |
| 주기 실행 | Laravel Scheduler | `service.bat` 자동 테스트에 대응 |
| 결제 | Paddle / Lemon Squeezy | 세금 처리 대행 |

### 로드맵

| 단계 | 기간 | 내용 |
|---|---|---|
| **1단계** | 1~2주 | React만으로 브라우저 진단 페이지. 결과를 신호등(🟢🟡🔴)으로 표시. 무료 공개로 트래픽 확보 |
| **2단계** | 2~4주 | PHP 백엔드 합류. 결과 저장·이력 그래프·서버 교차검증·공유 리포트 링크 |
| **3단계** | 1~2개월 | 팀 계정, 주기 모니터링, 슬랙 알림, 리전별 프로브, 월 구독 시작 |

### 기술적 한계 (사전 인지)

| 못 하는 것 | 이유 | 대안 |
|---|---|---|
| ping (ICMP) | 브라우저 권한 없음 | HTTP 왕복시간으로 대체 |
| 응답 본문 읽기 | CORS 차단 | `no-cors`로 도달 여부만 판정 |
| DNS 실패 vs 차단 구분 | 브라우저가 구분 정보를 주지 않음 | 서버 교차검증으로 추론 |
| 패킷 레벨 분석 | 샌드박스 제약 | 필요 시 별도 에이전트 추가 |

> 1~2단계 제품 범위에서는 위 한계가 실질적인 제약이 되지 않습니다.

---

## 9. 최종 정리

- 이 저장소는 **팔 물건이 아니라 배울 교재**입니다.
- 한국 환경에서는 실사용 필요성이 거의 없지만, 네트워크 저수준 지식을 얻기에는 훌륭합니다.
- 수익화는 **여기서 배운 기술을 합법적인 B2B 영역으로 옮기는 방향**이 정답입니다.
- 그 방향(네트워크 진단 SaaS)에서는 **React + PHP가 오히려 최적의 선택**입니다.
  무설치로 브라우저에서 동작한다는 점이 가장 큰 강점이기 때문입니다.
