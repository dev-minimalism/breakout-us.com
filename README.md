# O'Neil Breakout - US Stock Trading Signals

미국 주식 O'Neil 스타일 돌파매매(Pivot Breakout) 시그널을 감지하고, 시그널마다 Paper Trading 포지션을 자동으로 운용해 성과를 추적하는 대시보드입니다.

**Live Demo**: https://breakout-uscom.vercel.app

> ⚠️ **면책 조항**: 이 프로젝트는 학습 및 연구 목적입니다. 제공되는 시그널과 성과는 투자 권유가 아니며, 모든 거래는 가상(Paper Trading)입니다. 실제 투자 결정과 그 결과는 전적으로 본인 책임입니다.

## 기술 스택

| 구분 | 기술 |
|------|------|
| Frontend | Next.js 16, React 19, TypeScript, Tailwind CSS 4 |
| Backend | FastAPI, Python 3.13, uvicorn |
| Database | PostgreSQL |
| Market Data | yfinance |
| Hosting | Frontend: Vercel / Backend: Self-hosted (Ubuntu + PM2) |

## 주요 기능

- **Today's / Recent Signals**: 오늘 및 최근 N일 돌파 시그널
- **Paper Trading**: 시그널 발생 시 가상 포지션 자동 생성, 손절/익절/기간 만료 자동 청산
- **Performance**: 포트폴리오 수익률, 승률, 월간 성과, 거래 내역
- **Market Status**: 미국장 세션 표시 (프리마켓 / 정규장 / 애프터마켓)
- **Dual Watchlist**
  - 고정: O'Neil 스타일 종목 (`routers/watchlist.py`에 하드코딩)
  - 동적: S&P 500 + NASDAQ 100에서 매일 스크리닝한 상위 100개 종목
- **Backtest Results**: 과거 백테스트 결과 참조

## 아키텍처

```
┌─────────────────┐     ┌─────────────────────────────────┐
│     Vercel      │     │         Server (Ubuntu)         │
│  ┌───────────┐  │     │  ┌─────────────────────────┐    │
│  │  Next.js  │  │────▶│  │  FastAPI (uvicorn, PM2) │    │
│  │  Frontend │  │     │  │  HTTPS :8800            │    │
│  └───────────┘  │     │  │  + BackgroundScanner    │    │
└─────────────────┘     │  └───────────┬─────────────┘    │
                        │  ┌───────────▼─────────────┐    │
                        │  │  PostgreSQL             │    │
                        │  └─────────────────────────┘    │
                        └─────────────────────────────────┘
```

### 시그널 → 포지션 흐름

```
BackgroundScanner (정규장 중 30분 간격)
    ↓
워치리스트(고정 ∪ 동적) 종목을 BreakoutDetector로 분석
    ↓
시그널 감지 → alerts 테이블 저장
    ↓
Paper Trading 포지션 자동 생성 → positions 테이블
    ↓
PositionManager가 청산 조건 체크 → status='closed'
```

스캐너는 FastAPI 프로세스 내부(`BackgroundScanner`, lifespan으로 시작)와 독립 스크립트(`scripts/run_scanner.py`, crontab) 두 방식으로 실행할 수 있으며, 같은 테이블에 기록합니다.

## 트레이딩 규칙

### 돌파 조건 (Pivot Breakout)

| 조건 | 기준 |
|------|------|
| 가격 | 20일 고점 돌파 |
| 거래량 | 평균 대비 +50% 이상 |
| 돌파폭 | 저항선 대비 0~5% |

### 동적 스크리닝 조건

| 조건 | 기준 |
|------|------|
| 시가총액 | >= $500M |
| 주가 | >= $5 |
| 평균 거래량 | >= 50K |

### 포지션 / 청산

| 항목 | 기준 |
|------|------|
| 초기 자본 | $100,000 |
| 포지션 크기 | 자본의 20% |
| 최대 동시 포지션 | 5개 |
| 손절 (Stop Loss) | -8% |
| 익절 (Take Profit) | +20% |
| 최대 보유 | 30일 |

모든 값은 환경변수로 조정할 수 있습니다.

### 수익률 계산

개별 거래 수익률의 합이 아니라 **포트폴리오 수익률**로 계산합니다.

```
실현 손익 = 투자금액 × 수익률(%) / 100
포트폴리오 수익률 = 총 실현 손익 / 초기 자본 × 100
```

예: 5건 × $20,000 × 평균 +10% = $10,000 수익 → 포트폴리오 **+10%**

## 시작하기

### 사전 요구사항

- Python 3.13
- Node.js 20+
- PostgreSQL

### 1. Backend

```bash
cd backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

cp .env.example .env   # DB 접속 정보 등 편집
python scripts/init_db.py
```

### 2. Frontend

```bash
cd frontend
npm install
echo "NEXT_PUBLIC_API_URL=http://localhost:8000" > .env.local
```

### 3. 실행

```bash
./run.sh   # 백엔드 + 프론트엔드 동시 실행
```

| 서비스 | URL |
|--------|-----|
| Frontend | http://localhost:3000 |
| Backend API | http://localhost:8000 |
| API Docs (Swagger) | http://localhost:8000/docs |

## 환경변수

### Backend (`backend/.env`)

전체 목록은 [`backend/.env.example`](backend/.env.example)을 참고하세요.

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `USE_SSH_TUNNEL` | `true` | SSH 터널 경유 DB 연결 (로컬 개발용). 서버에서는 `false` |
| `DB_HOST` / `DB_PORT` / `DB_NAME` / `DB_USER` / `DB_PASSWORD` | - | PostgreSQL 접속 정보 |
| `SSH_HOST` / `SSH_PORT` / `SSH_USER` / `SSH_KEY_PATH` | - | SSH 터널 설정 (`USE_SSH_TUNNEL=true`일 때) |
| `DYNAMIC_WATCHLIST_PATH` | `./watchlist.json` | 동적 워치리스트 JSON 경로 |
| `SCANNER_ENABLED` | `true` | 백그라운드 스캐너 활성화 |
| `SCAN_INTERVAL_SECONDS` | `1800` | 스캔 주기 (초) |
| `SCAN_DELAY_PER_STOCK` | `0.3` | 종목별 지연 (yfinance rate limit 대응) |
| `SCREENING_HOUR` / `SCREENING_MINUTE` | `23` / `30` | 동적 스크리닝 시각 (KST) |
| `MIN_VOLUME_SURGE` | `50.0` | 거래량 급증 기준 (%) |
| `MAX_BREAKOUT_PCT` | `5.0` | 저항선 대비 최대 돌파폭 (%) |
| `INITIAL_CAPITAL` | `100000` | 초기 자본 |
| `POSITION_SIZE_PCT` | `0.20` | 포지션 크기 (**비율**, 0.20 = 20%) |
| `MAX_POSITIONS` | `5` | 최대 동시 포지션 수 |
| `STOP_LOSS_PCT` / `TAKE_PROFIT_PCT` | `0.08` / `0.20` | 손절/익절 (**비율**) |
| `MAX_HOLDING_DAYS` | `30` | 최대 보유일 |
| `ALERT_COOLDOWN_DAYS` | `7` | 같은 종목 재알림 방지 기간 |

### Frontend

| 변수 | 설명 |
|------|------|
| `NEXT_PUBLIC_API_URL` | 백엔드 API URL (미설정 시 `http://localhost:8000`) |

## 스크립트

`backend/`에서 venv 활성화 후 실행합니다.

```bash
python scripts/run_screener.py --max-stocks 100   # 동적 스크리닝 → watchlist.json
python scripts/run_scanner.py --source dynamic    # 시그널 스캔 (--source fixed|dynamic)
python scripts/run_position_manager.py            # 오픈 포지션 청산 조건 체크
```

`docs/reset_paper_trading.sql`은 `positions`와 `alerts`를 모두 비우고 ID 시퀀스를 초기화합니다 (되돌릴 수 없음).

### Crontab 예시 (서버)

FastAPI 내장 `BackgroundScanner`를 쓰지 않거나 보조로 돌릴 때:

```
# 정규장 (KST 23:30-06:00) 30분마다 시그널 스캔
30 23 * * 1-5 cd /var/www/breakout-us.com/backend && venv/bin/python scripts/run_scanner.py >> /tmp/scanner.log 2>&1
0,30 0-5 * * 2-6 cd /var/www/breakout-us.com/backend && venv/bin/python scripts/run_scanner.py >> /tmp/scanner.log 2>&1

# 스캔 5분 뒤 포지션 관리
35 23 * * 1-5 cd /var/www/breakout-us.com/backend && venv/bin/python scripts/run_position_manager.py >> /tmp/position.log 2>&1
5,35 0-5 * * 2-6 cd /var/www/breakout-us.com/backend && venv/bin/python scripts/run_position_manager.py >> /tmp/position.log 2>&1
```

## API 엔드포인트

| Method | Endpoint | 설명 |
|--------|----------|------|
| GET | `/api/watchlist` | 통합 워치리스트 |
| GET | `/api/signals/today` | 오늘 시그널 |
| GET | `/api/signals/recent?days=7` | 최근 N일 시그널 |
| GET | `/api/paper-trading/positions` | 오픈 포지션 (미실현 손익 포함) |
| GET | `/api/paper-trading/closed` | 청산 내역 |
| GET | `/api/paper-trading/stats` | 트레이딩 통계 |
| GET | `/api/paper-trading/monthly` | 월간 성과 |
| GET | `/api/backtest/results` | 백테스트 결과 |
| GET | `/api/backtest/stats` | 백테스트 통계 |
| GET | `/health` | 헬스체크 |

## 시장 세션 (KST 기준)

| 세션 | 시간 (KST) |
|------|------------|
| 프리마켓 | 18:00 - 23:30 |
| 정규장 | 23:30 - 06:00 |
| 애프터마켓 | 06:00 - 10:00 |

모든 스케줄링(스캔, 스크리닝, 로그 롤링)은 KST(UTC+9) 기준입니다. 위 시간은 미국 표준시(EST, 겨울) 기준이며, 서머타임(EDT) 기간에는 1시간씩 앞당겨집니다 (정규장 22:30 - 05:00). 스캐너는 두 경우를 모두 포함하도록 22:00 - 07:00을 장 시간으로 간주합니다.

## 배포

### Frontend (Vercel)

`main` 브랜치 푸시 시 자동 배포됩니다.

- Root Directory: `frontend`
- Environment Variable: `NEXT_PUBLIC_API_URL`

### Backend (GitHub Actions)

[`.github/workflows/deploy-backend.yml`](.github/workflows/deploy-backend.yml)이 `backend/**` 또는 `ecosystem.config.js` 변경 시(또는 수동 실행 시) 서버에 SSH로 접속해 `git reset --hard origin/main` → `pip install` → `pm2 restart breakout-backend` → 헬스체크를 수행합니다. 헬스체크에 실패하면 워크플로우도 실패합니다.

필요한 GitHub Secrets:

| Secret | 설명 |
|--------|------|
| `SSH_PRIVATE_KEY` | 서버 접속용 SSH 개인키 |
| `SERVER_IP` | 서버 IP 주소 |
| `SERVER_USER` | 서버 사용자명 |

서버에서는 PM2(`ecosystem.config.js`)로 uvicorn을 **8800 포트 HTTPS**로 실행합니다. Nginx 참고 설정은 [`nginx.conf`](nginx.conf)에 있습니다.

## 로깅

백엔드/프론트엔드 모두 파일 로깅을 사용하며, KST 자정 기준 1일 롤링, 30일 보관합니다.

| 위치 | 내용 |
|------|------|
| `backend/logs/api.log` | API 서버 로그 |
| `backend/logs/scanner.log` | 스캐너/시그널 로그 |
| `frontend/logs/frontend-YYYY-MM-DD.log` | 프론트엔드 서버사이드 로그 |

## 라이선스

[MIT](LICENSE)
