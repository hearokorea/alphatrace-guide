# AlphaTrace V2 — AI 자동 주식 트레이딩 시스템

> Claude AI + 키움증권 Open API 기반 초실시간 단타 자동매매 시스템

---

## 개요

AlphaTrace V2는 **Claude AI**를 두뇌로 사용하는 한국 주식 자동 트레이딩 시스템입니다.
키움증권 Open API+를 통해 실시간 시세를 수신하고, 규칙 기반 시그널 감지 + AI 최종 판단의 하이브리드 구조로 초단타 매매를 자동 실행합니다.

### 주요 특징

- **AI 기반 매매 판단** — Claude Sonnet 4.6이 시장 데이터를 분석하고 매매를 결정
- **초실시간 스캘핑** — 틱 단위 실시간 데이터 수신 → 규칙 기반 시그널 감지 → AI 최종 판단 → 자동 주문
- **텔레그램 제어** — 스마트폰에서 명령 전송, 현황 확인, 즉시 중지 가능
- **이중 브로커** — COM API 1차, HTS 화면 자동화(OpenClaw) 2차 폴백
- **10중 안전장치** — 손절/익절/트레일링/시간제한/킬스위치 등

---

## 시스템 아키텍처

```
┌─────────────────────────────────────────────────────┐
│                   텔레그램 봇                         │
│        (명령 수신 + 결과 보고 + 스캘핑 제어)          │
└────────┬────────────────────────────────────────────┘
         │ task_queue (스레드 안전)
┌────────▼────────────────────────────────────────────┐
│              메인 스레드 (QTimer 100ms)                │
│     ┌──────────────────┬──────────────────┐          │
│     │  Claude AI Brain  │  ScalpEngine     │          │
│     │  (전략 판단 +     │  (초단타 엔진)    │          │
│     │   자연어 명령)    │                   │          │
│     └────────┬─────────┴────────┬─────────┘          │
│              │                   │                    │
│     ┌────────▼───────────────────▼────────┐          │
│     │         HybridBroker                 │          │
│     │   COM API (1차) → HTS 폴백 (2차)     │          │
│     └─────────────────────────────────────┘          │
└──────────────────────────────────────────────────────┘
```

### 초단타 스캘핑 엔진

```
키움 실시간 데이터 (OnReceiveRealData)
    │
    ▼
TickCollector (틱 수집 → 1/3/5분봉 실시간 빌드)
    │
    ▼
SignalDetector (규칙 기반 시그널, <1ms, API 비용 없음)
    │── 시그널 없음 → skip
    │── 시그널 감지
         │
         ▼
    워커 스레드: Claude AI 최종 판단 (2~3초)
         │
         ▼
    COM API 주문 실행 (broker.buy)
         │
         ▼
    PositionMonitor (손절/익절/트레일링스탑 실시간 감시)
```

**핵심 원칙**: COM API 호출은 메인 스레드에서만, Claude API 호출은 워커 스레드에서.

---

## 기술 스택

| 항목 | 기술 |
|------|------|
| 언어 | Python 3.10 (32-bit) — 키움 COM 필수 |
| AI | Claude Sonnet 4.6 (Anthropic API, tool_use) |
| 증권 API | 키움 Open API+ (COM/QAxWidget) |
| HTS 폴백 | OpenClaw (화면 자동화) |
| GUI 프레임워크 | PyQt5 (COM OCX 호스팅 + 이벤트 루프) |
| 알림 | python-telegram-bot + urllib |
| 스케줄러 | APScheduler (BackgroundScheduler) |
| 시세 데이터 | pykrx (KRX 시장 데이터) |
| 로깅 | loguru |

---

## AI Brain — 도구 10개

| 도구 | 설명 |
|------|------|
| `get_balance` | 계좌 잔고 조회 (예수금, 총평가, 손익) |
| `get_holdings` | 보유 종목 목록 조회 |
| `get_price` | 특정 종목 현재가 조회 |
| `buy_stock` | 매수 주문 (시장가/지정가) |
| `sell_stock` | 매도 주문 (시장가/지정가) |
| `get_market_data` | KRX 시장 데이터 (pykrx, N일치) |
| `get_minute_bars` | 분봉 데이터 (COM API OPT10080, 1/3/5분) |
| `get_scalp_status` | 스캘핑 엔진 현황 조회 |
| `set_scalp_watchlist` | 스캘핑 감시 종목 변경 |
| `report` | 텔레그램 보고 메시지 전송 |

---

## 텔레그램 명령어

| 명령어 | 설명 |
|--------|------|
| `/status` | 계좌 현황 보고 (잔고, 보유종목, 손익) |
| `/analyze` | 즉시 시장 분석 + AI 매매 판단 |
| `/scalp_start` | 스캘핑 엔진 시작 (실시간 감시 모드) |
| `/scalp_stop` | 스캘핑 엔진 즉시 중지 (킬스위치) |
| `/scalp_status` | 스캘핑 현황 조회 (틱/시그널/포지션) |
| 자유 입력 | AI에게 자연어로 질문/명령 |

---

## 스캘핑 시그널 (4가지)

1. **거래량 돌파** — 현재봉 거래량 > 20봉 평균 x 2.5배 + 가격 상승
2. **모멘텀 반전** — 3봉 이상 하락 후 양봉 반전 + 거래량 증가
3. **이동평균 크로스** — 1분봉 5MA > 20MA 골든크로스 + 5분봉 추세 확인
4. **가격 돌파** — 최근 10봉 고점 돌파 + 거래량 1.5배 이상

각 시그널은 **strength 0.0~1.0** 스코어를 가지며, 0.6 이상만 AI 판단으로 전달됩니다.

---

## 포지션 관리

| 항목 | 설정 |
|------|------|
| 손절 | -1.5% (진입가 기준) |
| 익절 | +2.5% |
| 트레일링스탑 | 최고가 대비 1.0% 하락 시 청산 |
| 시간 제한 | 30분 초과 시 자동 청산 |
| 최대 동시 보유 | 5종목 |
| 건당 최대 투자 | 1,000만원 |
| 시그널 쿨다운 | 60초 (중복 방지) |

---

## 자동 스케줄러

| 시각 | 작업 |
|------|------|
| 09:05 | 장 시작 분석 |
| 10:00, 11:00, 13:00, 14:00 | 정기 분석 |
| 15:15 | 마감 보고 |

---

## 안전장치 (10중)

1. 최대 5종목 동시 보유 제한
2. 건당 1,000만원 한도
3. 일일 최대 20회 거래
4. 모든 포지션에 손절가 자동 설정 (-1.5%)
5. 30분 초과 보유 시 자동 청산
6. 시그널마다 AI가 최종 검증 (비합리적 매수 차단)
7. `/scalp_stop` 즉시 중지 (킬스위치)
8. 같은 종목 중복 진입 차단
9. 장 마감(15:30) 이후 주문 금지
10. 한국 주식(KRX)만 거래 가능 — 해외 주식 차단

---

## 설치 가이드

### 사전 요구사항

- Windows 10/11
- 키움증권 계좌 (모의투자 가능)
- Anthropic API 키 (Claude)
- 텔레그램 봇 토큰

### STEP 1: 키움증권 설정

1. **영웅문4 설치**: [키움증권](https://www.kiwoom.com) → 트레이딩 → 영웅문4
2. **Open API+ 모듈 설치**: 같은 페이지에서 다운로드 → 설치 후 재부팅
3. **Open API+ 사용 신청**: 키움 홈페이지 → 온라인업무 → 사용자 API → 신청
4. **모의투자 신청**: 키움 홈페이지 → 모의투자 → 신청

### STEP 2: Python 3.10 (32-bit) 설치

> **중요: 키움 COM API는 32비트 전용!**

1. [Python 3.10 다운로드](https://www.python.org/downloads/release/python-31011/)
2. **Windows installer (32-bit)** 선택
3. 설치 시 `Add Python to PATH` 체크

확인:
```powershell
python -c "import struct; print(f'{struct.calcsize(\"P\")*8}bit')"
# → 32bit
```

### STEP 3: 패키지 설치

```powershell
pip install -r requirements.txt
```

필요 패키지:
```
anthropic, python-telegram-bot, APScheduler, loguru,
pykrx, python-dotenv, PyQt5, pandas, matplotlib
```

### STEP 4: 환경변수 설정 (.env)

```env
# Claude AI
ANTHROPIC_API_KEY=sk-ant-xxxxx

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_CHAT_ID=your_chat_id

# 키움증권
KIWOOM_ACCOUNT_NO=your_account_number
KIWOOM_ACCOUNT_PW=your_password
KIWOOM_IS_VIRTUAL=true

# OpenClaw (HTS 폴백)
OPENCLAW_ENABLED=true

# 스캘핑 엔진
SCALP_WATCHLIST=005930,000660,035420,035720,068270
SCALP_MAX_POSITIONS=5
SCALP_MAX_AMOUNT=10000000
SCALP_STOP_LOSS=-1.5
SCALP_TAKE_PROFIT=2.5
SCALP_TRAILING_STOP=1.0
SCALP_MAX_HOLD_SEC=1800
SCALP_COOLDOWN_SEC=60
```

### STEP 5: 실행

```powershell
python main.py
```

**실행 순서:**
1. 프로그램 시작 → 키움 COM 연결 → 자동 로그인 창
2. 키움 로그인 완료
3. 계좌 비밀번호 등록 창 → 계좌 선택 → 비밀번호 입력 → 등록 → 닫기
4. 텔레그램 봇 시작
5. 텔레그램에서 명령 가능

---

## 프로젝트 구조

```
alphatrace-v2/
├── main.py                     # 메인 엔트리포인트 (QTimer 이벤트 루프)
├── config.py                   # 환경변수 설정
├── .env                        # API 키, 계좌 정보, 스캘핑 설정
├── requirements.txt            # Python 패키지 목록
│
├── broker/                     # 브로커 레이어
│   ├── base.py                 # BaseBroker ABC
│   ├── com_broker.py           # 키움 COM API (QAxWidget) + 실시간 데이터
│   ├── hts_broker.py           # HTS 화면 자동화 (OpenClaw)
│   └── hybrid.py               # COM 우선 → HTS 폴백
│
├── brain/                      # AI 두뇌
│   └── ai_brain.py             # Claude AI Brain (도구 10개)
│
├── strategy/                   # 초단타 전략 모듈
│   ├── tick_collector.py       # 틱 수집 + 1/3/5분봉 자동 빌드
│   ├── signal_detector.py      # 4가지 규칙 기반 시그널 감지
│   ├── position_monitor.py     # 포지션 관리 (손절/익절/트레일링/시간제한)
│   └── scalp_engine.py         # 스캘핑 엔진 (전체 조립)
│
├── notifier/                   # 알림
│   └── telegram_bot.py         # 텔레그램 봇 (명령 + 알림)
│
├── data/                       # 데이터 저장
└── logs/                       # 로그 파일
```

---

## 개발 히스토리

| 버전 | 날짜 | 내용 |
|------|------|------|
| v2.0 | 2026-02-27 | 초기 구축 (COM/HTS 브로커, AI Brain, 텔레그램, 스케줄러) |
| v2.1 | 2026-03-03 | 안정화 (COM 수신 버그, SendOrder 버그, asyncio 충돌 등 수정) |
| v2.2 | 2026-03-03 | 초실시간 스캘핑 엔진 (틱수집, 시그널감지, 포지션관리, AI판단) |

### 향후 로드맵 (v2.3~)
- 시그널 백테스트 기능
- 포지션별 PnL 히스토리 저장 (SQLite)
- 일일 거래 리포트 자동 생성
- 다중 타임프레임 복합 시그널
- 호가창 데이터 활용
- 웹 대시보드 (Flask/Streamlit)

---

## 라이선스

이 프로젝트는 개인 트레이딩 자동화를 위한 것입니다.
투자에 대한 최종 판단과 책임은 사용자에게 있습니다.

---

*Built with Claude AI by HEARO Korea*
