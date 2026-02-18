# 반응형 및 성능 최적화 구현 계획서

> **프로젝트**: Infinite Stairs (Vibe Coding)
> **작성일**: 2026-02-18
> **기반**: PRD Task 6 – 반응형 및 성능 최적화

---

## 목차

1. [반응형 레이아웃 설계](#1-반응형-레이아웃-설계)
2. [입력 시스템 통합 설계](#2-입력-시스템-통합-설계)
3. [성능 최적화 체크리스트](#3-성능-최적화-체크리스트)
4. [저사양 모드 구현 방안](#4-저사양-모드-구현-방안)
5. [모바일 특화 최적화](#5-모바일-특화-최적화)
6. [성능 측정 및 모니터링 도구](#6-성능-측정-및-모니터링-도구)
7. [구현 코드 예시 종합](#7-구현-코드-예시-종합)

---

## 1. 반응형 레이아웃 설계

### 1.1 Canvas 가변 크기 (ResizeObserver)

Canvas는 부모 컨테이너의 크기를 추적하여 자동으로 리사이즈된다. `window.resize` 이벤트 대신 `ResizeObserver`를 사용하여 정확한 컨테이너 크기를 반영한다.

**설계 원칙:**

- Canvas의 논리적 해상도와 CSS 크기를 분리한다
- `devicePixelRatio`를 반영하여 선명한 렌더링을 보장한다
- 리사이즈 시 디바운스를 적용하여 과도한 재계산을 방지한다

**구현 코드:**

```typescript
// hooks/useCanvasResize.ts
import { useEffect, useRef } from "react";

interface CanvasSize {
  width: number;
  height: number;
  scale: number;
}

export function useCanvasResize(
  containerRef: React.RefObject<HTMLDivElement>,
  canvasRef: React.RefObject<HTMLCanvasElement>,
  onResize: (size: CanvasSize) => void
) {
  const sizeRef = useRef<CanvasSize>({ width: 0, height: 0, scale: 1 });

  useEffect(() => {
    const container = containerRef.current;
    const canvas = canvasRef.current;
    if (!container || !canvas) return;

    const observer = new ResizeObserver((entries) => {
      const entry = entries[0];
      if (!entry) return;

      const { width, height } = entry.contentRect;
      const dpr = Math.min(window.devicePixelRatio || 1, 2); // 최대 2x로 제한

      // Canvas 논리적 해상도 설정
      canvas.width = Math.round(width * dpr);
      canvas.height = Math.round(height * dpr);

      // CSS 크기 설정
      canvas.style.width = `${width}px`;
      canvas.style.height = `${height}px`;

      const newSize: CanvasSize = { width, height, scale: dpr };
      sizeRef.current = newSize;
      onResize(newSize);
    });

    observer.observe(container);
    return () => observer.disconnect();
  }, [containerRef, canvasRef, onResize]);

  return sizeRef;
}
```

### 1.2 Safe-area 대응 (iOS Notch)

iOS 디바이스의 노치 및 홈 인디케이터 영역을 고려하여 UI 요소가 가려지지 않도록 한다.

**CSS 설정:**

```css
/* styles/safe-area.css */
:root {
  --sai-top: env(safe-area-inset-top, 0px);
  --sai-right: env(safe-area-inset-right, 0px);
  --sai-bottom: env(safe-area-inset-bottom, 0px);
  --sai-left: env(safe-area-inset-left, 0px);
}

/* 게임 컨테이너 */
.game-container {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  overflow: hidden;
}

/* Canvas는 전체 영역을 채움 */
.game-canvas {
  display: block;
  width: 100%;
  height: 100%;
  touch-action: none;
}

/* HUD/UI 오버레이는 safe-area 내부에 배치 */
.game-hud {
  position: absolute;
  top: var(--sai-top);
  right: var(--sai-right);
  bottom: var(--sai-bottom);
  left: var(--sai-left);
  pointer-events: none;
}

.game-hud > * {
  pointer-events: auto;
}
```

**HTML meta 태그:**

```html
<meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover" />
```

### 1.3 화면 방향 전환 (Portrait / Landscape)

모바일에서는 Portrait, 데스크탑에서는 Landscape를 기본으로 하되, 방향 전환 시 스케일만 조정하여 게임 로직에 영향을 주지 않는다.

**설계 원칙:**

- 게임 월드 좌표계는 고정 (예: 기준 너비 400px 기반)
- 화면 크기가 변해도 월드 좌표 → 화면 좌표 변환 스케일만 변경
- 세로 모드에서는 좌우 여백, 가로 모드에서는 상하 여백으로 letterbox 처리

**구현 코드:**

```typescript
// core/camera.ts
interface CameraConfig {
  worldWidth: number;   // 기준 월드 너비 (예: 400)
  worldHeight: number;  // 기준 월드 높이 (예: 700)
}

interface CameraTransform {
  scale: number;
  offsetX: number;
  offsetY: number;
}

export function calculateCameraTransform(
  screenWidth: number,
  screenHeight: number,
  config: CameraConfig
): CameraTransform {
  const scaleX = screenWidth / config.worldWidth;
  const scaleY = screenHeight / config.worldHeight;

  // 화면에 맞추되 비율 유지 (contain 방식)
  const scale = Math.min(scaleX, scaleY);

  // 중앙 정렬 오프셋
  const offsetX = (screenWidth - config.worldWidth * scale) / 2;
  const offsetY = (screenHeight - config.worldHeight * scale) / 2;

  return { scale, offsetX, offsetY };
}

// 화면 방향 감지
export function getOrientation(): "portrait" | "landscape" {
  if (typeof screen !== "undefined" && screen.orientation) {
    return screen.orientation.type.includes("portrait") ? "portrait" : "landscape";
  }
  return window.innerHeight > window.innerWidth ? "portrait" : "landscape";
}
```

---

## 2. 입력 시스템 통합 설계

### 2.1 Pointer Events 통합

마우스, 터치, 펜 입력을 `Pointer Events` API로 통합 처리한다. 별도의 `touchstart`/`mousedown` 분기 없이 단일 코드 경로를 유지한다.

**구현 코드:**

```typescript
// input/pointer-input.ts
export interface PointerState {
  isDown: boolean;
  x: number;
  y: number;
  pointerId: number;
  pointerType: string; // "mouse" | "touch" | "pen"
}

export class PointerInputManager {
  private pointers = new Map<number, PointerState>();
  private canvas: HTMLCanvasElement;
  private worldScale: number = 1;
  private worldOffsetX: number = 0;
  private worldOffsetY: number = 0;

  constructor(canvas: HTMLCanvasElement) {
    this.canvas = canvas;
    this.bind();
  }

  private bind() {
    const c = this.canvas;

    // CSS에서 touch-action: none 설정 필수
    c.style.touchAction = "none";

    c.addEventListener("pointerdown", this.onPointerDown);
    c.addEventListener("pointermove", this.onPointerMove);
    c.addEventListener("pointerup", this.onPointerUp);
    c.addEventListener("pointercancel", this.onPointerUp);

    // 컨텍스트 메뉴 방지 (길게 누르기 시)
    c.addEventListener("contextmenu", (e) => e.preventDefault());
  }

  private toWorldCoords(clientX: number, clientY: number) {
    const rect = this.canvas.getBoundingClientRect();
    const screenX = clientX - rect.left;
    const screenY = clientY - rect.top;
    return {
      x: (screenX - this.worldOffsetX) / this.worldScale,
      y: (screenY - this.worldOffsetY) / this.worldScale,
    };
  }

  private onPointerDown = (e: PointerEvent) => {
    // 터치 시 해당 포인터 캡처
    this.canvas.setPointerCapture(e.pointerId);

    const { x, y } = this.toWorldCoords(e.clientX, e.clientY);
    this.pointers.set(e.pointerId, {
      isDown: true,
      x,
      y,
      pointerId: e.pointerId,
      pointerType: e.pointerType,
    });
  };

  private onPointerMove = (e: PointerEvent) => {
    const pointer = this.pointers.get(e.pointerId);
    if (!pointer) return;

    const { x, y } = this.toWorldCoords(e.clientX, e.clientY);
    pointer.x = x;
    pointer.y = y;
  };

  private onPointerUp = (e: PointerEvent) => {
    this.pointers.delete(e.pointerId);
  };

  updateTransform(scale: number, offsetX: number, offsetY: number) {
    this.worldScale = scale;
    this.worldOffsetX = offsetX;
    this.worldOffsetY = offsetY;
  }

  getActivePointers(): PointerState[] {
    return Array.from(this.pointers.values());
  }

  getPrimaryPointer(): PointerState | null {
    const pointers = this.getActivePointers();
    return pointers.length > 0 ? pointers[0] : null;
  }

  destroy() {
    const c = this.canvas;
    c.removeEventListener("pointerdown", this.onPointerDown);
    c.removeEventListener("pointermove", this.onPointerMove);
    c.removeEventListener("pointerup", this.onPointerUp);
    c.removeEventListener("pointercancel", this.onPointerUp);
  }
}
```

### 2.2 터치 최적화 및 제스처

모바일 환경에서의 터치 경험을 최적화한다.

**엄지 영역 버튼 배치:**

```typescript
// ui/touch-layout.ts
interface TouchZone {
  id: string;
  label: string;
  x: number;      // 월드 좌표 비율 (0~1)
  y: number;
  width: number;
  height: number;
  anchor: "bottom-left" | "bottom-right" | "bottom-center";
}

// 엄지 도달 가능 영역 기반 버튼 배치
export const MOBILE_TOUCH_ZONES: TouchZone[] = [
  {
    id: "left-step",
    label: "왼발",
    x: 0.05,
    y: 0.7,
    width: 0.4,
    height: 0.3,
    anchor: "bottom-left",
  },
  {
    id: "right-step",
    label: "오른발",
    x: 0.55,
    y: 0.7,
    width: 0.4,
    height: 0.3,
    anchor: "bottom-right",
  },
];

// 최소 터치 타겟 크기: 44x44px (Apple HIG 기준)
export const MIN_TOUCH_TARGET_PX = 44;
```

**드래그 핸들 (큰 인터랙션 영역):**

```typescript
// input/drag-handler.ts
export interface DragState {
  isDragging: boolean;
  startX: number;
  startY: number;
  currentX: number;
  currentY: number;
  deltaX: number;
  deltaY: number;
}

export function createDragHandler(
  onDragStart?: (state: DragState) => void,
  onDragMove?: (state: DragState) => void,
  onDragEnd?: (state: DragState) => void
) {
  const state: DragState = {
    isDragging: false,
    startX: 0, startY: 0,
    currentX: 0, currentY: 0,
    deltaX: 0, deltaY: 0,
  };

  const DRAG_THRESHOLD = 8; // 드래그 인식 최소 거리 (px)

  return {
    onPointerDown(x: number, y: number) {
      state.startX = x;
      state.startY = y;
      state.currentX = x;
      state.currentY = y;
      state.deltaX = 0;
      state.deltaY = 0;
    },

    onPointerMove(x: number, y: number) {
      const dx = x - state.startX;
      const dy = y - state.startY;

      if (!state.isDragging) {
        const dist = Math.sqrt(dx * dx + dy * dy);
        if (dist >= DRAG_THRESHOLD) {
          state.isDragging = true;
          onDragStart?.(state);
        }
        return;
      }

      state.currentX = x;
      state.currentY = y;
      state.deltaX = dx;
      state.deltaY = dy;
      onDragMove?.(state);
    },

    onPointerUp() {
      if (state.isDragging) {
        onDragEnd?.(state);
      }
      state.isDragging = false;
    },

    getState: () => state,
  };
}
```

---

## 3. 성능 최적화 체크리스트

### 3.1 물리 엔진 최적화

| 항목 | 설명 | 우선순위 |
|------|------|----------|
| Fixed Timestep | 60Hz 고정 스텝 + accumulator 패턴 적용 | **필수** |
| Sleeping 활용 | 정지 상태 바디의 자동 sleep 활성화 | **필수** |
| 바디 수 상한 | 최대 바디 수 제한 (기본 200, 저사양 100) | **필수** |
| 컨스트레인트 상한 | 최대 조인트/컨스트레인트 수 제한 | 높음 |
| Off-screen 제거 | 화면 밖 오브젝트 자동 제거 (풀링) | 높음 |
| Broad-phase 튜닝 | AABB 트리 최적화 확인 | 중간 |

**Fixed Timestep 구현:**

```typescript
// core/game-loop.ts
const FIXED_DT = 1 / 60; // 60Hz 고정 타임스텝
const MAX_SUBSTEPS = 3;   // 최대 서브스텝 (프레임 드랍 방지)

export class GameLoop {
  private accumulator = 0;
  private lastTime = 0;
  private rafId = 0;
  private running = false;

  constructor(
    private update: (dt: number) => void,
    private render: (interpolation: number) => void
  ) {}

  start() {
    this.running = true;
    this.lastTime = performance.now();
    this.tick(this.lastTime);
  }

  stop() {
    this.running = false;
    cancelAnimationFrame(this.rafId);
  }

  private tick = (now: number) => {
    if (!this.running) return;
    this.rafId = requestAnimationFrame(this.tick);

    const frameTime = Math.min((now - this.lastTime) / 1000, FIXED_DT * MAX_SUBSTEPS);
    this.lastTime = now;
    this.accumulator += frameTime;

    // 고정 타임스텝으로 물리 업데이트
    let steps = 0;
    while (this.accumulator >= FIXED_DT && steps < MAX_SUBSTEPS) {
      this.update(FIXED_DT);
      this.accumulator -= FIXED_DT;
      steps++;
    }

    // 보간값으로 부드러운 렌더링
    const interpolation = this.accumulator / FIXED_DT;
    this.render(interpolation);
  };
}
```

### 3.2 렌더링 최적화

| 항목 | 설명 | 우선순위 |
|------|------|----------|
| React 렌더 최소화 | 게임 상태를 `ref`로 관리, `setState` 최소화 | **필수** |
| Canvas 직접 조작 | React DOM 우회, Canvas 2D context 직접 사용 | **필수** |
| 더티 영역 렌더링 | 변경된 영역만 다시 그리기 | 중간 |
| 스프라이트 아틀라스 | 이미지 drawcall 최소화 | 중간 |
| Off-screen canvas | 정적 배경 사전 렌더링 | 낮음 |

**React 렌더 최소화 패턴:**

```typescript
// components/GameCanvas.tsx
import { useRef, useEffect, useCallback } from "react";

export function GameCanvas() {
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const gameStateRef = useRef({
    score: 0,
    stairCount: 0,
    // 모든 게임 상태를 ref로 관리 (setState 금지)
  });
  const gameLoopRef = useRef<GameLoop | null>(null);

  // HUD 업데이트가 필요한 경우에만 제한적으로 setState 사용
  // 예: 게임 오버, 일시정지 등 화면 전환

  useEffect(() => {
    const canvas = canvasRef.current;
    if (!canvas) return;

    const ctx = canvas.getContext("2d")!;

    const update = (dt: number) => {
      // 물리 + 게임 로직 (ref 기반, 리렌더 없음)
    };

    const render = (interpolation: number) => {
      // Canvas 직접 렌더링
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      // ... 게임 오브젝트 렌더링
    };

    const loop = new GameLoop(update, render);
    gameLoopRef.current = loop;
    loop.start();

    return () => loop.stop();
  }, []);

  return (
    <canvas
      ref={canvasRef}
      className="game-canvas"
      style={{ touchAction: "none" }}
    />
  );
}
```

### 3.3 메모리 최적화

| 항목 | 설명 | 우선순위 |
|------|------|----------|
| 오브젝트 풀링 | 계단, 파티클 등 재사용 풀 | **필수** |
| GC 압박 최소화 | 루프 내 객체 생성 금지, 사전 할당 | **필수** |
| 텍스처 메모리 관리 | 미사용 이미지 해제 | 높음 |
| WASM 메모리 관리 | 선형 메모리 사이즈 적정 설정 | 높음 |

**오브젝트 풀 구현:**

```typescript
// core/object-pool.ts
export class ObjectPool<T> {
  private pool: T[] = [];
  private activeCount = 0;

  constructor(
    private factory: () => T,
    private reset: (obj: T) => void,
    initialSize: number = 32
  ) {
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(this.factory());
    }
  }

  acquire(): T {
    let obj: T;
    if (this.activeCount < this.pool.length) {
      obj = this.pool[this.activeCount];
    } else {
      obj = this.factory();
      this.pool.push(obj);
    }
    this.activeCount++;
    return obj;
  }

  release(obj: T) {
    this.reset(obj);
    // 활성 목록 끝과 교환
    const idx = this.pool.indexOf(obj);
    if (idx !== -1 && idx < this.activeCount) {
      this.activeCount--;
      const temp = this.pool[this.activeCount];
      this.pool[this.activeCount] = obj;
      this.pool[idx] = temp;
    }
  }

  getActiveItems(): T[] {
    return this.pool.slice(0, this.activeCount);
  }

  get active(): number {
    return this.activeCount;
  }
}
```

---

## 4. 저사양 모드 구현 방안

### 4.1 품질 옵션 단계

디바이스 성능에 따라 3단계 품질 옵션을 제공한다.

```typescript
// config/quality-settings.ts
export interface QualitySettings {
  label: string;
  maxBodies: number;
  maxConstraints: number;
  particlesEnabled: boolean;
  maxParticles: number;
  shadowsEnabled: boolean;
  dprLimit: number;            // devicePixelRatio 상한
  physicsSubsteps: number;
  backgroundDetail: "full" | "simple" | "none";
}

export const QUALITY_PRESETS: Record<string, QualitySettings> = {
  high: {
    label: "고품질",
    maxBodies: 200,
    maxConstraints: 300,
    particlesEnabled: true,
    maxParticles: 100,
    shadowsEnabled: true,
    dprLimit: 2,
    physicsSubsteps: 3,
    backgroundDetail: "full",
  },
  medium: {
    label: "중간",
    maxBodies: 150,
    maxConstraints: 200,
    particlesEnabled: true,
    maxParticles: 50,
    shadowsEnabled: false,
    dprLimit: 1.5,
    physicsSubsteps: 2,
    backgroundDetail: "simple",
  },
  low: {
    label: "저사양",
    maxBodies: 100,
    maxConstraints: 100,
    particlesEnabled: false,
    maxParticles: 0,
    shadowsEnabled: false,
    dprLimit: 1,
    physicsSubsteps: 1,
    backgroundDetail: "none",
  },
};
```

### 4.2 자동 품질 감지

FPS 드랍을 감지하여 자동으로 품질을 낮추는 어댑티브 시스템을 구현한다.

```typescript
// core/adaptive-quality.ts
export class AdaptiveQualityManager {
  private fpsSamples: number[] = [];
  private sampleWindow = 60; // 60프레임 윈도우
  private currentQuality: "high" | "medium" | "low" = "high";
  private downgradeThreshold = 45; // 평균 FPS가 이 이하이면 품질 다운
  private upgradeThreshold = 58;   // 평균 FPS가 이 이상이면 품질 업
  private cooldownFrames = 180;    // 품질 변경 후 쿨다운 (3초)
  private framesSinceChange = 0;

  onQualityChange?: (quality: string) => void;

  recordFrame(fps: number) {
    this.fpsSamples.push(fps);
    if (this.fpsSamples.length > this.sampleWindow) {
      this.fpsSamples.shift();
    }
    this.framesSinceChange++;

    if (this.fpsSamples.length < this.sampleWindow) return;
    if (this.framesSinceChange < this.cooldownFrames) return;

    const avgFps = this.fpsSamples.reduce((a, b) => a + b, 0) / this.fpsSamples.length;

    if (avgFps < this.downgradeThreshold) {
      this.downgrade();
    } else if (avgFps > this.upgradeThreshold) {
      this.upgrade();
    }
  }

  private downgrade() {
    const levels = ["high", "medium", "low"] as const;
    const idx = levels.indexOf(this.currentQuality);
    if (idx < levels.length - 1) {
      this.currentQuality = levels[idx + 1];
      this.framesSinceChange = 0;
      this.fpsSamples = [];
      this.onQualityChange?.(this.currentQuality);
    }
  }

  private upgrade() {
    const levels = ["high", "medium", "low"] as const;
    const idx = levels.indexOf(this.currentQuality);
    if (idx > 0) {
      this.currentQuality = levels[idx - 1];
      this.framesSinceChange = 0;
      this.fpsSamples = [];
      this.onQualityChange?.(this.currentQuality);
    }
  }

  getQuality() {
    return this.currentQuality;
  }
}
```

### 4.3 바디 수 제한 및 자동 정리

```typescript
// physics/body-manager.ts
export class BodyManager {
  private bodies: Set<unknown> = new Set();
  private maxBodies: number;

  constructor(qualitySettings: QualitySettings) {
    this.maxBodies = qualitySettings.maxBodies;
  }

  canAddBody(): boolean {
    return this.bodies.size < this.maxBodies;
  }

  /**
   * 화면 밖으로 벗어난 오브젝트를 자동 제거한다.
   * 카메라 뷰포트 기준으로 아래쪽 margin 밖의 바디를 정리한다.
   */
  cleanupOffScreen(cameraY: number, viewHeight: number, margin: number = 200) {
    const threshold = cameraY + viewHeight + margin;
    for (const body of this.bodies) {
      const b = body as { position: { y: number }; id: number };
      if (b.position.y > threshold) {
        // 물리 월드에서 제거 후 풀로 반환
        this.bodies.delete(body);
      }
    }
  }

  updateMaxBodies(max: number) {
    this.maxBodies = max;
  }
}
```

---

## 5. 모바일 특화 최적화

### 5.1 오디오 정책 대응

모바일 브라우저는 사용자 인터랙션 없이 오디오 재생을 차단한다. 첫 터치 시 AudioContext를 resume하는 패턴을 적용한다.

```typescript
// audio/audio-manager.ts
export class AudioManager {
  private ctx: AudioContext | null = null;
  private resumed = false;

  async init() {
    this.ctx = new AudioContext();

    // 사용자 인터랙션 시 resume
    const resumeAudio = async () => {
      if (this.ctx && this.ctx.state === "suspended") {
        await this.ctx.resume();
        this.resumed = true;
      }
      // 한 번만 실행
      document.removeEventListener("pointerdown", resumeAudio);
      document.removeEventListener("keydown", resumeAudio);
    };

    document.addEventListener("pointerdown", resumeAudio, { once: true });
    document.addEventListener("keydown", resumeAudio, { once: true });
  }

  // 배경음 볼륨을 visibility에 따라 조절
  setupVisibilityHandler() {
    document.addEventListener("visibilitychange", () => {
      if (!this.ctx) return;
      if (document.hidden) {
        this.ctx.suspend();
      } else {
        this.ctx.resume();
      }
    });
  }

  isReady(): boolean {
    return this.resumed && this.ctx?.state === "running";
  }
}
```

### 5.2 배터리 및 전력 최적화

```typescript
// core/power-manager.ts
export class PowerManager {
  private isLowPower = false;
  private isBackgrounded = false;
  private onLowPower?: () => void;

  init(onLowPower: () => void) {
    this.onLowPower = onLowPower;

    // 페이지 가시성 변화 감지
    document.addEventListener("visibilitychange", () => {
      this.isBackgrounded = document.hidden;
      if (document.hidden) {
        // 백그라운드에서는 게임 루프 일시정지
        // (배터리 절약 + 리소스 반환)
      }
    });

    // Battery API (지원 브라우저)
    if ("getBattery" in navigator) {
      (navigator as any).getBattery().then((battery: any) => {
        const checkBattery = () => {
          if (battery.level < 0.15 && !battery.charging) {
            this.isLowPower = true;
            this.onLowPower?.();
          }
        };
        battery.addEventListener("levelchange", checkBattery);
        battery.addEventListener("chargingchange", checkBattery);
        checkBattery();
      });
    }
  }

  shouldThrottle(): boolean {
    return this.isBackgrounded || this.isLowPower;
  }
}
```

### 5.3 터치 영역 최적화

```typescript
// ui/touch-feedback.ts

/**
 * 터치 피드백을 위한 시각적 효과
 * 진동 API를 활용한 햅틱 피드백 포함
 */
export function triggerHapticFeedback(type: "light" | "medium" | "heavy") {
  if (!("vibrate" in navigator)) return;

  const patterns: Record<string, number[]> = {
    light: [10],
    medium: [20],
    heavy: [30, 10, 30],
  };

  navigator.vibrate(patterns[type]);
}

/**
 * 터치 영역을 시각적으로 하이라이트 (디버그용)
 */
export function debugDrawTouchZones(
  ctx: CanvasRenderingContext2D,
  zones: Array<{ x: number; y: number; width: number; height: number }>,
  canvasWidth: number,
  canvasHeight: number
) {
  ctx.save();
  ctx.globalAlpha = 0.2;
  ctx.fillStyle = "#00ff00";

  for (const zone of zones) {
    ctx.fillRect(
      zone.x * canvasWidth,
      zone.y * canvasHeight,
      zone.width * canvasWidth,
      zone.height * canvasHeight
    );
  }

  ctx.restore();
}
```

---

## 6. 성능 측정 및 모니터링 도구

### 6.1 FPS 카운터

```typescript
// debug/fps-counter.ts
export class FPSCounter {
  private frames = 0;
  private lastTime = performance.now();
  private currentFPS = 0;
  private fpsHistory: number[] = [];
  private historySize = 300; // 5초 히스토리 (60fps 기준)

  tick(): number {
    this.frames++;
    const now = performance.now();
    const elapsed = now - this.lastTime;

    if (elapsed >= 1000) {
      this.currentFPS = Math.round((this.frames * 1000) / elapsed);
      this.fpsHistory.push(this.currentFPS);
      if (this.fpsHistory.length > this.historySize) {
        this.fpsHistory.shift();
      }
      this.frames = 0;
      this.lastTime = now;
    }

    return this.currentFPS;
  }

  getFPS(): number {
    return this.currentFPS;
  }

  getAverageFPS(): number {
    if (this.fpsHistory.length === 0) return 0;
    return Math.round(
      this.fpsHistory.reduce((a, b) => a + b, 0) / this.fpsHistory.length
    );
  }

  getMinFPS(): number {
    if (this.fpsHistory.length === 0) return 0;
    return Math.min(...this.fpsHistory);
  }

  /**
   * Canvas 위에 FPS 오버레이를 렌더링한다.
   */
  renderOverlay(ctx: CanvasRenderingContext2D, x = 10, y = 20) {
    const fps = this.currentFPS;
    const color = fps >= 55 ? "#00ff00" : fps >= 30 ? "#ffff00" : "#ff0000";

    ctx.save();
    ctx.font = "14px monospace";
    ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
    ctx.fillRect(x - 4, y - 14, 120, 52);

    ctx.fillStyle = color;
    ctx.fillText(`FPS: ${fps}`, x, y);
    ctx.fillStyle = "#ffffff";
    ctx.fillText(`AVG: ${this.getAverageFPS()}`, x, y + 16);
    ctx.fillText(`MIN: ${this.getMinFPS()}`, x, y + 32);
    ctx.restore();
  }
}
```

### 6.2 성능 프로파일러

```typescript
// debug/profiler.ts
interface ProfileEntry {
  name: string;
  totalMs: number;
  callCount: number;
  maxMs: number;
}

export class GameProfiler {
  private entries = new Map<string, ProfileEntry>();
  private startTimes = new Map<string, number>();
  private enabled = false;

  setEnabled(enabled: boolean) {
    this.enabled = enabled;
  }

  begin(name: string) {
    if (!this.enabled) return;
    this.startTimes.set(name, performance.now());
  }

  end(name: string) {
    if (!this.enabled) return;
    const start = this.startTimes.get(name);
    if (start === undefined) return;

    const elapsed = performance.now() - start;
    let entry = this.entries.get(name);
    if (!entry) {
      entry = { name, totalMs: 0, callCount: 0, maxMs: 0 };
      this.entries.set(name, entry);
    }

    entry.totalMs += elapsed;
    entry.callCount++;
    entry.maxMs = Math.max(entry.maxMs, elapsed);
  }

  /**
   * 프레임 종료 시 호출. 통계를 콘솔에 출력한다.
   */
  reportFrame() {
    if (!this.enabled) return;

    const report: Record<string, string> = {};
    for (const [name, entry] of this.entries) {
      const avg = entry.callCount > 0 ? entry.totalMs / entry.callCount : 0;
      report[name] = `avg=${avg.toFixed(2)}ms max=${entry.maxMs.toFixed(2)}ms calls=${entry.callCount}`;
    }

    return report;
  }

  reset() {
    this.entries.clear();
  }

  /**
   * Canvas에 프로파일링 결과를 오버레이로 렌더링한다.
   */
  renderOverlay(ctx: CanvasRenderingContext2D, x = 10, y = 80) {
    if (!this.enabled) return;

    ctx.save();
    ctx.font = "12px monospace";
    ctx.fillStyle = "rgba(0, 0, 0, 0.6)";

    const entries = Array.from(this.entries.values());
    const height = entries.length * 16 + 8;
    ctx.fillRect(x - 4, y - 12, 260, height);

    ctx.fillStyle = "#ffffff";
    let offsetY = y;
    for (const entry of entries) {
      const avg = entry.callCount > 0 ? entry.totalMs / entry.callCount : 0;
      const text = `${entry.name}: ${avg.toFixed(1)}ms (max ${entry.maxMs.toFixed(1)}ms)`;
      ctx.fillText(text, x, offsetY);
      offsetY += 16;
    }

    ctx.restore();
  }
}
```

### 6.3 메모리 모니터

```typescript
// debug/memory-monitor.ts
export class MemoryMonitor {
  /**
   * Chrome 전용 performance.memory API를 활용한 메모리 모니터링.
   * 지원하지 않는 브라우저에서는 null을 반환한다.
   */
  static getMemoryInfo(): {
    usedJSHeapSize: number;
    totalJSHeapSize: number;
    jsHeapSizeLimit: number;
  } | null {
    const perf = performance as any;
    if (perf.memory) {
      return {
        usedJSHeapSize: perf.memory.usedJSHeapSize,
        totalJSHeapSize: perf.memory.totalJSHeapSize,
        jsHeapSizeLimit: perf.memory.jsHeapSizeLimit,
      };
    }
    return null;
  }

  static formatBytes(bytes: number): string {
    if (bytes < 1024) return `${bytes}B`;
    if (bytes < 1024 * 1024) return `${(bytes / 1024).toFixed(1)}KB`;
    return `${(bytes / (1024 * 1024)).toFixed(1)}MB`;
  }

  static renderOverlay(ctx: CanvasRenderingContext2D, x = 10, y = 200) {
    const mem = MemoryMonitor.getMemoryInfo();
    if (!mem) return;

    ctx.save();
    ctx.font = "12px monospace";
    ctx.fillStyle = "rgba(0, 0, 0, 0.5)";
    ctx.fillRect(x - 4, y - 12, 200, 36);

    ctx.fillStyle = "#00ccff";
    ctx.fillText(
      `Heap: ${MemoryMonitor.formatBytes(mem.usedJSHeapSize)} / ${MemoryMonitor.formatBytes(mem.totalJSHeapSize)}`,
      x, y
    );
    ctx.fillText(
      `Limit: ${MemoryMonitor.formatBytes(mem.jsHeapSizeLimit)}`,
      x, y + 16
    );
    ctx.restore();
  }
}
```

---

## 7. 구현 코드 예시 종합

### 7.1 WASM 비동기 초기화 및 Code Splitting

```typescript
// physics/wasm-loader.ts

/**
 * WASM 물리 엔진을 비동기로 로드한다.
 * dynamic import를 통해 code splitting을 적용하여
 * 초기 번들 크기를 최소화한다.
 */
export async function loadPhysicsEngine() {
  // Dynamic import → 별도 청크로 분리됨
  const module = await import("./rapier-wrapper");
  await module.init(); // WASM 바이너리 비동기 로드
  return module;
}

// 사용 예시
// const physics = await loadPhysicsEngine();
// physics.createWorld({ gravity: { x: 0, y: 9.81 } });
```

### 7.2 워커 기반 물리 처리 (선택적)

성능 병목이 확인된 경우에만 적용한다. 메인 스레드와 워커 간 통신 오버헤드를 고려하여 판단한다.

```typescript
// workers/physics-worker.ts
// Web Worker 내부 코드

interface PhysicsCommand {
  type: "init" | "step" | "addBody" | "removeBody";
  payload: unknown;
}

interface PhysicsResult {
  type: "stepped" | "initialized";
  bodies: Array<{ id: number; x: number; y: number; angle: number }>;
}

self.onmessage = async (e: MessageEvent<PhysicsCommand>) => {
  const { type, payload } = e.data;

  switch (type) {
    case "init": {
      // WASM 초기화
      // await initPhysics();
      self.postMessage({ type: "initialized" } as PhysicsResult);
      break;
    }
    case "step": {
      // 물리 시뮬레이션 1스텝 실행
      // const result = world.step(FIXED_DT);
      // self.postMessage({ type: "stepped", bodies: result });
      break;
    }
  }
};
```

```typescript
// physics/physics-worker-bridge.ts
// 메인 스레드에서 워커와 통신하는 브릿지

export class PhysicsWorkerBridge {
  private worker: Worker;
  private pendingResolve: ((bodies: any[]) => void) | null = null;

  constructor() {
    this.worker = new Worker(
      new URL("../workers/physics-worker.ts", import.meta.url),
      { type: "module" }
    );

    this.worker.onmessage = (e) => {
      if (e.data.type === "stepped" && this.pendingResolve) {
        this.pendingResolve(e.data.bodies);
        this.pendingResolve = null;
      }
    };
  }

  async init() {
    this.worker.postMessage({ type: "init" });
  }

  step(): Promise<any[]> {
    return new Promise((resolve) => {
      this.pendingResolve = resolve;
      this.worker.postMessage({ type: "step" });
    });
  }

  destroy() {
    this.worker.terminate();
  }
}
```

### 7.3 전체 통합: 게임 초기화 흐름

```typescript
// main.ts - 게임 진입점 통합 예시

import { loadPhysicsEngine } from "./physics/wasm-loader";
import { GameLoop } from "./core/game-loop";
import { PointerInputManager } from "./input/pointer-input";
import { AdaptiveQualityManager } from "./core/adaptive-quality";
import { AudioManager } from "./audio/audio-manager";
import { FPSCounter } from "./debug/fps-counter";
import { GameProfiler } from "./debug/profiler";
import { QUALITY_PRESETS } from "./config/quality-settings";

export async function initGame(canvas: HTMLCanvasElement) {
  // 1. WASM 물리 엔진 비동기 로드
  const physics = await loadPhysicsEngine();

  // 2. 입력 시스템 초기화
  const input = new PointerInputManager(canvas);

  // 3. 오디오 초기화
  const audio = new AudioManager();
  await audio.init();
  audio.setupVisibilityHandler();

  // 4. 품질 관리자 초기화
  const qualityManager = new AdaptiveQualityManager();
  let currentSettings = QUALITY_PRESETS.high;

  qualityManager.onQualityChange = (quality) => {
    currentSettings = QUALITY_PRESETS[quality];
    console.log(`품질 자동 조절: ${currentSettings.label}`);
  };

  // 5. 디버그 도구
  const fps = new FPSCounter();
  const profiler = new GameProfiler();
  profiler.setEnabled(import.meta.env.DEV);

  // 6. 게임 루프 시작
  const ctx = canvas.getContext("2d")!;

  const loop = new GameLoop(
    (dt) => {
      profiler.begin("physics");
      // physics.step(dt);
      profiler.end("physics");

      profiler.begin("gameLogic");
      // 게임 로직 업데이트
      profiler.end("gameLogic");
    },
    (interpolation) => {
      profiler.begin("render");
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      // 게임 렌더링 ...

      // 디버그 오버레이
      const currentFPS = fps.tick();
      qualityManager.recordFrame(currentFPS);

      if (import.meta.env.DEV) {
        fps.renderOverlay(ctx);
        profiler.renderOverlay(ctx);
      }
      profiler.end("render");
    }
  );

  loop.start();

  // 정리 함수 반환
  return () => {
    loop.stop();
    input.destroy();
  };
}
```

---

## 요약 체크리스트

| 카테고리 | 항목 | 상태 |
|----------|------|------|
| **레이아웃** | ResizeObserver 기반 Canvas 리사이즈 | 미구현 |
| **레이아웃** | Safe-area CSS 적용 | 미구현 |
| **레이아웃** | Portrait/Landscape 스케일 전환 | 미구현 |
| **입력** | Pointer Events 통합 | 미구현 |
| **입력** | touch-action: none 적용 | 미구현 |
| **입력** | 엄지 영역 터치존 설계 | 미구현 |
| **물리** | Fixed timestep + accumulator | 미구현 |
| **물리** | 바디/컨스트레인트 수 상한 | 미구현 |
| **물리** | Sleeping 활성화 | 미구현 |
| **물리** | Off-screen 오브젝트 제거 | 미구현 |
| **렌더링** | React setState 최소화 (ref 패턴) | 미구현 |
| **렌더링** | 오브젝트 풀링 | 미구현 |
| **성능** | WASM async init + code splitting | 미구현 |
| **성능** | 어댑티브 품질 시스템 | 미구현 |
| **모바일** | 오디오 resume 정책 대응 | 미구현 |
| **모바일** | Visibility/배터리 대응 | 미구현 |
| **디버그** | FPS 카운터 | 미구현 |
| **디버그** | 프로파일러 | 미구현 |
| **디버그** | 메모리 모니터 | 미구현 |
