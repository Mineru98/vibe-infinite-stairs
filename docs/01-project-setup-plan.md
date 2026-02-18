# 프로젝트 초기 설정 및 기술 스택 구현 계획서

> **프로젝트명:** Infinite Stairs (무한 계단)
> **장르:** 물리 기반 캐주얼 아케이드 게임
> **기술 스택:** React + TypeScript + Vite + Matter.js + Zustand + Canvas 2D API
> **패키지 매니저:** bun
> **작성일:** 2026-02-18

---

## 1. 프로젝트 초기화

### 1.1 Vite 프로젝트 생성[완료]

```bash
bun create vite infinite-stairs --template react-ts
cd infinite-stairs
```

위 명령어를 실행하면 React + TypeScript 템플릿 기반의 Vite 프로젝트가 생성된다. 생성 직후 기본 구조는 다음과 같다.

```
infinite-stairs/
├── index.html          # 진입점 HTML
├── package.json        # 프로젝트 메타데이터 및 의존성
├── tsconfig.json       # TypeScript 설정
├── tsconfig.node.json  # Node 환경용 TypeScript 설정
├── vite.config.ts      # Vite 빌드 설정
├── public/
│   └── vite.svg
└── src/
    ├── App.tsx          # 루트 컴포넌트
    ├── main.tsx         # 앱 엔트리포인트
    ├── index.css        # 글로벌 스타일
    ├── App.css
    └── vite-env.d.ts    # Vite 타입 선언
```

### 1.2 기본 구조 설명

- **`index.html`**: Vite의 진입점. `<div id="root">` 에 React 앱이 마운트된다. 모바일 게임 최적화를 위한 viewport 메타 태그를 여기에 추가한다.
- **`src/main.tsx`**: ReactDOM을 통해 `App` 컴포넌트를 렌더링하는 엔트리포인트.
- **`vite.config.ts`**: 빌드 설정, 경로 별칭(alias), 청크 분할 등을 구성한다.

---

## 2. 핵심 의존성 목록

### 2.1 런타임 의존성

| 패키지 | 권장 버전 | 용도 |
|---|---|---|
| `react` | ^19.x | UI 렌더링 라이브러리 |
| `react-dom` | ^19.x | React DOM 렌더러 |
| `matter-js` | ^0.20.x | 2D 물리 엔진 (강체, 충돌, 중력 시뮬레이션) |
| `zustand` | ^5.x | 경량 상태 관리 라이브러리 |

### 2.2 개발 의존성

| 패키지 | 권장 버전 | 용도 |
|---|---|---|
| `typescript` | ^5.x | 정적 타입 시스템 |
| `vite` | ^6.x | 번들러 및 개발 서버 |
| `@vitejs/plugin-react` | ^4.x | Vite용 React 플러그인 (Fast Refresh) |
| `@types/react` | ^19.x | React 타입 정의 |
| `@types/react-dom` | ^19.x | React DOM 타입 정의 |
| `@types/matter-js` | ^0.19.x | Matter.js 타입 정의 |
| `eslint` | ^9.x | JavaScript/TypeScript 린터 |
| `prettier` | ^3.x | 코드 포맷터 |
| `@eslint/js` | ^9.x | ESLint 기본 규칙 |
| `typescript-eslint` | ^8.x | TypeScript ESLint 파서 및 규칙 |
| `eslint-plugin-react-hooks` | ^5.x | React Hooks 린트 규칙 |
| `eslint-plugin-react-refresh` | ^0.4.x | React Fast Refresh 린트 규칙 |

### 2.3 오디오 전략

오디오 처리는 두 가지 선택지가 있다.

**옵션 A: Howler.js 사용**
```bash
bun add howler
bun add -d @types/howler
```
- 크로스 브라우저 호환성이 뛰어남
- 스프라이트 오디오, 페이드, 공간 오디오 등 풍부한 기능
- 번들 크기 약 10KB (gzip)

**옵션 B: Web Audio API 직접 사용**
```typescript
// src/audio/AudioManager.ts
class AudioManager {
  private context: AudioContext;
  private buffers: Map<string, AudioBuffer> = new Map();

  constructor() {
    this.context = new AudioContext();
  }

  async loadSound(name: string, url: string): Promise<void> {
    const response = await fetch(url);
    const arrayBuffer = await response.arrayBuffer();
    const audioBuffer = await this.context.decodeAudioData(arrayBuffer);
    this.buffers.set(name, audioBuffer);
  }

  play(name: string, volume = 1.0): void {
    const buffer = this.buffers.get(name);
    if (!buffer) return;

    const source = this.context.createBufferSource();
    const gainNode = this.context.createGain();
    gainNode.gain.value = volume;

    source.buffer = buffer;
    source.connect(gainNode);
    gainNode.connect(this.context.destination);
    source.start(0);
  }
}
```
- 외부 의존성 없음
- 번들 크기 절약
- 세밀한 제어 가능
- 직접 구현해야 하므로 개발 비용 증가

> **권장:** 프로토타입 단계에서는 Web Audio API를 직접 사용하고, 필요 시 Howler.js로 마이그레이션한다.

---

## 3. 디렉터리 구조 설계

```
src/
├── scenes/           # 게임 씬 (화면 단위)
│   ├── BootScene.ts          # 엔진 초기화, 최소 에셋 로드
│   ├── LoadingScene.ts       # 에셋 프리로드, 프로그레스 바
│   ├── HomeScene.ts          # 메인 메뉴 UI
│   ├── GameScene.ts          # 핵심 게임플레이 루프
│   ├── ResultScene.ts        # 게임 오버, 점수 표시
│   ├── SettingsScene.ts      # 설정 (사운드, 품질)
│   └── CreditsScene.ts       # 크레딧 표시
│
├── engine/           # 게임 엔진 코어
│   ├── PhysicsEngine.ts      # Matter.js 래핑, 월드 생성/관리
│   ├── GameLoop.ts           # requestAnimationFrame 기반 루프
│   ├── Renderer.ts           # Canvas 2D 렌더링 파이프라인
│   └── SceneManager.ts       # 씬 전환 및 생명주기 관리
│
├── components/       # React UI 컴포넌트
│   ├── Canvas.tsx            # 메인 Canvas 엘리먼트
│   ├── HUD.tsx               # 점수, 콤보 등 인게임 UI
│   ├── PauseMenu.tsx         # 일시정지 오버레이
│   ├── StartButton.tsx       # 게임 시작 버튼
│   └── ScoreDisplay.tsx      # 점수 표시 위젯
│
├── stores/           # Zustand 상태 저장소
│   ├── gameStore.ts          # 게임 상태 (점수, 레벨, 진행도)
│   └── settingsStore.ts      # 설정 상태 (사운드, 품질, 언어)
│
├── assets/           # 에셋 관리
│   ├── manifest.ts           # 에셋 목록 및 경로 정의
│   └── AssetLoader.ts        # 에셋 프리로더
│
├── input/            # 입력 처리
│   └── InputController.ts    # Pointer Events 기반 입력 통합
│
├── audio/            # 오디오 시스템
│   └── AudioManager.ts       # 사운드 재생 및 관리
│
├── config/           # 설정 및 상수
│   ├── gameConfig.ts         # 게임 밸런스 상수 (중력, 속도 등)
│   └── qualityPresets.ts     # 품질 프리셋 (Low/Medium/High)
│
├── debug/            # 디버그 도구
│   ├── FPSCounter.ts         # FPS 모니터
│   └── Profiler.ts           # 성능 프로파일러
│
├── types/            # 공통 타입 정의
│   ├── game.ts               # 게임 관련 타입/인터페이스
│   ├── scene.ts              # 씬 관련 타입
│   └── physics.ts            # 물리 관련 타입
│
├── utils/            # 유틸리티 함수
│   ├── math.ts               # 수학 유틸리티 (clamp, lerp 등)
│   ├── random.ts             # 난수 생성 유틸리티
│   └── platform.ts           # 플랫폼 감지 유틸리티
│
├── App.tsx                   # 루트 React 컴포넌트
├── main.tsx                  # 앱 엔트리포인트
└── vite-env.d.ts             # Vite 타입 선언

public/
└── assets/           # 정적 에셋
    ├── images/               # 스프라이트, 배경 이미지
    │   ├── character/
    │   ├── stairs/
    │   └── backgrounds/
    ├── sounds/               # 효과음, BGM
    │   ├── sfx/
    │   └── bgm/
    └── fonts/                # 커스텀 폰트
```

### 3.1 디렉터리별 역할 상세

| 디렉터리 | 역할 | 핵심 파일 |
|---|---|---|
| `scenes/` | 화면 단위의 게임 상태를 관리. 각 씬은 `enter()`, `update()`, `render()`, `exit()` 생명주기를 갖는다. | `GameScene.ts` |
| `engine/` | 게임 엔진의 핵심 모듈. 물리 연산, 렌더링, 루프 관리를 담당한다. | `PhysicsEngine.ts`, `GameLoop.ts` |
| `components/` | React 컴포넌트로 구현되는 UI 레이어. Canvas 위에 오버레이로 표시된다. | `HUD.tsx`, `Canvas.tsx` |
| `stores/` | 전역 상태를 관리하는 Zustand 스토어. 게임 로직과 UI 간의 데이터 브릿지 역할. | `gameStore.ts` |
| `input/` | 터치/마우스/키보드 입력을 통합 인터페이스로 처리한다. | `InputController.ts` |
| `config/` | 하드코딩 없이 게임 밸런스를 조정할 수 있도록 상수를 분리한다. | `gameConfig.ts` |

---

## 4. 개발 환경 설정

### 4.1 ESLint 설정 (React + TypeScript)

Vite의 react-ts 템플릿은 ESLint flat config(`eslint.config.js`)를 기본 생성한다. 아래는 권장 설정이다.

```javascript
// eslint.config.js
import js from '@eslint/js';
import globals from 'globals';
import reactHooks from 'eslint-plugin-react-hooks';
import reactRefresh from 'eslint-plugin-react-refresh';
import tseslint from 'typescript-eslint';

export default tseslint.config(
  { ignores: ['dist'] },
  {
    extends: [js.configs.recommended, ...tseslint.configs.recommended],
    files: ['**/*.{ts,tsx}'],
    languageOptions: {
      ecmaVersion: 2024,
      globals: globals.browser,
    },
    plugins: {
      'react-hooks': reactHooks,
      'react-refresh': reactRefresh,
    },
    rules: {
      ...reactHooks.configs.recommended.rules,
      'react-refresh/only-export-components': [
        'warn',
        { allowConstantExport: true },
      ],
      // 게임 개발 시 유용한 커스텀 규칙
      '@typescript-eslint/no-unused-vars': ['warn', { argsIgnorePattern: '^_' }],
      '@typescript-eslint/explicit-function-return-type': 'off',
      'no-console': ['warn', { allow: ['warn', 'error'] }],
    },
  }
);
```

### 4.2 Prettier 설정

```json
// .prettierrc
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "all",
  "printWidth": 100,
  "bracketSpacing": true,
  "arrowParens": "always",
  "endOfLine": "lf"
}
```

```
// .prettierignore
dist
node_modules
*.md
```

### 4.3 TypeScript 설정 (tsconfig.json)

```json
{
  "compilerOptions": {
    /* 기본 설정 */
    "target": "ES2020",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",

    /* 엄격 모드 */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "forceConsistentCasingInFileNames": true,

    /* 경로 별칭 (Path Alias) */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@engine/*": ["./src/engine/*"],
      "@scenes/*": ["./src/scenes/*"],
      "@components/*": ["./src/components/*"],
      "@stores/*": ["./src/stores/*"],
      "@config/*": ["./src/config/*"],
      "@types/*": ["./src/types/*"],
      "@utils/*": ["./src/utils/*"],
      "@audio/*": ["./src/audio/*"],
      "@input/*": ["./src/input/*"],
      "@assets/*": ["./src/assets/*"],
      "@debug/*": ["./src/debug/*"]
    },

    /* 출력 */
    "skipLibCheck": true,
    "isolatedModules": true,
    "allowImportingTsExtensions": true,
    "moduleDetection": "force",
    "noEmit": true,

    /* 기타 */
    "resolveJsonModule": true,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true
  },
  "include": ["src"],
  "references": [{ "path": "./tsconfig.node.json" }]
}
```

경로 별칭을 사용하면 다음과 같이 깔끔한 import가 가능하다.

```typescript
// 경로 별칭 사용 전
import { PhysicsEngine } from '../../../engine/PhysicsEngine';

// 경로 별칭 사용 후
import { PhysicsEngine } from '@engine/PhysicsEngine';
```

### 4.4 .gitignore

```gitignore
# Dependencies
node_modules/

# Build output
dist/
dist-ssr/

# Environment
.env
.env.local
.env.*.local

# IDE
.vscode/*
!.vscode/extensions.json
.idea/

# OS
.DS_Store
Thumbs.db

# Debug
*.log
npm-debug.log*

# Bun
bun.lockb

# TypeScript
*.tsbuildinfo

# 에셋 원본 (PSD, AI 등 대용량 소스 파일)
assets-raw/
```

---

## 5. 빌드/배포 파이프라인

### 5.1 Vite 설정 (vite.config.ts)

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';
import { resolve } from 'path';

export default defineConfig({
  plugins: [react()],

  // GitHub Pages 배포 시 base path 설정
  // 예: https://username.github.io/infinite-stairs/
  base: '/infinite-stairs/',

  resolve: {
    alias: {
      '@': resolve(__dirname, './src'),
      '@engine': resolve(__dirname, './src/engine'),
      '@scenes': resolve(__dirname, './src/scenes'),
      '@components': resolve(__dirname, './src/components'),
      '@stores': resolve(__dirname, './src/stores'),
      '@config': resolve(__dirname, './src/config'),
      '@types': resolve(__dirname, './src/types'),
      '@utils': resolve(__dirname, './src/utils'),
      '@audio': resolve(__dirname, './src/audio'),
      '@input': resolve(__dirname, './src/input'),
      '@assets': resolve(__dirname, './src/assets'),
      '@debug': resolve(__dirname, './src/debug'),
    },
  },

  build: {
    // 빌드 타겟
    target: 'es2020',

    // 출력 디렉터리
    outDir: 'dist',

    // 소스맵 (프로덕션에서는 false 권장)
    sourcemap: false,

    // 청크 크기 경고 임계값 (KB)
    chunkSizeWarningLimit: 500,

    // 청크 분할 전략
    rollupOptions: {
      output: {
        manualChunks: {
          // React 관련 모듈을 별도 청크로 분리
          'vendor-react': ['react', 'react-dom'],
          // 물리 엔진을 별도 청크로 분리 (용량이 크므로)
          'vendor-physics': ['matter-js'],
          // 상태 관리를 별도 청크로 분리
          'vendor-state': ['zustand'],
        },
      },
    },

    // CSS 코드 분할
    cssCodeSplit: true,

    // 에셋 인라인 임계값 (4KB 이하 파일은 base64로 인라인)
    assetsInlineLimit: 4096,
  },

  // 개발 서버 설정
  server: {
    port: 3000,
    open: true,
    // 모바일 디바이스에서 접근할 수 있도록 호스트 설정
    host: true,
  },

  // 프리뷰 서버 설정 (빌드 결과물 확인)
  preview: {
    port: 4173,
  },
});
```

### 5.2 모바일 최적화 (index.html)

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport"
          content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover" />

    <!-- 모바일 웹앱 메타 태그 -->
    <meta name="mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-capable" content="yes" />
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent" />
    <meta name="apple-mobile-web-app-title" content="무한 계단" />

    <!-- 화면 꺼짐 방지 -->
    <meta name="screen-orientation" content="portrait" />

    <!-- 터치 하이라이트 제거 -->
    <style>
      * {
        -webkit-tap-highlight-color: transparent;
        -webkit-touch-callout: none;
        -webkit-user-select: none;
        user-select: none;
      }
      html, body {
        margin: 0;
        padding: 0;
        overflow: hidden;
        width: 100%;
        height: 100%;
        background: #000;
      }
    </style>

    <title>무한 계단</title>
    <link rel="icon" type="image/svg+xml" href="/vite.svg" />
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

### 5.3 GitHub Pages 배포

GitHub Actions를 사용하여 자동 배포를 구성한다.

```yaml
# .github/workflows/deploy.yml
name: Deploy to GitHub Pages

on:
  push:
    branches: ['main']

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: 'pages'
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - name: Install dependencies
        run: bun install --frozen-lockfile

      - name: Build
        run: bun run build

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 5.4 Vercel 배포 (대안)

Vercel을 사용할 경우 별도의 CI/CD 설정이 필요 없다.

```json
// vercel.json
{
  "buildCommand": "bun run build",
  "installCommand": "bun install",
  "outputDirectory": "dist",
  "framework": "vite"
}
```

```bash
# Vercel CLI 설치 및 배포
bunx vercel --prod
```

---

## 6. 실행 명령어

### 6.1 프로젝트 생성 및 초기 설정

```bash
# 1. Vite 프로젝트 생성
bun create vite infinite-stairs --template react-ts

# 2. 프로젝트 디렉터리 이동
cd infinite-stairs

# 3. 기본 의존성 설치
bun install

# 4. 런타임 의존성 추가
bun add matter-js zustand

# 5. 개발 의존성 추가
bun add -d @types/matter-js eslint prettier

# 6. (선택) Howler.js 오디오 라이브러리
# bun add howler
# bun add -d @types/howler
```

### 6.2 개발 명령어

```bash
# 개발 서버 실행 (HMR 지원, 기본 포트 3000)
bun dev

# 프로덕션 빌드
bun run build

# 빌드 결과물 로컬 프리뷰
bun run preview

# 린트 실행
bunx eslint src/

# 포맷팅 실행
bunx prettier --write src/

# 타입 체크 (빌드 없이)
bunx tsc --noEmit
```

### 6.3 package.json 스크립트 설정

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint src/",
    "lint:fix": "eslint src/ --fix",
    "format": "prettier --write src/",
    "typecheck": "tsc --noEmit"
  }
}
```

---

## 부록: 주요 코드 예시

### A. Zustand 게임 스토어 예시

```typescript
// src/stores/gameStore.ts
import { create } from 'zustand';

interface GameState {
  // 상태
  score: number;
  highScore: number;
  combo: number;
  isPlaying: boolean;
  isPaused: boolean;

  // 액션
  incrementScore: (points: number) => void;
  incrementCombo: () => void;
  resetCombo: () => void;
  startGame: () => void;
  pauseGame: () => void;
  resumeGame: () => void;
  endGame: () => void;
  reset: () => void;
}

export const useGameStore = create<GameState>((set, get) => ({
  score: 0,
  highScore: 0,
  combo: 0,
  isPlaying: false,
  isPaused: false,

  incrementScore: (points) =>
    set((state) => ({
      score: state.score + points * (1 + state.combo * 0.1),
    })),

  incrementCombo: () =>
    set((state) => ({ combo: state.combo + 1 })),

  resetCombo: () => set({ combo: 0 }),

  startGame: () =>
    set({ isPlaying: true, isPaused: false, score: 0, combo: 0 }),

  pauseGame: () => set({ isPaused: true }),

  resumeGame: () => set({ isPaused: false }),

  endGame: () =>
    set((state) => ({
      isPlaying: false,
      highScore: Math.max(state.highScore, state.score),
    })),

  reset: () =>
    set({ score: 0, combo: 0, isPlaying: false, isPaused: false }),
}));
```

### B. 게임 루프 예시

```typescript
// src/engine/GameLoop.ts
type UpdateCallback = (deltaTime: number) => void;
type RenderCallback = () => void;

export class GameLoop {
  private lastTime = 0;
  private accumulator = 0;
  private readonly fixedTimeStep = 1000 / 60; // 60 FPS 고정 물리 스텝
  private animationFrameId: number | null = null;
  private isRunning = false;

  constructor(
    private onUpdate: UpdateCallback,
    private onRender: RenderCallback,
  ) {}

  start(): void {
    if (this.isRunning) return;
    this.isRunning = true;
    this.lastTime = performance.now();
    this.loop(this.lastTime);
  }

  stop(): void {
    this.isRunning = false;
    if (this.animationFrameId !== null) {
      cancelAnimationFrame(this.animationFrameId);
      this.animationFrameId = null;
    }
  }

  private loop = (currentTime: number): void => {
    if (!this.isRunning) return;

    const deltaTime = currentTime - this.lastTime;
    this.lastTime = currentTime;

    // 프레임 시간이 너무 길면 제한 (탭 비활성화 등)
    this.accumulator += Math.min(deltaTime, 200);

    // 고정 시간 간격으로 물리 업데이트
    while (this.accumulator >= this.fixedTimeStep) {
      this.onUpdate(this.fixedTimeStep);
      this.accumulator -= this.fixedTimeStep;
    }

    // 렌더링은 매 프레임 수행
    this.onRender();

    this.animationFrameId = requestAnimationFrame(this.loop);
  };
}
```

### C. 물리 엔진 래핑 예시

```typescript
// src/engine/PhysicsEngine.ts
import Matter from 'matter-js';

export class PhysicsEngine {
  public engine: Matter.Engine;
  public world: Matter.World;

  constructor() {
    this.engine = Matter.Engine.create({
      gravity: { x: 0, y: 1.5, scale: 0.001 },
    });
    this.world = this.engine.world;
  }

  update(delta: number): void {
    Matter.Engine.update(this.engine, delta);
  }

  addBody(body: Matter.Body): void {
    Matter.Composite.add(this.world, body);
  }

  removeBody(body: Matter.Body): void {
    Matter.Composite.remove(this.world, body);
  }

  createStair(x: number, y: number, width: number, height: number): Matter.Body {
    return Matter.Bodies.rectangle(x, y, width, height, {
      isStatic: true,
      friction: 0.8,
      restitution: 0.2,
      label: 'stair',
    });
  }

  createCharacter(x: number, y: number): Matter.Body {
    return Matter.Bodies.rectangle(x, y, 30, 50, {
      friction: 0.5,
      restitution: 0.1,
      density: 0.002,
      label: 'character',
    });
  }

  onCollision(callback: (pairs: Matter.IPair[]) => void): void {
    Matter.Events.on(this.engine, 'collisionStart', (event) => {
      callback(event.pairs);
    });
  }

  destroy(): void {
    Matter.Engine.clear(this.engine);
    Matter.World.clear(this.world, false);
  }
}
```

### D. Canvas 컴포넌트 예시

```tsx
// src/components/Canvas.tsx
import { useRef, useEffect } from 'react';

interface CanvasProps {
  onInit: (ctx: CanvasRenderingContext2D, canvas: HTMLCanvasElement) => void;
  onCleanup?: () => void;
}

export function Canvas({ onInit, onCleanup }: CanvasProps) {
  const canvasRef = useRef<HTMLCanvasElement>(null);

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    // 캔버스 크기를 화면에 맞춤
    const resize = () => {
      const dpr = window.devicePixelRatio || 1;
      canvas.width = window.innerWidth * dpr;
      canvas.height = window.innerHeight * dpr;
      canvas.style.width = `${window.innerWidth}px`;
      canvas.style.height = `${window.innerHeight}px`;

      const ctx = canvas.getContext('2d');
      if (ctx) {
        ctx.scale(dpr, dpr);
      }
    };

    resize();
    window.addEventListener('resize', resize);

    const ctx = canvas.getContext('2d');
    if (ctx) {
      onInit(ctx, canvas);
    }

    return () => {
      window.removeEventListener('resize', resize);
      onCleanup?.();
    };
  }, [onInit, onCleanup]);

  return (
    <canvas
      ref={canvasRef}
      style={{
        display: 'block',
        touchAction: 'none', // 터치 스크롤 방지
      }}
    />
  );
}
```

### E. 게임 설정 상수 예시

```typescript
// src/config/gameConfig.ts
export const GAME_CONFIG = {
  // 물리
  GRAVITY: 1.5,
  CHARACTER_JUMP_FORCE: -0.08,
  CHARACTER_MOVE_SPEED: 3,

  // 계단
  STAIR_WIDTH: 80,
  STAIR_HEIGHT: 15,
  STAIR_GAP_MIN: 50,
  STAIR_GAP_MAX: 90,
  STAIR_OFFSET_MAX: 60,

  // 게임플레이
  INITIAL_SPEED: 1.0,
  SPEED_INCREMENT: 0.05,
  MAX_SPEED: 3.0,
  COMBO_TIMEOUT_MS: 2000,

  // 점수
  SCORE_PER_STAIR: 10,
  COMBO_MULTIPLIER: 0.1,

  // 카메라
  CAMERA_FOLLOW_SPEED: 0.1,
  CAMERA_LOOK_AHEAD: 100,

  // 화면
  DESIGN_WIDTH: 390,
  DESIGN_HEIGHT: 844,
} as const;
```

```typescript
// src/config/qualityPresets.ts
export interface QualityPreset {
  name: string;
  particleCount: number;
  shadowEnabled: boolean;
  backgroundDetail: 'low' | 'medium' | 'high';
  antiAlias: boolean;
}

export const QUALITY_PRESETS: Record<string, QualityPreset> = {
  low: {
    name: 'Low',
    particleCount: 10,
    shadowEnabled: false,
    backgroundDetail: 'low',
    antiAlias: false,
  },
  medium: {
    name: 'Medium',
    particleCount: 30,
    shadowEnabled: true,
    backgroundDetail: 'medium',
    antiAlias: false,
  },
  high: {
    name: 'High',
    particleCount: 60,
    shadowEnabled: true,
    backgroundDetail: 'high',
    antiAlias: true,
  },
};
```

---

## 다음 단계

이 계획서를 기반으로 다음 순서대로 구현을 진행한다.

1. **프로젝트 생성 및 의존성 설치** - 위 명령어 실행
2. **엔진 코어 구현** - `GameLoop`, `Renderer`, `PhysicsEngine`
3. **씬 시스템 구현** - `SceneManager` 및 기본 씬
4. **입력 시스템 구현** - `InputController` (터치/마우스 통합)
5. **게임 오브젝트 구현** - 캐릭터, 계단 생성 및 물리 적용
6. **UI 레이어 구현** - HUD, 메뉴, 결과 화면
7. **오디오 시스템 구현** - 효과음 및 BGM
8. **최적화 및 폴리싱** - 성능 튜닝, 시각 효과 추가
9. **배포** - GitHub Pages 또는 Vercel에 배포
