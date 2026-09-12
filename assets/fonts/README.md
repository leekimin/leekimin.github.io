# 글꼴

블로그 전체와 진입 화면 모두 **둥근모꼴(DungGeunMo)** 하나를 씁니다.
도스 · PC통신 시절의 16px 격자 비트맵 글꼴이고, 영문 80 / 한글 160 units 로
정확히 반각 · 전각 고정폭이라 단말 화면 재현에 그대로 맞습니다.

> Public Domain · Kil Hyung-jin 제작, Kim Jung-tae · Darien Gavin Valentine 디자인

**16의 배수(16px · 32px)에서만 픽셀이 딱 떨어집니다.** 그 사이 크기는 흐려집니다.

| 파일 | 크기 | 쓰는 곳 |
|---|---|---|
| `DungGeunMo.woff2` | 924 KB | 블로그 전체 (제목 · 본문 · 코드) |
| `DungGeunMo-intro.woff2` | 5 KB | 진입 화면(`/`) 전용 부분집합 |
| `DungGeunMo.woff` / `.eot` | 1.6 MB | 구형 브라우저 대비 |

## 진입 화면 부분집합

`/` 의 접속 연출에는 **239자만 나옵니다.** 그 글자만 남겨 924KB → 5KB 로 줄였습니다.
같은 글꼴이라 블로그와 이어지는 느낌은 그대로입니다.

### 다시 만드는 법

**연출 문구를 고치면 글자가 모자랄 수 있습니다.** 없는 글자는 조용히 다른 글꼴로 찍히고,
그러면 글자 폭이 달라져 단말 화면의 열 정렬이 깨집니다. 문구를 바꿨다면 반드시 다시 만드세요.

```powershell
py -m pip install fonttools brotli

# 1) 화면에 나오는 글자 모으기
#    _includes/common/bbs-intro.html 의 S 배열 + 마크업 텍스트 + ASCII → glyphs.txt

# 2) 부분집합 생성
py -m fontTools.subset assets\fonts\DungGeunMo.woff2 `
   --text-file=glyphs.txt `
   --flavor=woff2 --layout-features= --no-hinting --desubroutinize `
   --output-file=assets\fonts\DungGeunMo-intro.woff2

# 3) 누락 확인 — '□' 하나만 빠지는 게 정상 (제목 표시줄은 시스템 글꼴을 쓴다)
```

### 이 글꼴에 없는 기호

`□ ▶ ▷ ◈ ◆ ◇ ▲ ▼` 는 둥근모꼴에 **없습니다.** 대신 이런 것들이 있습니다.

```
■ ● ○ ◎ ★ ☆ ※ ‣ » «  → ← ↑ ↓
┌ ┐ └ ┘ ├ ┤ ┬ ┴ ┼ ─ │  ═ ║ ╔ ╗ ╚ ╝  ▒ ░ ▓
```

연출 화면은 이 범위 안에서만 그립니다. 당시 화면도 같은 제약 아래 있었으니 오히려 맞습니다.
`■` 만 전각(160)이고 나머지 기호는 반각(80)이라 열을 맞출 때 주의하세요.

## 블로그 본문 글꼴

임의의 한글이 올라오므로 **부분집합으로 줄일 수 없습니다.** 대신 이렇게 다룹니다.

- `woff2`(924KB)를 먼저 참조합니다. 예전에는 저장소에 있는데도 참조하지 않아 `woff`(1.6MB)를 받았습니다
- `font-display: swap` — 받아오는 동안 대체 글꼴로 글자를 먼저 보여줍니다.
  기본값(`auto`)은 최대 3초간 글자를 감춥니다
- `_layouts/blog.html` 에서 `preload` 로 미리 받기 시작합니다

## bitter / junge / ubuntu-c

테마의 원래 글꼴입니다. 글꼴 목록에서 둥근모꼴 뒤에 있고 둥근모꼴이 필요한 글자를
모두 가지고 있어, 실제로 내려받아지지는 않습니다.
