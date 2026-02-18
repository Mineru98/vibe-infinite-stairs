# 에셋 관리 및 오디오 파이프라인 구현 계획서

> **프로젝트**: 무한 계단 (Infinite Stairs) — React 기반 웹 캐주얼 게임
> **작성일**: 2026-02-18
> **관련 PRD**: Task 2 (에셋 출처/라이선스), Task 3 (Scene 목록), Task 5 (컴포넌트 설계)

---

## 목차

1. [에셋 카테고리별 필요 목록](#1-에셋-카테고리별-필요-목록)
2. [추천 에셋 소스 및 라이선스 관리 전략](#2-추천-에셋-소스-및-라이선스-관리-전략)
3. [에셋 파이프라인 설계](#3-에셋-파이프라인-설계)
4. [스프라이트시트 / 텍스처 아틀라스 전략](#4-스프라이트시트--텍스처-아틀라스-전략)
5. [AudioManager 상세 설계](#5-audiomanager-상세-설계)
6. [에셋 프리로딩 시스템 구현 방안](#6-에셋-프리로딩-시스템-구현-방안)
7. [CREDITS / 라이선스 관리 체계](#7-credits--라이선스-관리-체계)
8. [핵심 코드 예시](#8-핵심-코드-예시)

---

## 1. 에셋 카테고리별 필요 목록

### 1.1 스프라이트 (캐릭터)

| 에셋 | 설명 | 프레임 수(예상) | 우선순위 |
|------|------|----------------|----------|
| 캐릭터 Idle | 대기 애니메이션 | 4~6 | 필수 |
| 캐릭터 Run/Walk | 이동 애니메이션 | 6~8 | 필수 |
| 캐릭터 Jump | 점프 상승 | 2~4 | 필수 |
| 캐릭터 Fall | 낙하 | 2~3 | 필수 |
| 캐릭터 Land | 착지 이펙트 | 2~3 | 권장 |
| 캐릭터 GameOver | 떨어지는/사망 | 3~4 | 필수 |

### 1.2 타일 (계단)

| 에셋 | 설명 | 우선순위 |
|------|------|----------|
| 기본 계단 타일 | 일반 스텝 (좌/우) | 필수 |
| 특수 계단 타일 | 부서지는/미끄러운/이동하는 계단 | 권장 |
| 계단 장식 | 테마별 변형 (눈, 사막, 용암 등) | 선택 |

### 1.3 배경

| 에셋 | 설명 | 우선순위 |
|------|------|----------|
| 배경 레이어 1 | 원경 (하늘/구름) — 패럴랙스 | 필수 |
| 배경 레이어 2 | 중경 (산/건물) — 패럴랙스 | 권장 |
| 배경 레이어 3 | 근경 (장식 요소) | 선택 |
| 테마 배경 | 스테이지/난이도별 변경 | 선택 |

### 1.4 UI

| 에셋 | 설명 | 우선순위 |
|------|------|----------|
| 버튼 (Play, Retry, Home, Settings) | 기본 + 눌림 상태 | 필수 |
| 아이콘 (Sound On/Off, Pause, Star) | 16~32px 아이콘셋 | 필수 |
| 점수판/프레임 | HUD 배경 패널 | 필수 |
| 로딩바 | Preload 화면용 | 필수 |
| 다이얼로그 프레임 | 팝업/모달 배경 | 권장 |

### 1.5 폰트

| 에셋 | 설명 | 우선순위 |
|------|------|----------|
| 메인 게임 폰트 | 한글 지원, 픽셀/캐주얼 스타일 | 필수 |
| 숫자 전용 폰트 | 점수 표시용 (모노스페이스 권장) | 권장 |

### 1.6 사운드

| 에셋 | 타입 | 포맷 | 우선순위 |
|------|------|------|----------|
| BGM — 메인 테마 | BGM | OGG + MP3 | 필수 |
| BGM — 게임플레이 | BGM | OGG + MP3 | 필수 |
| SFX — 점프 | SFX | OGG + MP3 | 필수 |
| SFX — 착지 | SFX | OGG + MP3 | 필수 |
| SFX — 게임오버 | SFX | OGG + MP3 | 필수 |
| SFX — 버튼 클릭 | SFX | OGG + MP3 | 필수 |
| SFX — 콤보/보너스 | SFX | OGG + MP3 | 권장 |
| SFX — 특수 계단 밟기 | SFX | OGG + MP3 | 권장 |

---

## 2. 추천 에셋 소스 및 라이선스 관리 전략

### 2.1 추천 소스 매트릭스

| 카테고리 | 1순위 소스 | 2순위 소스 | 라이선스 |
|----------|-----------|-----------|----------|
| 캐릭터/타일 스프라이트 | Kenney (CC0) | itch.io CC0 태그 | 상업적 사용 자유 |
| 배경 이미지 | Kenney (CC0) | Pixabay (Content License) | attribution 불필요 |
| UI 아이콘 | game-icons.net (CC BY 3.0) | Kenney (CC0) | CC BY는 크레딧 필수 |
| 폰트 | Google Fonts (SIL OFL) | — | 라이선스 파일 동봉 |
| BGM | Mixkit (Free License) | Pixabay (Content License) | attribution 불필요 |
| SFX | Mixkit (Free License) | Freesound (CC0 필터) | CC0만 사용 권장 |
| SFX (보조) | ZapSplat (Standard) | — | attribution 필수 |

### 2.2 라이선스 위험도 분류

```
[안전]  CC0, Pixabay Content License, Mixkit Free License
         → 크레딧 불필요, 상업적 사용 자유

[주의]  CC BY 3.0/4.0, ZapSplat Standard
         → 크레딧 표기 필수, 누락 시 라이선스 위반

[위험]  CC BY-SA, GPL, CC BY-NC
         → 상업 프로젝트에서 회피 (ShareAlike 파급, NC 제한)
```

### 2.3 운영 원칙

1. **CC0 또는 attribution 불필요 라이선스 우선 사용**
2. CC BY 에셋 사용 시 반드시 `CREDITS.md` 및 게임 내 Credits Scene에 기재
3. **CC BY-SA, GPL, CC BY-NC 에셋은 원칙적으로 사용 금지** (대체 에셋 탐색)
4. 에셋 도입 시 `licenses/` 폴더에 원본 라이선스 텍스트 사본 추가
5. 에셋 매니페스트(`asset-manifest.json`)에 출처/라이선스 메타데이터 기록

---

## 3. 에셋 파이프라인 설계

### 3.1 폴더 구조

```
public/
  assets/
    sprites/
      character/
        character-idle.png
        character-run.png
        character-jump.png
      stairs/
        stair-normal.png
        stair-breaking.png
      effects/
        land-dust.png
    tiles/
      stair-tile-set.png
    backgrounds/
      bg-sky.png
      bg-mountains.png
    ui/
      buttons/
        btn-play.png
        btn-play-pressed.png
      icons/
        icon-sound-on.png
        icon-sound-off.png
      panels/
        hud-frame.png
    fonts/
      game-font.woff2
      score-font.woff2
    audio/
      bgm/
        bgm-main.ogg
        bgm-main.mp3
        bgm-gameplay.ogg
        bgm-gameplay.mp3
      sfx/
        sfx-jump.ogg
        sfx-jump.mp3
        sfx-land.ogg
        sfx-land.mp3
        sfx-gameover.ogg
        sfx-gameover.mp3
        sfx-button.ogg
        sfx-button.mp3
    atlas/
      character-atlas.json
      character-atlas.png
      ui-atlas.json
      ui-atlas.png

src/
  assets/
    asset-manifest.ts      # 에셋 경로/메타데이터 중앙 관리
    asset-loader.ts         # 프리로딩 시스템
    audio-manager.ts        # 오디오 매니저

licenses/
  kenney-cc0.txt
  game-icons-cc-by-3.txt
  google-fonts-ofl.txt
  mixkit-license.txt

CREDITS.md
```

### 3.2 네이밍 컨벤션

| 규칙 | 예시 |
|------|------|
| 소문자 + 하이픈(kebab-case) | `character-idle.png` |
| 카테고리 접두사 | `bgm-`, `sfx-`, `bg-`, `btn-`, `icon-` |
| 아틀라스 접미사 | `-atlas.json`, `-atlas.png` |
| 오디오 이중 포맷 | `.ogg` + `.mp3` 쌍으로 제공 |
| 해상도 접미사 (필요 시) | `bg-sky@2x.png` |

### 3.3 최적화 전략

| 대상 | 최적화 방법 |
|------|------------|
| PNG 스프라이트 | TinyPNG/pngquant 압축, 불필요한 알파 제거 |
| 텍스처 아틀라스 | 2의 제곱 크기 (512x512, 1024x1024), POT 텍스처 |
| 배경 이미지 | WebP 포맷 우선 (fallback: PNG) |
| 오디오 | OGG Vorbis (주), MP3 (fallback). 모노 22kHz로 SFX 용량 절감 |
| 폰트 | WOFF2 포맷, 한글 서브셋 (사용 글자만 추출) |
| 전체 | gzip/brotli 압축 (서버/CDN 설정) |

---

## 4. 스프라이트시트 / 텍스처 아틀라스 전략

### 4.1 아틀라스 분류

| 아틀라스 이름 | 포함 내용 | 예상 크기 |
|--------------|----------|-----------|
| `character-atlas` | 캐릭터 모든 애니메이션 프레임 | 512x512 |
| `stairs-atlas` | 계단 타일 전체 변형 | 512x256 |
| `ui-atlas` | 버튼, 아이콘, 패널 | 512x512 |
| `effects-atlas` | 파티클, 착지 먼지, 점프 이펙트 | 256x256 |

### 4.2 아틀라스 포맷

JSON Hash 포맷 (TexturePacker 호환)을 사용한다.

```json
{
  "frames": {
    "character-idle-0": {
      "frame": { "x": 0, "y": 0, "w": 32, "h": 48 },
      "sourceSize": { "w": 32, "h": 48 }
    },
    "character-idle-1": {
      "frame": { "x": 32, "y": 0, "w": 32, "h": 48 },
      "sourceSize": { "w": 32, "h": 48 }
    }
  },
  "meta": {
    "image": "character-atlas.png",
    "size": { "w": 512, "h": 512 },
    "scale": "1"
  }
}
```

### 4.3 아틀라스 도구

| 도구 | 용도 | 비고 |
|------|------|------|
| **TexturePacker** (CLI/GUI) | 자동 아틀라스 생성 | 무료 버전 사용 가능, CLI 빌드 파이프라인 연동 |
| **free-tex-packer** | 오픈소스 대안 | npm 패키지로 빌드 스크립트에 포함 가능 |
| **Shoebox** | 무료 GUI 도구 | 수동 작업 시 사용 |

### 4.4 런타임 아틀라스 파싱

```typescript
// 아틀라스 JSON에서 프레임 데이터를 파싱하여 Canvas drawImage에 활용
interface AtlasFrame {
  frame: { x: number; y: number; w: number; h: number };
  sourceSize: { w: number; h: number };
}

function drawAtlasFrame(
  ctx: CanvasRenderingContext2D,
  atlasImage: HTMLImageElement,
  frame: AtlasFrame,
  dx: number,
  dy: number
): void {
  const { x, y, w, h } = frame.frame;
  ctx.drawImage(atlasImage, x, y, w, h, dx, dy, w, h);
}
```

---

## 5. AudioManager 상세 설계

### 5.1 아키텍처 개요

```
┌─────────────────────────────────────────────────┐
│                  AudioManager                    │
│                  (싱글톤)                         │
├─────────────────────────────────────────────────┤
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐ │
│  │ BGM 채널  │  │ SFX 풀   │  │ 모바일 Unlock │ │
│  │ (단일)    │  │ (다중)   │  │   Handler     │ │
│  └──────────┘  └──────────┘  └───────────────┘ │
│        │              │              │           │
│        └──────────────┴──────────────┘           │
│                       │                          │
│              ┌────────┴────────┐                 │
│              │  AudioContext   │                 │
│              │  (Web Audio API)│                 │
│              └─────────────────┘                 │
└─────────────────────────────────────────────────┘
```

### 5.2 Web Audio API 활용 구조

| 컴포넌트 | 역할 |
|----------|------|
| `AudioContext` | 전역 오디오 컨텍스트 (앱 당 1개) |
| `GainNode` (마스터) | 전체 볼륨 제어 |
| `GainNode` (BGM) | BGM 전용 볼륨 |
| `GainNode` (SFX) | SFX 전용 볼륨 |
| `AudioBufferSourceNode` | 각 사운드 재생 인스턴스 |

**노드 그래프:**

```
Source(BGM) → GainNode(BGM) ─┐
                              ├→ GainNode(Master) → destination
Source(SFX) → GainNode(SFX) ─┘
```

### 5.3 모바일 오디오 정책 처리

모바일 브라우저(특히 iOS Safari, Chrome)는 사용자 상호작용(터치/클릭) 없이 AudioContext를 시작할 수 없다. 이를 해결하기 위한 전략:

1. **AudioContext를 `suspended` 상태로 생성**
2. **첫 번째 사용자 입력 이벤트에서 `context.resume()` 호출**
3. **unlock 완료 후 이벤트 리스너 제거** (메모리 누수 방지)
4. **unlock 전 요청된 사운드는 큐에 저장**, unlock 후 재생

```
[앱 시작] → AudioContext 생성 (suspended)
    ↓
[사용자 터치/클릭] → context.resume() → 상태: running
    ↓
[이벤트 리스너 제거] → 큐에 쌓인 사운드 재생
```

### 5.4 SFX 풀링 시스템

동시에 여러 SFX(예: 연속 점프음)가 재생될 수 있으므로, 같은 사운드에 대해 동시 재생 수를 제한하는 풀링이 필요하다.

| 설정 | 값 |
|------|-----|
| 동일 SFX 최대 동시 재생 | 3 |
| SFX 풀 전체 최대 활성 소스 | 8 |
| 재생 완료 시 | 자동 풀 반환 |

### 5.5 BGM 관리

- **단일 BGM 채널**: 동시에 1곡만 재생
- **크로스페이드**: 씬 전환 시 페이드아웃(300ms) → 페이드인(300ms)
- **루프 재생**: `AudioBufferSourceNode.loop = true`
- **일시정지/재개**: Pause Overlay에서 BGM 일시정지 지원

### 5.6 설정 연동

| 설정 | 저장 위치 | 기본값 |
|------|----------|--------|
| 마스터 볼륨 | localStorage | 1.0 |
| BGM 볼륨 | localStorage | 0.7 |
| SFX 볼륨 | localStorage | 1.0 |
| BGM On/Off | localStorage | On |
| SFX On/Off | localStorage | On |

---

## 6. 에셋 프리로딩 시스템 구현 방안

### 6.1 로딩 전략

```
[Boot/Init Scene]
  └→ 디바이스 체크, 저장 데이터 로드

[Preload/Loading Scene]
  ├→ 필수 에셋 (Critical) — 즉시 로드
  │   ├ 캐릭터 아틀라스
  │   ├ 계단 아틀라스
  │   ├ UI 아틀라스
  │   ├ 게임 폰트
  │   ├ BGM (게임플레이)
  │   └ 핵심 SFX (점프, 착지, 게임오버)
  │
  └→ 지연 에셋 (Deferred) — 게임 시작 후 백그라운드 로드
      ├ 추가 배경 레이어
      ├ 특수 계단 텍스처
      ├ 보너스 SFX
      └ 테마별 에셋

[로딩 완료] → Title/Home Scene 전환
```

### 6.2 로딩 진행률 표시

- 전체 에셋 수 대비 완료 수로 퍼센트 계산
- 로딩바 UI + 팁 문구 표시
- 최소 로딩 시간(500ms) 설정 (깜빡임 방지)

### 6.3 에셋 타입별 로딩 방법

| 타입 | 로딩 방법 | 캐싱 |
|------|----------|------|
| 이미지/아틀라스 PNG | `new Image()` + `onload` | 브라우저 캐시 + 메모리 참조 |
| 아틀라스 JSON | `fetch()` + `json()` | 메모리 Map |
| 오디오 | `fetch()` + `arrayBuffer()` + `decodeAudioData()` | `AudioBuffer` Map |
| 폰트 | `FontFace` API + `document.fonts.add()` | 브라우저 폰트 캐시 |
| WASM (물리엔진) | `import()` 동적 로드 | 모듈 캐시 |

### 6.4 에러 처리

- 개별 에셋 로드 실패 시 재시도 (최대 3회)
- 재시도 실패 시 폴백 에셋 사용 또는 해당 에셋 건너뛰기
- 치명적 에셋(캐릭터, 계단 등) 실패 시 에러 화면 표시

---

## 7. CREDITS / 라이선스 관리 체계

### 7.1 CREDITS.md 구조

```markdown
# Credits

## Sprites & Tiles
- **[에셋 팩 이름]** by [제작자]
  - License: CC0
  - Source: https://kenney.nl/assets/...

## UI Icons
- **[아이콘셋 이름]** by [제작자]
  - License: CC BY 3.0
  - Source: https://game-icons.net/...

## Fonts
- **[폰트 이름]** by [제작자]
  - License: SIL Open Font License 1.1
  - Source: https://fonts.google.com/...

## Music
- **[곡 제목]** by [아티스트]
  - License: Mixkit Free License
  - Source: https://mixkit.co/...

## Sound Effects
- **[효과음 이름]** by [제작자]
  - License: CC0
  - Source: https://freesound.org/...

## Licenses
Full license texts are available in the `licenses/` directory.
```

### 7.2 게임 내 Credits Scene

- `SceneCredits` 컴포넌트에서 스크롤 가능한 크레딧 목록 표시
- `CREDITS.md`를 빌드 시 JSON으로 변환하거나, 별도 `credits-data.ts` 유지
- CC BY 에셋의 경우 저작자 링크를 클릭 가능하게 표시

### 7.3 licenses/ 폴더 관리

```
licenses/
  CC0-1.0.txt           # CC0 전문
  CC-BY-3.0.txt         # CC BY 3.0 전문
  SIL-OFL-1.1.txt       # SIL Open Font License 전문
  mixkit-license.txt    # Mixkit 라이선스 전문
  pixabay-license.txt   # Pixabay Content License 전문
```

### 7.4 에셋 도입 프로세스

```
1. 에셋 후보 선정
   ↓
2. 라이선스 확인 (CC0/CC BY만 허용, SA/NC/GPL 불가)
   ↓
3. 라이선스 텍스트 → licenses/ 폴더에 추가
   ↓
4. CREDITS.md에 에셋명/제작자/라이선스/링크 기재
   ↓
5. asset-manifest.ts에 경로 및 메타데이터 등록
   ↓
6. 에셋 파일을 네이밍 컨벤션에 맞춰 public/assets/ 배치
```

---

## 8. 핵심 코드 예시

### 8.1 에셋 매니페스트 (`asset-manifest.ts`)

```typescript
export interface AssetEntry {
  key: string;
  type: 'image' | 'atlas' | 'audio' | 'font';
  src: string;
  /** 아틀라스인 경우 JSON 경로 */
  atlasSrc?: string;
  /** 오디오 폴백 경로 */
  fallbackSrc?: string;
  /** 필수 에셋 여부 */
  critical: boolean;
  /** 라이선스 정보 */
  license: {
    type: string;
    author: string;
    source: string;
  };
}

export const ASSET_MANIFEST: AssetEntry[] = [
  // --- 스프라이트 아틀라스 ---
  {
    key: 'character-atlas',
    type: 'atlas',
    src: '/assets/atlas/character-atlas.png',
    atlasSrc: '/assets/atlas/character-atlas.json',
    critical: true,
    license: { type: 'CC0', author: 'Kenney', source: 'https://kenney.nl/assets/...' },
  },
  {
    key: 'ui-atlas',
    type: 'atlas',
    src: '/assets/atlas/ui-atlas.png',
    atlasSrc: '/assets/atlas/ui-atlas.json',
    critical: true,
    license: { type: 'CC0', author: 'Kenney', source: 'https://kenney.nl/assets/...' },
  },

  // --- 배경 ---
  {
    key: 'bg-sky',
    type: 'image',
    src: '/assets/backgrounds/bg-sky.png',
    critical: true,
    license: { type: 'CC0', author: 'Kenney', source: 'https://kenney.nl/assets/...' },
  },

  // --- 오디오 ---
  {
    key: 'bgm-gameplay',
    type: 'audio',
    src: '/assets/audio/bgm/bgm-gameplay.ogg',
    fallbackSrc: '/assets/audio/bgm/bgm-gameplay.mp3',
    critical: true,
    license: { type: 'Mixkit Free License', author: 'Mixkit', source: 'https://mixkit.co/...' },
  },
  {
    key: 'sfx-jump',
    type: 'audio',
    src: '/assets/audio/sfx/sfx-jump.ogg',
    fallbackSrc: '/assets/audio/sfx/sfx-jump.mp3',
    critical: true,
    license: { type: 'CC0', author: 'Unknown', source: 'https://freesound.org/...' },
  },
  {
    key: 'sfx-land',
    type: 'audio',
    src: '/assets/audio/sfx/sfx-land.ogg',
    fallbackSrc: '/assets/audio/sfx/sfx-land.mp3',
    critical: true,
    license: { type: 'CC0', author: 'Unknown', source: 'https://freesound.org/...' },
  },
  {
    key: 'sfx-gameover',
    type: 'audio',
    src: '/assets/audio/sfx/sfx-gameover.ogg',
    fallbackSrc: '/assets/audio/sfx/sfx-gameover.mp3',
    critical: true,
    license: { type: 'CC0', author: 'Unknown', source: 'https://freesound.org/...' },
  },

  // --- 폰트 ---
  {
    key: 'game-font',
    type: 'font',
    src: '/assets/fonts/game-font.woff2',
    critical: true,
    license: { type: 'SIL OFL 1.1', author: 'Google Fonts', source: 'https://fonts.google.com/...' },
  },
];
```

### 8.2 AssetLoader (`asset-loader.ts`)

```typescript
import { ASSET_MANIFEST, type AssetEntry } from './asset-manifest';

type ProgressCallback = (loaded: number, total: number, currentAsset: string) => void;

interface LoadedAssets {
  images: Map<string, HTMLImageElement>;
  atlasData: Map<string, Record<string, any>>;
  audioBuffers: Map<string, AudioBuffer>;
}

class AssetLoader {
  private assets: LoadedAssets = {
    images: new Map(),
    atlasData: new Map(),
    audioBuffers: new Map(),
  };

  private audioContext: AudioContext | null = null;

  setAudioContext(ctx: AudioContext): void {
    this.audioContext = ctx;
  }

  /**
   * 필수(critical) 에셋만 로드한다.
   * 로딩 진행률은 onProgress 콜백으로 전달한다.
   */
  async loadCritical(onProgress?: ProgressCallback): Promise<LoadedAssets> {
    const criticalAssets = ASSET_MANIFEST.filter((a) => a.critical);
    let loaded = 0;
    const total = criticalAssets.length;

    const tasks = criticalAssets.map(async (entry) => {
      await this.loadAsset(entry);
      loaded++;
      onProgress?.(loaded, total, entry.key);
    });

    await Promise.all(tasks);
    return this.assets;
  }

  /**
   * 지연(deferred) 에셋을 백그라운드로 로드한다.
   */
  async loadDeferred(): Promise<void> {
    const deferredAssets = ASSET_MANIFEST.filter((a) => !a.critical);
    await Promise.allSettled(deferredAssets.map((entry) => this.loadAsset(entry)));
  }

  private async loadAsset(entry: AssetEntry, retries = 3): Promise<void> {
    try {
      switch (entry.type) {
        case 'image':
          await this.loadImage(entry);
          break;
        case 'atlas':
          await this.loadAtlas(entry);
          break;
        case 'audio':
          await this.loadAudio(entry);
          break;
        case 'font':
          await this.loadFont(entry);
          break;
      }
    } catch (error) {
      if (retries > 0) {
        console.warn(`에셋 로드 재시도 (${entry.key}), 남은 횟수: ${retries - 1}`);
        await this.loadAsset(entry, retries - 1);
      } else {
        console.error(`에셋 로드 실패: ${entry.key}`, error);
        if (entry.critical) throw error;
      }
    }
  }

  private loadImage(entry: AssetEntry): Promise<void> {
    return new Promise((resolve, reject) => {
      const img = new Image();
      img.onload = () => {
        this.assets.images.set(entry.key, img);
        resolve();
      };
      img.onerror = reject;
      img.src = entry.src;
    });
  }

  private async loadAtlas(entry: AssetEntry): Promise<void> {
    // 이미지 로드
    await this.loadImage(entry);

    // JSON 데이터 로드
    if (entry.atlasSrc) {
      const response = await fetch(entry.atlasSrc);
      const data = await response.json();
      this.assets.atlasData.set(entry.key, data);
    }
  }

  private async loadAudio(entry: AssetEntry): Promise<void> {
    if (!this.audioContext) return;

    let response: Response;
    try {
      response = await fetch(entry.src);
    } catch {
      if (entry.fallbackSrc) {
        response = await fetch(entry.fallbackSrc);
      } else {
        throw new Error(`오디오 로드 실패: ${entry.key}`);
      }
    }

    const arrayBuffer = await response.arrayBuffer();
    const audioBuffer = await this.audioContext.decodeAudioData(arrayBuffer);
    this.assets.audioBuffers.set(entry.key, audioBuffer);
  }

  private async loadFont(entry: AssetEntry): Promise<void> {
    const fontFace = new FontFace(entry.key, `url(${entry.src})`);
    const loadedFont = await fontFace.load();
    document.fonts.add(loadedFont);
  }

  getImage(key: string): HTMLImageElement | undefined {
    return this.assets.images.get(key);
  }

  getAtlasData(key: string): Record<string, any> | undefined {
    return this.assets.atlasData.get(key);
  }

  getAudioBuffer(key: string): AudioBuffer | undefined {
    return this.assets.audioBuffers.get(key);
  }
}

export const assetLoader = new AssetLoader();
```

### 8.3 AudioManager (`audio-manager.ts`)

```typescript
type SoundKey = string;

interface ActiveSFX {
  source: AudioBufferSourceNode;
  key: string;
}

class AudioManager {
  private context: AudioContext | null = null;
  private masterGain: GainNode | null = null;
  private bgmGain: GainNode | null = null;
  private sfxGain: GainNode | null = null;

  private bgmSource: AudioBufferSourceNode | null = null;
  private currentBgmKey: string | null = null;

  private activeSfx: ActiveSFX[] = [];
  private readonly MAX_CONCURRENT_SFX = 8;
  private readonly MAX_SAME_SFX = 3;

  private unlocked = false;
  private pendingQueue: Array<() => void> = [];

  private audioBuffers: Map<string, AudioBuffer> = new Map();

  // --- 볼륨 설정 ---
  private settings = {
    masterVolume: 1.0,
    bgmVolume: 0.7,
    sfxVolume: 1.0,
    bgmEnabled: true,
    sfxEnabled: true,
  };

  /**
   * AudioContext를 생성하고 모바일 unlock 핸들러를 등록한다.
   */
  init(): void {
    this.context = new (window.AudioContext || (window as any).webkitAudioContext)();

    // 노드 그래프 구성
    this.masterGain = this.context.createGain();
    this.bgmGain = this.context.createGain();
    this.sfxGain = this.context.createGain();

    this.bgmGain.connect(this.masterGain);
    this.sfxGain.connect(this.masterGain);
    this.masterGain.connect(this.context.destination);

    // 저장된 설정 복원
    this.loadSettings();
    this.applyVolumes();

    // 모바일 오디오 unlock
    if (this.context.state === 'suspended') {
      this.setupUnlockHandler();
    } else {
      this.unlocked = true;
    }
  }

  /**
   * 외부(AssetLoader)에서 디코딩한 AudioBuffer를 등록한다.
   */
  registerBuffer(key: string, buffer: AudioBuffer): void {
    this.audioBuffers.set(key, buffer);
  }

  /**
   * 모든 디코딩된 버퍼를 일괄 등록한다.
   */
  registerBuffers(buffers: Map<string, AudioBuffer>): void {
    buffers.forEach((buffer, key) => this.audioBuffers.set(key, buffer));
  }

  // ─── 모바일 Unlock ───

  private setupUnlockHandler(): void {
    const events = ['touchstart', 'touchend', 'mousedown', 'click', 'keydown'];

    const unlock = async () => {
      if (this.context && this.context.state === 'suspended') {
        try {
          await this.context.resume();
        } catch (e) {
          console.warn('AudioContext resume 실패:', e);
          return;
        }
      }

      this.unlocked = true;

      // 이벤트 리스너 정리
      events.forEach((evt) => document.removeEventListener(evt, unlock, true));

      // 대기열 재생
      this.pendingQueue.forEach((fn) => fn());
      this.pendingQueue = [];
    };

    events.forEach((evt) => document.addEventListener(evt, unlock, { once: false, capture: true }));
  }

  // ─── BGM ───

  playBGM(key: SoundKey): void {
    if (!this.settings.bgmEnabled) return;

    const play = () => {
      if (!this.context || !this.bgmGain) return;

      const buffer = this.audioBuffers.get(key);
      if (!buffer) {
        console.warn(`BGM 버퍼를 찾을 수 없음: ${key}`);
        return;
      }

      // 기존 BGM 페이드아웃 후 교체
      if (this.bgmSource) {
        this.fadeOut(this.bgmGain, 0.3, () => {
          this.bgmSource?.stop();
          this.startBGM(key, buffer);
        });
      } else {
        this.startBGM(key, buffer);
      }
    };

    if (!this.unlocked) {
      this.pendingQueue.push(play);
    } else {
      play();
    }
  }

  private startBGM(key: string, buffer: AudioBuffer): void {
    if (!this.context || !this.bgmGain) return;

    const source = this.context.createBufferSource();
    source.buffer = buffer;
    source.loop = true;
    source.connect(this.bgmGain);
    source.start(0);

    this.bgmSource = source;
    this.currentBgmKey = key;

    // 페이드인
    this.fadeIn(this.bgmGain, 0.3);
  }

  stopBGM(): void {
    if (this.bgmSource && this.bgmGain) {
      this.fadeOut(this.bgmGain, 0.3, () => {
        this.bgmSource?.stop();
        this.bgmSource = null;
        this.currentBgmKey = null;
      });
    }
  }

  pauseBGM(): void {
    if (this.context && this.context.state === 'running') {
      this.context.suspend();
    }
  }

  resumeBGM(): void {
    if (this.context && this.context.state === 'suspended' && this.unlocked) {
      this.context.resume();
    }
  }

  // ─── SFX ───

  playSFX(key: SoundKey): void {
    if (!this.settings.sfxEnabled) return;

    const play = () => {
      if (!this.context || !this.sfxGain) return;

      const buffer = this.audioBuffers.get(key);
      if (!buffer) {
        console.warn(`SFX 버퍼를 찾을 수 없음: ${key}`);
        return;
      }

      // 동시 재생 수 제한
      if (this.activeSfx.length >= this.MAX_CONCURRENT_SFX) return;

      const sameCount = this.activeSfx.filter((s) => s.key === key).length;
      if (sameCount >= this.MAX_SAME_SFX) return;

      const source = this.context.createBufferSource();
      source.buffer = buffer;
      source.connect(this.sfxGain);

      const entry: ActiveSFX = { source, key };
      this.activeSfx.push(entry);

      source.onended = () => {
        const idx = this.activeSfx.indexOf(entry);
        if (idx !== -1) this.activeSfx.splice(idx, 1);
      };

      source.start(0);
    };

    if (!this.unlocked) {
      this.pendingQueue.push(play);
    } else {
      play();
    }
  }

  // ─── 볼륨 & 페이드 ───

  setMasterVolume(value: number): void {
    this.settings.masterVolume = Math.max(0, Math.min(1, value));
    this.applyVolumes();
    this.saveSettings();
  }

  setBGMVolume(value: number): void {
    this.settings.bgmVolume = Math.max(0, Math.min(1, value));
    this.applyVolumes();
    this.saveSettings();
  }

  setSFXVolume(value: number): void {
    this.settings.sfxVolume = Math.max(0, Math.min(1, value));
    this.applyVolumes();
    this.saveSettings();
  }

  toggleBGM(enabled: boolean): void {
    this.settings.bgmEnabled = enabled;
    if (!enabled) this.stopBGM();
    this.saveSettings();
  }

  toggleSFX(enabled: boolean): void {
    this.settings.sfxEnabled = enabled;
    this.saveSettings();
  }

  private applyVolumes(): void {
    if (this.masterGain) this.masterGain.gain.value = this.settings.masterVolume;
    if (this.bgmGain) this.bgmGain.gain.value = this.settings.bgmVolume;
    if (this.sfxGain) this.sfxGain.gain.value = this.settings.sfxVolume;
  }

  private fadeIn(gainNode: GainNode, duration: number): void {
    if (!this.context) return;
    gainNode.gain.setValueAtTime(0, this.context.currentTime);
    gainNode.gain.linearRampToValueAtTime(
      this.settings.bgmVolume,
      this.context.currentTime + duration
    );
  }

  private fadeOut(gainNode: GainNode, duration: number, onComplete?: () => void): void {
    if (!this.context) return;
    gainNode.gain.setValueAtTime(gainNode.gain.value, this.context.currentTime);
    gainNode.gain.linearRampToValueAtTime(0, this.context.currentTime + duration);
    if (onComplete) {
      setTimeout(onComplete, duration * 1000);
    }
  }

  // ─── 설정 영속화 ───

  private saveSettings(): void {
    try {
      localStorage.setItem('audio-settings', JSON.stringify(this.settings));
    } catch {
      // localStorage 사용 불가 환경 무시
    }
  }

  private loadSettings(): void {
    try {
      const saved = localStorage.getItem('audio-settings');
      if (saved) {
        Object.assign(this.settings, JSON.parse(saved));
      }
    } catch {
      // 파싱 실패 시 기본값 유지
    }
  }

  // ─── 정리 ───

  destroy(): void {
    this.stopBGM();
    this.activeSfx.forEach((s) => s.source.stop());
    this.activeSfx = [];
    this.context?.close();
    this.context = null;
  }
}

export const audioManager = new AudioManager();
```

### 8.4 Preload Scene 통합 예시 (`SceneLoading.tsx`)

```tsx
import { useEffect, useState } from 'react';
import { assetLoader } from '../assets/asset-loader';
import { audioManager } from '../assets/audio-manager';

interface SceneLoadingProps {
  onLoadComplete: () => void;
}

export function SceneLoading({ onLoadComplete }: SceneLoadingProps) {
  const [progress, setProgress] = useState(0);
  const [tip] = useState(() => getRandomTip());

  useEffect(() => {
    let cancelled = false;

    async function load() {
      // 1. AudioManager 초기화
      audioManager.init();
      assetLoader.setAudioContext(
        (audioManager as any).context // 내부 context 참조
      );

      // 2. 필수 에셋 로드
      const startTime = Date.now();

      await assetLoader.loadCritical((loaded, total, _key) => {
        if (!cancelled) {
          setProgress(Math.round((loaded / total) * 100));
        }
      });

      // 3. AudioBuffer를 AudioManager에 등록
      audioManager.registerBuffers(
        (assetLoader as any).assets.audioBuffers
      );

      // 4. 최소 로딩 시간 보장 (깜빡임 방지)
      const elapsed = Date.now() - startTime;
      const minTime = 500;
      if (elapsed < minTime) {
        await new Promise((r) => setTimeout(r, minTime - elapsed));
      }

      // 5. 지연 에셋은 백그라운드 로드
      assetLoader.loadDeferred();

      if (!cancelled) {
        onLoadComplete();
      }
    }

    load().catch(console.error);

    return () => {
      cancelled = true;
    };
  }, [onLoadComplete]);

  return (
    <div className="scene-loading">
      <div className="loading-bar-container">
        <div className="loading-bar" style={{ width: `${progress}%` }} />
      </div>
      <p className="loading-percent">{progress}%</p>
      <p className="loading-tip">{tip}</p>
    </div>
  );
}

const TIPS = [
  '계단을 빠르게 올라갈수록 더 높은 점수를 얻을 수 있어요!',
  '리듬감 있게 터치하면 콤보 보너스가 쌓여요!',
  '특수 계단을 조심하세요!',
];

function getRandomTip(): string {
  return TIPS[Math.floor(Math.random() * TIPS.length)];
}
```

### 8.5 AudioManager React Hook (`useAudio.ts`)

```typescript
import { useCallback } from 'react';
import { audioManager } from '../assets/audio-manager';

/**
 * 게임 컴포넌트에서 오디오를 간편하게 사용하기 위한 훅
 */
export function useAudio() {
  const playSFX = useCallback((key: string) => {
    audioManager.playSFX(key);
  }, []);

  const playBGM = useCallback((key: string) => {
    audioManager.playBGM(key);
  }, []);

  const stopBGM = useCallback(() => {
    audioManager.stopBGM();
  }, []);

  const setMasterVolume = useCallback((v: number) => {
    audioManager.setMasterVolume(v);
  }, []);

  const setBGMVolume = useCallback((v: number) => {
    audioManager.setBGMVolume(v);
  }, []);

  const setSFXVolume = useCallback((v: number) => {
    audioManager.setSFXVolume(v);
  }, []);

  const toggleBGM = useCallback((enabled: boolean) => {
    audioManager.toggleBGM(enabled);
  }, []);

  const toggleSFX = useCallback((enabled: boolean) => {
    audioManager.toggleSFX(enabled);
  }, []);

  return {
    playSFX,
    playBGM,
    stopBGM,
    setMasterVolume,
    setBGMVolume,
    setSFXVolume,
    toggleBGM,
    toggleSFX,
  };
}
```

---

## 부록: 체크리스트

### 에셋 준비 체크리스트

- [ ] 캐릭터 스프라이트시트 (Idle, Run, Jump, Fall, Land, GameOver)
- [ ] 계단 타일셋 (기본, 특수)
- [ ] 배경 레이어 (최소 1장, 권장 2~3장)
- [ ] UI 아틀라스 (버튼, 아이콘, 패널)
- [ ] 게임 폰트 (한글 지원, WOFF2)
- [ ] BGM 2곡 (메인 테마, 게임플레이)
- [ ] 핵심 SFX 4종 (점프, 착지, 게임오버, 버튼 클릭)
- [ ] 모든 에셋 라이선스 확인 완료
- [ ] CREDITS.md 작성 완료
- [ ] licenses/ 폴더에 라이선스 텍스트 추가 완료

### 구현 체크리스트

- [ ] `asset-manifest.ts` 작성
- [ ] `AssetLoader` 구현 (프리로드 + 지연 로드)
- [ ] `AudioManager` 구현 (Web Audio API + 모바일 unlock)
- [ ] `SceneLoading` 구현 (로딩바 + 진행률)
- [ ] `useAudio` 훅 구현
- [ ] 아틀라스 빌드 파이프라인 설정
- [ ] 오디오 OGG + MP3 이중 포맷 준비
- [ ] 폰트 서브셋팅 완료
- [ ] 에셋 최적화 (압축, POT 텍스처)
- [ ] 모바일 오디오 unlock 테스트 (iOS Safari, Chrome)
