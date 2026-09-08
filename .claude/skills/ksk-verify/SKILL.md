---
name: ksk-verify
description: ksk_coin 을 실제로 브라우저에서 돌려 검증한다 — 문법 검사, 콘솔 오류, 경제 불변식, 채굴 지분, 모바일 레이아웃. 모든 ksk-* 작업을 끝낼 때 사용하고, "검증해줘", "제대로 돌아가나", "테스트" 같은 요청에도 사용한다.
---

# ksk_coin 검증 절차

작업을 끝냈다고 말하기 전에 이 절차를 통과해야 한다. **"문법이 통과했다"는 검증이 아니다.**

## 1. 문법 검사

```bash
node -e "const s=require('fs').readFileSync('index.html','utf8');const m=s.match(/<script>([\s\S]*)<\/script>/);new Function(m[1]);console.log('JS 문법 통과')"
```

## 2. 구역 마커가 온전한지

```bash
grep -c "==== \[F" index.html
```

**30 이어야 한다** (5개 기능 × 시작/끝 × CSS/패널/JS). 줄어들었으면 마커를 지운 것이므로 되살린다.

## 3. 브라우저에서 실제로 돌린다

미리보기 창에서 이 저장소의 `index.html` 을 `file://` 로 연다.

**주의 — 같은 URL 로 다시 이동해야 파일이 다시 읽힌다.** 이 창은 파일을 `data:` URL 스냅샷으로
띄우므로 `location.reload()` 는 **디스크의 새 파일을 읽지 않는다.** 코드를 고친 뒤에는
반드시 `navigate` 를 다시 호출한다. 이걸 놓쳐 구버전을 검증한 적이 있다.

확인 방법: 고친 코드에만 있는 심볼이 존재하는지 본다.

```js
typeof myNewFunction    // "function" 이어야 한다
```

`localStorage` 는 `data:` URL 에서 차단된다. 예외가 나는 것이 정상이고, 코드가
`try/catch` 로 삼켜 화면이 깨지지 않아야 한다. **저장 기능 자체는 이 창에서 검증할 수 없다.**

## 3.5 게임 루프가 실제로 등록됐는지 (놓치기 쉽다)

로드 중 오류가 나면 스크립트가 끊겨 **맨 아래 `setInterval` 이 등록되지 않는다.**
그런데 화면은 초기 HTML 이라 멀쩡해 보이고, 콘솔 캡처에도 안 잡힐 수 있다.

```js
typeof last            // "number" 여야 한다. "undefined" 면 세계가 멈춘 것이다
```

`undefined` 면 초기화 함수를 직접 불러 오류를 꺼낸다.

```js
let err=null; try{ initTabs(); }catch(e){ err=e.name+': '+e.message; } err
```

`const`/`let` 을 쓰는 함수를 **선언보다 위에서 호출하면** TDZ 오류가 난다
(`Cannot access 'X' before initialization`). 실제로 이걸로 세계가 멈춘 적이 있다.
초기화 호출은 **스크립트 맨 끝**에 둔다.

## 3.6 화면을 눈으로 본다 (상태값만 읽지 말 것)

`JSON.stringify(S)` 로 상태만 확인하면 **표시 버그를 전부 놓친다.**
필드 이름을 바꾼 뒤 `undefined` 나 `NaN` 이 화면에 찍히는 것이 대표적이다.

```js
[...document.querySelectorAll('#rigs .rig')].map(d => d.textContent)
```

반드시 **스크린샷을 한 장 찍고**, 그 안에 `undefined` `NaN` `null` `[object`
가 없는지 확인한다. 실제로 `undefinedkW` 와 `손익분기 시세 NaN원` 을 이렇게 잡았다.

## 3.7 "켜져 있다" ≠ "돌고 있다" — 요금만 새는지 본다

채굴을 켰는데 해시가 안 도는 상황이 실제로 있었다(라이브 사이트에서 해시 0회인데 현금 66원 감소).
브라우저가 `requestAnimationFrame` 을 멈추면 채굴은 서는데 `setInterval` 기반 요금은 계속 나간다.

`watchdog()` 이 3초 넘게 해시 진행이 없으면 채굴을 접는다. 이게 실제로 발동하는지 시험한다.

```js
M.on = false;                                  // 먼저 rAF 루프를 정말 죽인다
// (1~2초 기다린 뒤 M.raf === 0 을 확인한다)
M.on = true; M.progressAt = performance.now() - 5000;   // 진행이 멈춘 상태로 위조
// 3초 기다린 뒤
JSON.stringify({fired: !M.on, spent: 500000 - S.cash})  // fired true, spent 0 이어야 한다
```

**함정**: 루프를 죽이지 않고 `progressAt` 만 위조하면 살아 있는 루프가 매 프레임 덮어써서
감시견이 발동하지 않는다. 그건 감시견 버그가 아니라 시험이 틀린 것이다.

## 3.8 검증을 무효로 만드는 함정 셋 (오늘 다섯 번 밟았다)

**A. 렌더 주기(500ms) 전에 DOM 을 읽으면 낡은 값이 나온다.**
`renderHooks` 는 500ms 주기로 돈다. 상태를 바꾸거나 버튼을 누른 **직후 동기적으로**
`textContent` 를 읽으면 이전 렌더의 값이다. 코드 버그로 오인하기 쉽다.
→ DOM 을 읽기 전에 **2~3초 기다린다.** 실제로 순자산 `0원`, 합격률 `47%`,
시계 `1월 2일` 을 이렇게 오진했다.

**B. 같은 URL 로 `navigate` 하면 페이지가 재사용돼 `S` 상태가 남는다.**
코드만 낡는 게 아니라 **상태도 남는다.** 앞 테스트의 `S.over = true` 가 남아 있어
파산 회생이 "작동하지 않는" 것처럼 보였다 — 실제로는 `recover()` 첫 줄에서
정상적으로 리턴한 것이었다.
→ 검증 시작 시 **`S.over` 와 `S.cash` 를 먼저 확인**하고, 필요하면 아래로 초기화한다.

**C. `location.reload()` 는 `data:` URL 에서 브라우저가 차단한다.**
`Not allowed to navigate top frame to data URL` 오류가 나고 **조용히 실패**한다.
리로드로 상태를 초기화할 수 없다.
→ 상태 초기화는 이렇게 한다.

```js
S = freshState(); M.on = false; M.raf = 0; S.baseHash = null;
document.getElementById('ovEnd').classList.add('hide');
```

## 4. 돌려 보고 상태를 읽는다

```js
document.getElementById('bStart').click();
document.getElementById('bMine').click();
```

10초쯤 기다린 뒤 상태를 읽는다.

```js
JSON.stringify({
  mode: S.mode,
  cpuPct: (hashMs()/(16.7*DUTY)*100).toFixed(1)+'%',
  share: +(share()*100).toFixed(2),
  issued: Math.round(S.issued),
  mine: Math.round(S.mined),
  invariantOk: S.mined <= S.issued,
  cash: Math.round(S.cash),
  price: +S.price.toFixed(5)
})
```

### 통과 기준

| 항목 | 기대값 |
|---|---|
| 콘솔 오류 | **0건** |
| `typeof last` | **"number"** — 게임 루프가 돌고 있다 |
| 화면에 `undefined`/`NaN` | **없음** (스크린샷으로 확인) |
| 1단계 CPU 점유 | 약 6% (PC) / 4% (모바일) |
| 1단계 풀 지분 | **4% ± 0.3** |
| 불변식 | `S.mined <= S.issued` — **반드시 true** |
| 시세 | 0.009~0.012원 구간 (희소성이 오르기 전) |
| 현금 | 갑자기 폭증·폭감하지 않는다 |
| 순현금흐름 | `netFlowPerSec()` 이 실측 현금 변화와 ±2% 안에서 일치 |
| 이중 청구 | 층을 추가한 뒤에도 위 일치가 유지된다 |

지분이 4% 에서 크게 벗어나면 **네트워크 규모 계산이 오염된 것**이다. 백서 §4.2 의 폭주 고리를
다시 만든 것일 수 있으니 `S.issued` / `S.mined` 를 섞어 쓰지 않았는지 확인한다.

## 5. 지분증명 전환

```js
merge();
document.getElementById('bMine').click();
```

| 항목 | 기대값 |
|---|---|
| `M.raf` | **0** — 작업증명 루프가 멈춰야 한다 |
| `M.hashes` (머지 후) | 0 에서 시작해 12초마다 1 증가 |
| `stakeShare()` | 예치량에 비례. 80% 같은 값이 나오면 폭주 고리다 |

## 6. 모바일

뷰포트를 `mobile`(375×812)로 바꾼 뒤 확인한다.

```js
JSON.stringify({
  hScroll: document.documentElement.scrollWidth > innerWidth,   // false 여야 한다
  btnH: Math.round(document.getElementById('bMine').getBoundingClientRect().height),
  budget: hashMs()
})
```

| 항목 | 기대값 |
|---|---|
| 가로 스크롤 | **없음** |
| 버튼 높이 | 46px 이상 |
| 해시 예산 | 2ms (모바일 판정) |

**뷰포트 에뮬레이션 중에는 좌표 클릭이 스케일링 때문에 엇나간다.** `ref` 로 클릭하거나
`element.click()` 을 쓴다. 검증이 끝나면 `desktop` 프리셋으로 되돌린다.

## 7. 문서와 커밋

- `PRD.md` §6 의 해당 항목을 `⬜ → ✅` 로 갱신했는가
- 경제 규칙을 바꿨다면 `WHITEPAPER.md` 의 해당 절도 고쳤는가
- 커밋 메시지가 한국어로 **무엇을 왜** 바꿨는지 말하는가
- **이 저장소 폴더 안에서** `git add` 했는가 (상위 작업 폴더에서 하면 남의 저장소를 삼킨다)
- `git push` 는 **사용자 확인 없이 하지 않는다** — 공개 저장소다

## 8. 보고 형식

통과·실패를 값으로 보고한다. "동작합니다"라고만 쓰지 않는다.

```
지분 4.03% (설계 4%) · CPU 6% · 불변식 유지 · 콘솔 오류 0건
매입 → 보유 반영 → 매도 → 현금 회수 확인
```

실패한 항목은 숨기지 않고 그대로 적는다.
