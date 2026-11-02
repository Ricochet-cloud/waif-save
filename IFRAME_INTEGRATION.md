# JUMP & SURVIVE - iframe 통합 가이드

## 개요

JUMP & SURVIVE는 iframe을 통해 외부 웹사이트에 쉽게 임베드할 수 있는 독립형 게임 모듈입니다.

### 핵심 특징

- ✅ **독립적 실행**: 단독으로도 실행 가능, iframe 내에서도 실행 가능
- ✅ **난이도 선택**: EASY, NORMAL, HARD 세 가지 난이도 지원
- ✅ **postMessage 통신**: 부모 페이지와 안전한 메시지 기반 통신
- ✅ **결과 전송**: 게임 종료 시 결과 데이터를 부모에게 전송
- ✅ **플레이어 정보**: 플레이어 ID와 이름을 게임에 전달 가능

---

## 빠른 시작

### 기본 임베드 (5분)

부모 페이지에서 다음과 같이 iframe을 삽입하세요:

```html
<iframe
  id="gameFrame"
  src="https://your-domain.com/jump-survive.html"
  width="800"
  height="500"
  allow="fullscreen"
  loading="lazy">
</iframe>

<script>
  const gameFrame = document.getElementById('gameFrame');

  // 게임 난이도 설정 및 플레이어 정보 전송
  function startGame(difficulty, playerId, playerName) {
    gameFrame.contentWindow.postMessage({
      type: 'gameConfig',
      difficulty: difficulty,        // 'EASY', 'NORMAL', 'HARD'
      playerId: playerId,           // 고유한 플레이어 ID
      playerName: playerName        // 플레이어 이름
    }, '*');
  }

  // 게임 결과 수신
  window.addEventListener('message', function(event) {
    if (event.data.type === 'gameResult') {
      console.log('게임 결과:', event.data);
      // 여기서 결과를 서버에 전송하거나 UI를 업데이트합니다
    }
  });

  // 게임 시작
  startGame('NORMAL', 'player_001', '테스트 플레이어');
</script>
```

---

## API 상세 문서

### 메시지 포맷

#### 1. 게임 설정 메시지 (부모 → 게임)

게임을 시작할 때 보내는 메시지:

```javascript
{
  type: 'gameConfig',
  difficulty: 'NORMAL',      // 필수: 'EASY' | 'NORMAL' | 'HARD'
  playerId: 'user_123',      // 선택사항: 고유한 플레이어 ID
  playerName: '플레이어명'   // 선택사항: 표시할 플레이어 이름
}
```

**예제:**
```javascript
gameFrame.contentWindow.postMessage({
  type: 'gameConfig',
  difficulty: 'HARD',
  playerId: 'user_001',
  playerName: 'Alice'
}, '*');
```

#### 2. 게임 결과 메시지 (게임 → 부모)

게임이 종료될 때 반환되는 메시지:

```javascript
{
  type: 'gameResult',
  playerId: 'user_123',           // 플레이어 ID (설정했을 경우)
  playerName: '플레이어명',       // 플레이어 이름 (설정했을 경우)
  difficulty: 'NORMAL',          // 게임 난이도
  survived: 45.2,                // 생존한 시간 (초)
  duration: 60,                  // 목표 시간 (초)
  isWin: false,                  // 게임 성공 여부 (true=클리어, false=실패)
  timestamp: 1704067200000      // Unix 타임스탬프
}
```

**예제:**
```javascript
{
  type: 'gameResult',
  playerId: 'user_123',
  playerName: 'Alice',
  difficulty: 'NORMAL',
  survived: 60.0,
  duration: 60,
  isWin: true,
  timestamp: 1704067200000
}
```

---

## 난이도 설정

각 난이도별 특성:

### EASY (초보자 친화적)
- **게임 시간**: 50초
- **특징**: 느린 투사체, 긴 생성 간격, 낮은 중력
- **목표 클리어율**: 약 90% 이상
- **권장 대상**: 게임 초보자, 아동

### NORMAL (표준)
- **게임 시간**: 60초
- **특징**: 균형잡힌 난이도
- **목표 클리어율**: 약 70~85%
- **권장 대상**: 일반 플레이어 (기본값)

### HARD (도전적)
- **게임 시간**: 70초
- **특징**: 빠른 투사체, 짧은 생성 간격, 높은 중력
- **목표 클리어율**: 약 50~70%
- **권장 대상**: 숙련된 플레이어

---

## 보안 고려사항

### Origin 검증 (향후 구현)

현재 게임은 모든 origin에서의 메시지를 수락하지만, 향후 배포 시 아래와 같이 origin 화이트리스트를 추가할 예정입니다:

```javascript
const ALLOWED_ORIGINS = [
  'https://your-main-site.com',
  'https://partner-site.com',
  'http://localhost:3000'
];

function handleMessage(event) {
  if (!ALLOWED_ORIGINS.includes(event.origin)) {
    return; // 허용되지 않은 origin 거부
  }
  // 처리...
}
```

### 클라이언트 결과 검증 경고

⚠️ **중요**: 클라이언트에서 보낸 게임 결과만으로 보상을 지급하면 부정행위가 가능합니다.

**필수 서버측 검증:**
1. 플레이어 인증 확인 (로그인 상태)
2. 게임 결과의 합리성 검증
   - 생존 시간이 게임 지속시간을 초과하지 않는지 확인
   - 플레이어의 평균 성공률과 비교하여 이상치 감지
3. 중복 결과 방지 (중복 클리어로 인한 중복 보상 방지)
4. 속도 제한 (플레이어가 비정상적으로 빠른 속도로 게임을 반복하는지 확인)

**예제 서버측 검증:**
```javascript
// Node.js + Express 예제
app.post('/api/game-result', async (req, res) => {
  const { playerId, difficulty, survived, duration, isWin } = req.body;

  // 1. 인증 확인
  const player = await Player.findById(playerId);
  if (!player) return res.status(401).json({ error: '인증 필요' });

  // 2. 결과 검증
  if (survived > duration * 1.1) {
    return res.status(400).json({ error: '유효하지 않은 결과' });
  }

  // 3. 중복 방지 (마지막 게임이 1초 이내라면 중복으로 간주)
  if (Date.now() - player.lastGameTime < 1000) {
    return res.status(429).json({ error: '너무 빠른 플레이' });
  }

  // 4. 결과 저장 및 보상 지급
  await GameResult.create({ playerId, difficulty, survived, duration, isWin });
  if (isWin) {
    player.points += getRewardPoints(difficulty);
    await player.save();
  }

  res.json({ success: true, points: player.points });
});
```

---

## 실제 통합 예제

### 1. 간단한 예제 (example_simple_embed.html)

난이도 선택만 포함한 기본 예제입니다.

```bash
# 1. 파일 확인
cat example_simple_embed.html

# 2. 로컬 서버 실행
python -m http.server 8000

# 3. 브라우저에서 접속
http://localhost:8000/example_simple_embed.html
```

### 2. 고급 예제 (example_advanced_embed.html)

플레이어 정보, 결과 검증, 결과 표시 등을 포함한 완전한 예제입니다.

```bash
# 로컬 서버 실행
python -m http.server 8000

# 브라우저에서 접속
http://localhost:8000/example_advanced_embed.html
```

---

## 문제 해결

### iframe이 로드되지 않음

1. 브라우저 콘솔(F12)에서 CORS 오류 확인
2. 게임 파일의 경로가 올바른지 확인
3. 브라우저의 보안 설정 확인

### postMessage가 작동하지 않음

```javascript
// 문제 진단
console.log('부모 window:', window.parent);
console.log('iframe window:', gameFrame.contentWindow);
console.log('같은 origin 여부:', window.location.origin === gameFrame.contentWindow.location.origin);
```

### 게임 결과가 수신되지 않음

1. `message` 이벤트 리스너가 올바르게 등록되었는지 확인
2. 콘솔에서 게임 결과 메시지 확인 (DevTools → Console → Network)
3. postMessage의 targetOrigin 설정 확인 ('*' 사용 또는 명시적 origin)

---

## 배포 체크리스트

향후 프로덕션 배포 전 확인사항:

- [ ] Origin 화이트리스트 설정 (ALLOWED_ORIGINS 수정)
- [ ] 서버측 결과 검증 로직 구현
- [ ] 플레이어 인증 시스템 연동
- [ ] 보상 지급 로직 구현
- [ ] 결과 데이터베이스 스키마 생성
- [ ] HTTPS 적용 (postMessage origin 검증에 필수)
- [ ] 클라이언트 부정행위 탐지 로직 추가
- [ ] 스트레스 테스트 (동시 플레이어 수 확인)

---

## 지원되는 브라우저

- Chrome 50+
- Firefox 45+
- Safari 10+
- Edge 15+
- 모바일 브라우저 (iOS Safari, Chrome Android)

---

## 라이센스 및 지원

이 게임은 iframe 기반으로 설계되어 여러 웹사이트에 쉽게 임베드될 수 있습니다.

추가 질문이나 통합 지원이 필요한 경우 연락주세요.
