# AlphaTrace V3.7 — AI 주도형 자율 매매 시스템

> Claude AI + 키움증권 Open API 기반 자율 주식 매매 시스템

---

## 개요

AlphaTrace V3는 **Claude AI**가 직접 시장을 분석하고, 판단하고, 매매를 실행하는 **AI 주도형 자율 트레이딩 시스템**입니다.

V2의 스캘핑 엔진을 기반으로, V3에서는 AI가 **자율적으로 루프를 돌며** 시장을 스캔하고, 조건식을 평가하고, 전략을 실행합니다.

### V2 → V3 핵심 변화

| 항목 | V2 | V3 |
|------|----|----|
| 매매 방식 | 사용자 명령 → AI 판단 | **AI 자율 루프** (자동 스캔/판단/실행) |
| 설정 변경 | `.env` 수정 → 재시작 | **텔레그램 실시간 변경** (재시작 불요) |
| 운영 모드 | 수동 파라미터 | **프리셋 모드** (aggressive/normal/conservative/daily) |
| 루프 트리거 | 고정 스케줄러 | **적응형 간격 + 장 시작/종료 스케줄** |
| 조건식 | 없음 | **자연어 조건식** 등록/평가/자동 실행 |
| 전략 | 4가지 시그널 고정 | **전략 엔진** (사용자 정의 전략 + 전종목 스크리닝) |
| 복구 | 수동 재시작 | **COM 자동 재연결** + 포지션/통계 영속화 |

### 주요 특징

- **AI 자율 매매** — Claude가 자율적으로 시장 스캔 → 분석 → 매수/매도 직접 실행
- **텔레그램 실시간 제어** — 모든 파라미터를 텔레그램에서 즉시 변경 (재시작 불요)
- **운영 모드 프리셋** — "공격적으로 바꿔" 한마디로 AI+스캘핑 10개 파라미터 일괄 변경
- **스케줄 트리거** — 장 시작/종료 시각에 자동으로 AI 루프 실행
- **초실시간 스캘핑** — 틱 단위 시그널 감지 + AI 검증 + 자동 주문
- **자연어 조건식** — "삼성전자 7만원 이하면 500만원 매수" → AI가 파싱/평가/실행
- **전략 스크리너** — pykrx 전종목 기술적 스크리닝
- **COM 자동 복구** — 키움 API 끊김 감지 → 자동 재연결 → 실시간 재구독

---

## 시스템 아키텍처

```
┌─────────────────────────────────────────────────────────┐
│                    텔레그램 봇                            │
│     (명령 수신 + AI 대화 + 설정 변경 + 스캘핑 제어)       │
└─────────┬───────────────────────────────────────────────┘
          │ task_queue (스레드 안전)
┌─────────▼───────────────────────────────────────────────┐
│               메인 스레드 (QTimer 이벤트 루프)              │
│                                                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐   │
│  │  AI 자율 루프  │  │ 스캘핑 엔진   │  │ 헬스 모니터   │   │
│  │  (AICore)     │  │ (ScalpEngine)│  │ (자동 재연결) │   │
│  │  적응형 간격   │  │ 실시간 틱     │  │ COM 감시     │   │
│  │  +스케줄 트리거│  │ +AI 판단      │  │ +TR 타임아웃 │   │
│  └──────┬───────┘  └──────┬───────┘  └──────────────┘   │
│         │                  │                             │
│  ┌──────▼──────────────────▼──────────────────────────┐  │
│  │        RuntimeConfig (V3.7 — 24개 파라미터)          │  │
│  │     JSON 영속 + 텔레그램 즉시 변경 + 모드 프리셋      │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐  │
│  │              HybridBroker                           │  │
│  │        COM API (1차) → HTS 폴백 (2차)               │  │
│  └────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

### AI 자율 루프 (V3 핵심)

```
AICore.tick() — 2초마다 체크
    │
    ├─ 간격 트리거: 마지막 루프 후 N초 경과?
    │   └─ 적응형: 포지션 보유(90초) / 조건식 활성(120초) / 유휴(300초)
    │
    └─ 스케줄 트리거: 장 시작(09:00) / 장 종료(15:20) / 커스텀 시각?
         │
         ▼ (어느 하나 충족)
    run_loop() 실행:
    ① 보유종목 조회 → ② 거래대금 상위 스캔 → ③ 조건식 평가
    ④ 전략 스크리닝 → ⑤ AI 자율 매매 판단 → ⑥ 텔레그램 보고
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

---

## RuntimeConfig — 텔레그램 실시간 설정 (V3.7)

모든 운영 파라미터를 **텔레그램에서 즉시 변경** 가능. 재시작 불필요.

### 파라미터 목록 (24개)

| 카테고리 | 파라미터 | 기본값 | 설명 |
|----------|----------|--------|------|
| **AI 루프** | `ai_loop_interval_position` | 90 | 포지션 보유 시 루프 주기(초) |
| | `ai_loop_interval_active` | 120 | 조건식 활성 시 루프 주기(초) |
| | `ai_loop_interval_idle` | 300 | 유휴 시 루프 주기(초) |
| | `ai_loop_max_iter` | 16 | 루프당 최대 도구 호출 수 |
| | `ai_report_interval` | 600 | 텔레그램 정기 보고 주기(초) |
| | `ai_loop_schedule` | "" | 스케줄 트리거 (아래 참조) |
| **스캘핑** | `scalp_cooldown_sec` | 60 | 동일 종목 시그널 쿨다운(초) |
| | `scalp_max_hold_sec` | 1800 | 포지션 최대 보유 시간(초) |
| | `scalp_stop_loss_pct` | -1.5 | 손절 %(음수) |
| | `scalp_take_profit_pct` | 2.5 | 익절 % |
| | `scalp_trailing_stop_pct` | 1.0 | 트레일링스탑 % |
| | `scalp_max_positions` | 5 | 최대 동시 포지션 수 |
| | `scalp_max_amount` | 10,000,000 | 건당 최대 금액(원) |
| | `scalp_notify_interval` | 600 | 스캘핑 알림 주기(초) |
| | `scalp_tick_stall_sec` | 180 | 틱 미수신 경고 임계(초) |
| | `scalp_resubscribe_sec` | 300 | 틱 미수신 재구독 임계(초) |
| | `scalp_heartbeat_sec` | 60 | 스캘핑 하트비트 주기(초) |
| **시그널** | `signal_strength_threshold` | 0.6 | 시그널 강도 임계값 |
| | `signal_volume_mult` | 2.5 | 거래량 돌파 배수 |
| **캐시** | `cache_ttl_holdings` | 60 | 보유종목 캐시 TTL(초) |
| | `cache_ttl_price` | 120 | 시세 캐시 TTL(초) |
| | `cache_ttl_top_stocks` | 300 | 상위종목 캐시 TTL(초) |
| | `cache_ttl_stock_info` | 1800 | 종목정보 캐시 TTL(초) |
| **장 시간** | `auto_shutdown_time` | "15:35" | 자동 종료 시각(HH:MM) |

> 모든 시간 파라미터는 최소 1초, 상한 없음 (1초 ~ 무제한)

### 스케줄 트리거 (`ai_loop_schedule`)

| 값 | 동작 |
|----|------|
| `""` (빈값) | 간격 트리거만 사용 |
| `"market_open"` | 09:00에 자동 실행 |
| `"market_close"` | 15:20에 자동 실행 |
| `"market_both"` | 09:00 + 15:20 |
| `"09:00,12:00"` | 커스텀 시각 (쉼표 구분) |
| `"market_open,12:00,15:00"` | 혼합 가능 |

### 운영 모드 프리셋

텔레그램에서 한마디로 AI+스캘핑 파라미터를 일괄 변경:

| 모드 | AI 루프 간격 | 스캘핑 HB | 스케줄 | 설명 |
|------|-------------|-----------|--------|------|
| `aggressive` | 60/60/120초 | 10초 | 장 시작 | 빠른 반응, 많은 포지션 |
| `normal` | 90/120/300초 | 60초 | 장 시작 | 기본 설정 |
| `conservative` | 180/300/600초 | 120초 | 장 시작+종료 | 느린 반응, 적은 포지션 |
| `daily` | 86400초 | 300초 | 장 시작+15:20 | 하루 2회만 |

**텔레그램 사용 예시:**
```
"공격적 모드로 바꿔"           → set_mode("aggressive")
"루프 간격 60초로 변경해"       → set_config("ai_loop_interval_idle", 60)
"장 시작할 때 루프 돌려"        → set_config("ai_loop_schedule", "market_open")
"손절 2%로 변경"               → set_config("scalp_stop_loss_pct", -2.0)
"현재 설정 보여줘"             → get_config()
"사용 가능한 모드 보여줘"       → get_modes()
```

---

## 기술 스택

| 항목 | 기술 |
|------|------|
| 언어 | Python 3.10 (32-bit) — 키움 COM 필수 |
| AI (메인) | Claude Sonnet 4.6 (Anthropic API, tool_use) |
| AI (스캘핑) | Claude Haiku 4.5 (빠른 응답, 10배 저렴) |
| 증권 API | 키움 Open API+ (COM/QAxWidget) |
| HTS 폴백 | OpenClaw (화면 자동화) |
| GUI | PyQt5 (COM OCX 호스팅 + 이벤트 루프) |
| 알림 | python-telegram-bot + urllib |
| 시세 데이터 | pykrx (KRX 전종목 시장 데이터) |
| 로깅 | loguru |

---

## AI Brain — 도구 목록

### 매매/조회 도구

| 도구 | 설명 |
|------|------|
| `get_balance` | 계좌 잔고 조회 |
| `get_holdings` | 보유 종목 목록 |
| `get_price` | 종목 현재가 조회 |
| `get_prices_batch` | 여러 종목 일괄 시세 조회 |
| `get_minute_bars` | 분봉 OHLCV 데이터 |
| `get_stock_info` | 종목 상세정보 (시총, PER, PBR) |
| `get_investor_trend` | 투자자별 매매동향 |
| `search_top_stocks` | 거래대금/등락률/거래량 상위 종목 |
| `buy_stock` | 매수 주문 (시장가/지정가) |
| `sell_stock` | 매도 주문 (시장가/지정가) |
| `get_pending_orders` | 미체결 주문 조회 |

### 조건식/전략 도구

| 도구 | 설명 |
|------|------|
| `add_condition` | 자연어 조건식 등록 |
| `list_conditions` | 등록된 조건식 목록 |
| `remove_condition` | 조건식 삭제 |
| `list_strategies` | 전략 목록 조회 |

### 설정/모드 도구

| 도구 | 설명 |
|------|------|
| `get_config` | 런타임 설정 조회 (전체 또는 개별) |
| `set_config` | 런타임 설정 즉시 변경 |
| `set_mode` | 운영 모드 프리셋 일괄 적용 |
| `get_modes` | 사용 가능한 모드 목록 |

### 유틸리티 도구

| 도구 | 설명 |
|------|------|
| `get_scalp_status` | 스캘핑 엔진 현황 |
| `get_trade_history` | 거래 이력 조회 |
| `report` | 텔레그램 보고 메시지 전송 |
| `save_memo` | 전략 메모 저장 |
| `read_source` | 소스코드 읽기 (자가 개선) |
| `modify_source` | 소스코드 수정 (자가 개선) |
| `get_modification_log` | 코드 수정 이력 |

---

## 텔레그램 명령어

| 명령어 | 설명 |
|--------|------|
| `/status` | 계좌 현황 보고 |
| `/analyze` | 즉시 시장 분석 + AI 매매 판단 |
| `/auto_start` | AI 자율 루프 시작 |
| `/auto_stop` | AI 자율 루프 중지 |
| `/scalp_start` | 스캘핑 엔진 시작 |
| `/scalp_stop` | 스캘핑 엔진 즉시 중지 (킬스위치) |
| 자유 입력 | AI에게 자연어로 질문/명령/조건식/설정변경 |

**자유 입력 예시:**
```
"삼성전자 7만원 이하면 500만원 매수"    → 조건식 자동 등록
"공격적 모드로 바꿔"                   → 모드 프리셋 적용
"루프 간격 60초로 줄여"                → 런타임 설정 변경
"현재 포지션 정리해"                   → AI 판단 후 매도 실행
"장 시작할 때만 분석해"                → 스케줄 트리거 설정
```

---

## 안전장치

1. 최대 동시 보유 제한 (기본 5종목, 동적 변경 가능)
2. 건당 최대 투자금 한도 (기본 1,000만원)
3. 일일 최대 거래 횟수 제한
4. 모든 포지션 자동 손절가 설정
5. 최대 보유 시간 초과 시 자동 청산
6. 시그널마다 AI 최종 검증
7. `/scalp_stop` 킬스위치
8. 동일 종목 중복 진입 차단
9. 장 마감 후 주문 차단
10. COM 자동 재연결 (끊김 감지 → 복구 → 재구독)
11. 중복 실행 방지 (Windows Named Mutex)
12. 장 마감 자동 종료 (RuntimeConfig 동적 시각)

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

### STEP 3: 패키지 설치

```powershell
pip install anthropic python-telegram-bot loguru pykrx python-dotenv PyQt5 pandas
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

# 스캘핑 엔진
SCALP_WATCHLIST=005930,000660,035420,035720,068270
```

### STEP 5: 실행

```powershell
python main.py
```

---

## 프로젝트 구조

```
alphatrace-v3/
├── main.py                        # 메인 엔트리포인트 (QTimer 이벤트 루프)
├── config.py                      # 환경변수 설정
├── .env                           # API 키, 계좌 정보
│
├── brain/                         # AI 두뇌
│   ├── ai_brain.py                # Claude AI Brain (도구 25개+)
│   ├── ai_core.py                 # AI 자율 루프 코어 (적응형 간격 + 스케줄)
│   ├── runtime_config.py          # 런타임 설정 (V3.7, 텔레그램 즉시 변경)
│   ├── condition_store.py         # 조건식 엔진 (자연어 파싱/평가/실행)
│   ├── state_manager.py           # 상태 영속화 (거래기록, 토큰, 루프 카운트)
│   └── self_modifier.py           # 자가 수정 엔진 (코드 수정/롤백)
│
├── broker/                        # 브로커 레이어
│   ├── com_broker.py              # 키움 COM API (QAxWidget) + 자동 재연결
│   ├── hts_broker.py              # HTS 화면 자동화 (OpenClaw)
│   └── hybrid.py                  # COM 우선 → HTS 폴백
│
├── strategy/                      # 전략 모듈
│   ├── scalp_engine.py            # 스캘핑 엔진 (틱→시그널→AI→주문)
│   ├── tick_collector.py          # 틱 수집 + 분봉 자동 빌드
│   ├── signal_detector.py         # 4가지 규칙 기반 시그널 감지
│   ├── position_monitor.py        # 포지션 관리 (손절/익절/트레일링)
│   ├── safety_layer.py            # 안전장치 레이어
│   ├── strategy_store.py          # 전략 저장소
│   ├── strategy_screener.py       # 전종목 전략 스크리닝
│   ├── market_data.py             # pykrx 시장 데이터
│   └── indicator_calc.py          # 기술적 지표 계산
│
├── notifier/                      # 알림
│   └── telegram_bot.py            # 텔레그램 봇 (명령 + AI 대화)
│
├── data/                          # 런타임 데이터 (JSON 영속)
│   ├── runtime_config.json        # 런타임 설정 (텔레그램에서 변경)
│   ├── ai_state.json              # AI 상태 (토큰, 루프 카운트, 메모)
│   ├── conditions.json            # 등록된 조건식
│   ├── positions.json             # 스캘핑 포지션 (재시작 시 복원)
│   ├── trades.json                # 거래 이력
│   ├── scalp_stats.json           # 스캘핑 누적 통계
│   └── strategies/                # 사용자 정의 전략
│
└── logs/                          # 일별 로그 파일
```

---

## 개발 히스토리

| 버전 | 날짜 | 내용 |
|------|------|------|
| v2.0 | 2026-02-27 | 초기 구축 (COM/HTS, AI Brain, 텔레그램, 스케줄러) |
| v2.1 | 2026-03-03 | 안정화 (COM 버그, asyncio 충돌 수정) |
| v2.2 | 2026-03-03 | 초실시간 스캘핑 엔진 (틱수집, 시그널, AI판단) |
| v3.0 | 2026-03-04 | AI 자율 루프 코어, 조건식 엔진, 자가 수정 |
| v3.1 | 2026-03-05 | 워커 스레드 제거 (메인 스레드 통합) |
| v3.2 | 2026-03-05 | 포지션/통계 영속화 (재시작 시 복원) |
| v3.3 | 2026-03-06 | 전략 엔진 (사용자 정의 전략 + 스크리닝) |
| v3.4 | 2026-03-07 | COM 자동 재연결, 헬스 모니터 |
| v3.5 | 2026-03-08 | 시세 캐시, 적응형 루프 주기, 데이터 사전주입 |
| v3.6 | 2026-03-09 | pykrx 전종목 스크리닝, 중복 실행 방지 |
| v3.7 | 2026-03-10 | RuntimeConfig (텔레그램 즉시 변경), 운영 모드 프리셋, 스케줄 트리거 |

---

## 라이선스

이 프로젝트는 개인 트레이딩 자동화를 위한 것입니다.
투자에 대한 최종 판단과 책임은 사용자에게 있습니다.

---

*Built with Claude AI by HEARO Korea*
