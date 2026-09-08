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

미리보기 창에서 `file:///C:/AI-Agent/ksk_coin/index.html` 을 연다.

**주의 — 같은 URL 로 다시 이동해야 파일이 다시 읽힌다.** 이 창은 파일을 `data:` URL 스냅샷으로
띄우므로 `location.reload()` 는 **디스크의 새 파일을 읽지 않는다.** 코드를 고친 뒤에는
반드시 `navigate` 를 다시 호출한다. 이걸 놓쳐 구버전을 검증한 적이 있다.

확인 방법: 고친 코드에만 있는 심볼이 존재하는지 본다.

```js
typeof myNewFunction    // "function" 이어야 한다
```

`localStorage` 는 `data:` URL 에서 차단된다. 예외가 나는 것이 정상이고, 코드가
`try/catch` 로 삼켜 화면이 깨지지 않아야 한다. **저장 기능 자체는 이 창에서 검증할 수 없다.**

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
| 1단계 CPU 점유 | 약 6% (PC) / 4% (모바일) |
| 1단계 풀 지분 | **4% ± 0.3** |
| 불변식 | `S.mined <= S.issued` — **반드시 true** |
| 시세 | 0.009~0.012원 구간 (희소성이 오르기 전) |
| 현금 | 갑자기 폭증·폭감하지 않는다 |

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
- **`ksk_coin` 폴더 안에서** `git add` 했는가 (루트 `C:\AI-Agent` 에서 하면 남의 저장소를 삼킨다)
- `git push` 는 **사용자 확인 없이 하지 않는다** — 공개 저장소다

## 8. 보고 형식

통과·실패를 값으로 보고한다. "동작합니다"라고만 쓰지 않는다.

```
지분 4.03% (설계 4%) · CPU 6% · 불변식 유지 · 콘솔 오류 0건
매입 → 보유 반영 → 매도 → 현금 회수 확인
```

실패한 항목은 숨기지 않고 그대로 적는다.
