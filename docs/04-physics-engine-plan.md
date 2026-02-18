# 물리 엔진 통합 구현 계획서

## 1. 물리 엔진 선택 근거 및 비교표

### 엔진 비교표

| 항목 | Matter.js | Planck.js | Rapier (WASM) |
|------|-----------|-----------|---------------|
| **언어** | JavaScript | JavaScript (Box2D 포트) | Rust → WASM |
| **2D/3D** | 2D 전용 | 2D 전용 | 2D + 3D |
| **번들 크기** | ~80KB (min) | ~120KB (min) | ~200KB+ (WASM) |
| **성능** | 중 (캐주얼 게임 충분) | 중상 (Box2D 기반) | 상 (네이티브급) |
| **학습 곡선** | 낮음 | 중간 (Box2D API) | 높음 (WASM 연동) |
| **React 적합도** | 높음 (직접 통합 용이) | 중간 | 낮음 (WASM 초기화 비동기) |
| **생태계/예제** | 매우 풍부 | 보통 | 성장 중 |
| **라이선스** | MIT | Zlib (허용적) | Apache 2.0 |
| **디버그 렌더러** | 내장 (Matter.Render) | 별도 구현 필요 | 별도 구현 필요 |
| **Sleeping 지원** | 내장 | 내장 | 내장 |
| **연속 충돌 감지 (CCD)** | 제한적 | 지원 | 지원 |

### Matter.js 선택 근거

"무한 계단" 게임은 다음과 같은 특성을 가진다:

1. **2D 캐주얼 게임에 최적**: 복잡한 물리 시뮬레이션이 필요 없고, 단순한 충돌 감지와 중력 처리만 요구된다. Matter.js는 이러한 경량 시뮬레이션에 최적화되어 있다.

2. **빠른 프로토타이핑**: API가 직관적이고, 엔진/월드/바디 생성이 몇 줄이면 충분하다. 개발 속도가 중요한 프로젝트에 적합하다.

3. **풍부한 생태계와 예제**: GitHub 스타 16k+, 수많은 튜토리얼과 코드 예제가 존재하여 문제 해결이 용이하다.

4. **MIT 라이선스**: 상업적 사용에 제한이 없다.

5. **내장 디버그 렌더러**: `Matter.Render`를 통해 즉시 물리 바디를 시각화할 수 있어 개발 효율이 높다.

6. **React 통합 용이**: 순수 JavaScript이므로 Context/Hook 패턴으로 쉽게 래핑할 수 있고, WASM 초기화 같은 비동기 문제가 없다.

---

## 2. PhysicsEngineProvider 상세 설계

### 아키텍처 개요

```
PhysicsEngineProvider (React Context)
├── Matter.Engine   ← 물리 시뮬레이션 핵심
├── Matter.World    ← 물리 바디 컨테이너
├── Game Loop       ← rAF + fixed timestep
├── Collision Events ← 충돌 콜백 관리
└── Debug Renderer  ← 개발 모드 시각화
```

### 타입 정의

```typescript
// src/physics/types.ts

import Matter from 'matter-js';

/** 물리 엔진 설정 */
export interface PhysicsConfig {
  /** 중력 벡터 (기본: { x: 0, y: 1 }) */
  gravity: { x: number; y: number };
  /** 고정 타임스텝 (기본: 1/60) */
  fixedTimestep: number;
  /** 최대 서브스텝 수 (기본: 5) */
  maxSubSteps: number;
  /** 바디 수면 활성화 (기본: true) */
  enableSleeping: boolean;
  /** 디버그 렌더링 (기본: DEV 모드) */
  debug: boolean;
  /** 최대 바디 수 (기본: 200) */
  maxBodies: number;
}

/** 물리 컨텍스트 값 */
export interface PhysicsContextValue {
  /** Matter.js 엔진 인스턴스 */
  engine: Matter.Engine;
  /** Matter.js 월드 인스턴스 */
  world: Matter.World;
  /** 물리 바디 추가 */
  addBody: (body: Matter.Body) => void;
  /** 물리 바디 제거 */
  removeBody: (body: Matter.Body) => void;
  /** 충돌 리스너 등록 */
  onCollision: (
    event: CollisionEventType,
    callback: CollisionCallback
  ) => () => void;
  /** 물리 엔진 일시정지 */
  pause: () => void;
  /** 물리 엔진 재개 */
  resume: () => void;
  /** 엔진 실행 중 여부 */
  isRunning: boolean;
  /** 현재 설정 */
  config: PhysicsConfig;
}

export type CollisionEventType =
  | 'collisionStart'
  | 'collisionActive'
  | 'collisionEnd';

export type CollisionCallback = (
  event: Matter.IEventCollision<Matter.Engine>
) => void;

/** 바디 카테고리 (비트마스크) */
export const BodyCategory = {
  DEFAULT: 0x0001,
  CHARACTER: 0x0002,
  STAIR: 0x0004,
  OBSTACLE: 0x0008,
  SENSOR: 0x0010,
  WALL: 0x0020,
} as const;

/** 바디 라벨 */
export const BodyLabel = {
  CHARACTER: 'character',
  STAIR: 'stair',
  OBSTACLE: 'obstacle',
  SCORE_SENSOR: 'score-sensor',
  DEATH_ZONE: 'death-zone',
  LEFT_WALL: 'left-wall',
  RIGHT_WALL: 'right-wall',
} as const;
```

### PhysicsEngineProvider 구현

```typescript
// src/physics/PhysicsEngineProvider.tsx

import React, {
  createContext,
  useContext,
  useRef,
  useCallback,
  useEffect,
  useState,
  useMemo,
} from 'react';
import Matter from 'matter-js';
import type {
  PhysicsConfig,
  PhysicsContextValue,
  CollisionEventType,
  CollisionCallback,
} from './types';

const DEFAULT_CONFIG: PhysicsConfig = {
  gravity: { x: 0, y: 1 },
  fixedTimestep: 1000 / 60, // ~16.67ms
  maxSubSteps: 5,
  enableSleeping: true,
  debug: import.meta.env.DEV,
  maxBodies: 200,
};

const PhysicsContext = createContext<PhysicsContextValue | null>(null);

interface PhysicsEngineProviderProps {
  children: React.ReactNode;
  config?: Partial<PhysicsConfig>;
}

export function PhysicsEngineProvider({
  children,
  config: configOverrides,
}: PhysicsEngineProviderProps) {
  const config = useMemo(
    () => ({ ...DEFAULT_CONFIG, ...configOverrides }),
    [configOverrides]
  );

  // --- 엔진/월드 생성 (한 번만) ---
  const engineRef = useRef<Matter.Engine | null>(null);
  const worldRef = useRef<Matter.World | null>(null);
  const runningRef = useRef(true);
  const accumulatorRef = useRef(0);
  const lastTimeRef = useRef(0);
  const rafIdRef = useRef<number>(0);
  const collisionListenersRef = useRef<
    Map<CollisionEventType, Set<CollisionCallback>>
  >(new Map());

  const [isRunning, setIsRunning] = useState(true);

  // 엔진 초기화
  if (!engineRef.current) {
    const engine = Matter.Engine.create({
      gravity: config.gravity,
      enableSleeping: config.enableSleeping,
    });
    engineRef.current = engine;
    worldRef.current = engine.world;
  }

  // --- 바디 추가/제거 ---
  const addBody = useCallback((body: Matter.Body) => {
    const world = worldRef.current;
    if (!world) return;

    // 최대 바디 수 체크
    const currentCount = Matter.Composite.allBodies(world).length;
    if (currentCount >= config.maxBodies) {
      console.warn(
        `[Physics] 최대 바디 수(${config.maxBodies}) 초과. 바디 추가 무시.`
      );
      return;
    }

    Matter.Composite.add(world, body);
  }, [config.maxBodies]);

  const removeBody = useCallback((body: Matter.Body) => {
    const world = worldRef.current;
    if (!world) return;
    Matter.Composite.remove(world, body);
  }, []);

  // --- 충돌 리스너 ---
  const onCollision = useCallback(
    (event: CollisionEventType, callback: CollisionCallback) => {
      if (!collisionListenersRef.current.has(event)) {
        collisionListenersRef.current.set(event, new Set());
      }
      collisionListenersRef.current.get(event)!.add(callback);

      // 해제 함수 반환
      return () => {
        collisionListenersRef.current.get(event)?.delete(callback);
      };
    },
    []
  );

  // --- 일시정지/재개 ---
  const pause = useCallback(() => {
    runningRef.current = false;
    setIsRunning(false);
  }, []);

  const resume = useCallback(() => {
    runningRef.current = true;
    setIsRunning(true);
    lastTimeRef.current = performance.now();
    accumulatorRef.current = 0;
  }, []);

  // --- 충돌 이벤트 바인딩 ---
  useEffect(() => {
    const engine = engineRef.current;
    if (!engine) return;

    const handleCollision =
      (type: CollisionEventType) =>
      (event: Matter.IEventCollision<Matter.Engine>) => {
        const listeners = collisionListenersRef.current.get(type);
        if (listeners) {
          listeners.forEach((cb) => cb(event));
        }
      };

    const onStart = handleCollision('collisionStart');
    const onActive = handleCollision('collisionActive');
    const onEnd = handleCollision('collisionEnd');

    Matter.Events.on(engine, 'collisionStart', onStart);
    Matter.Events.on(engine, 'collisionActive', onActive);
    Matter.Events.on(engine, 'collisionEnd', onEnd);

    return () => {
      Matter.Events.off(engine, 'collisionStart', onStart);
      Matter.Events.off(engine, 'collisionActive', onActive);
      Matter.Events.off(engine, 'collisionEnd', onEnd);
    };
  }, []);

  // --- 게임 루프 (rAF + fixed timestep) ---
  useEffect(() => {
    const engine = engineRef.current;
    if (!engine) return;

    lastTimeRef.current = performance.now();

    const loop = (currentTime: number) => {
      rafIdRef.current = requestAnimationFrame(loop);

      if (!runningRef.current) {
        lastTimeRef.current = currentTime;
        return;
      }

      const frameTime = currentTime - lastTimeRef.current;
      lastTimeRef.current = currentTime;

      // 프레임 시간이 너무 크면 제한 (탭 비활성 등)
      const clampedFrameTime = Math.min(frameTime, config.fixedTimestep * config.maxSubSteps);
      accumulatorRef.current += clampedFrameTime;

      let steps = 0;
      while (
        accumulatorRef.current >= config.fixedTimestep &&
        steps < config.maxSubSteps
      ) {
        Matter.Engine.update(engine, config.fixedTimestep);
        accumulatorRef.current -= config.fixedTimestep;
        steps++;
      }
    };

    rafIdRef.current = requestAnimationFrame(loop);

    return () => {
      cancelAnimationFrame(rafIdRef.current);
    };
  }, [config.fixedTimestep, config.maxSubSteps]);

  // --- 클린업 ---
  useEffect(() => {
    return () => {
      const engine = engineRef.current;
      if (engine) {
        Matter.Events.off(engine);
        Matter.Engine.clear(engine);
        Matter.World.clear(engine.world, false);
      }
      cancelAnimationFrame(rafIdRef.current);
      collisionListenersRef.current.clear();
    };
  }, []);

  // --- 컨텍스트 값 ---
  const contextValue = useMemo<PhysicsContextValue>(
    () => ({
      engine: engineRef.current!,
      world: worldRef.current!,
      addBody,
      removeBody,
      onCollision,
      pause,
      resume,
      isRunning,
      config,
    }),
    [addBody, removeBody, onCollision, pause, resume, isRunning, config]
  );

  return (
    <PhysicsContext.Provider value={contextValue}>
      {children}
    </PhysicsContext.Provider>
  );
}

/** 물리 엔진 컨텍스트 hook */
export function usePhysicsContext(): PhysicsContextValue {
  const ctx = useContext(PhysicsContext);
  if (!ctx) {
    throw new Error(
      'usePhysicsContext는 PhysicsEngineProvider 내부에서만 사용할 수 있습니다.'
    );
  }
  return ctx;
}
```

---

## 3. Fixed Timestep 구현

### 개념

물리 시뮬레이션은 프레임 간 시간 차이(deltaTime)가 일정해야 결정론적(deterministic)이고 안정적인 결과를 보장한다. 브라우저의 `requestAnimationFrame`은 디스플레이 주사율에 따라 변동되므로, **고정 타임스텝 + accumulator 패턴**을 사용한다.

### 동작 흐름

```
매 rAF 프레임마다:
  1. frameTime = currentTime - lastTime
  2. accumulator += frameTime
  3. while (accumulator >= fixedDt && steps < maxSteps):
       Engine.update(engine, fixedDt)
       accumulator -= fixedDt
       steps++
  4. alpha = accumulator / fixedDt  (보간 계수)
  5. 렌더링 시 alpha로 위치 보간
```

### 상세 구현

```typescript
// src/physics/useFixedTimestep.ts

import { useRef, useCallback } from 'react';
import Matter from 'matter-js';

interface FixedTimestepOptions {
  /** 고정 타임스텝 (ms). 기본값: 1000/60 */
  fixedDt?: number;
  /** 최대 서브스텝 수. 기본값: 5 */
  maxSubSteps?: number;
}

interface FixedTimestepState {
  accumulator: number;
  lastTime: number;
  /** 보간 계수 (0~1) - 렌더링 시 활용 */
  alpha: number;
}

/**
 * 고정 타임스텝 accumulator 패턴을 관리하는 hook.
 * 매 프레임 step()을 호출하면 내부적으로 고정 간격으로 물리를 갱신한다.
 */
export function useFixedTimestep(
  engine: Matter.Engine,
  options: FixedTimestepOptions = {}
) {
  const { fixedDt = 1000 / 60, maxSubSteps = 5 } = options;

  const stateRef = useRef<FixedTimestepState>({
    accumulator: 0,
    lastTime: 0,
    alpha: 0,
  });

  const step = useCallback(
    (currentTime: number) => {
      const state = stateRef.current;

      if (state.lastTime === 0) {
        state.lastTime = currentTime;
        return { steps: 0, alpha: 0 };
      }

      let frameTime = currentTime - state.lastTime;
      state.lastTime = currentTime;

      // 비활성 탭에서 돌아온 경우 시간 폭주 방지
      const maxFrameTime = fixedDt * maxSubSteps;
      if (frameTime > maxFrameTime) {
        frameTime = maxFrameTime;
      }

      state.accumulator += frameTime;

      let stepCount = 0;
      while (state.accumulator >= fixedDt && stepCount < maxSubSteps) {
        Matter.Engine.update(engine, fixedDt);
        state.accumulator -= fixedDt;
        stepCount++;
      }

      // 보간 계수: 남은 accumulator / fixedDt
      state.alpha = state.accumulator / fixedDt;

      return { steps: stepCount, alpha: state.alpha };
    },
    [engine, fixedDt, maxSubSteps]
  );

  const reset = useCallback(() => {
    stateRef.current = { accumulator: 0, lastTime: 0, alpha: 0 };
  }, []);

  return { step, reset, stateRef };
}
```

### 보간(Interpolation)으로 부드러운 렌더링

```typescript
// src/physics/interpolate.ts

import Matter from 'matter-js';

interface InterpolatedPosition {
  x: number;
  y: number;
  angle: number;
}

/**
 * 물리 업데이트 사이의 위치를 보간하여 부드러운 렌더링을 제공한다.
 *
 * @param body - Matter.js 바디
 * @param prevPosition - 이전 물리 스텝의 위치
 * @param alpha - 보간 계수 (0~1)
 */
export function interpolateBody(
  body: Matter.Body,
  prevPosition: { x: number; y: number; angle: number },
  alpha: number
): InterpolatedPosition {
  return {
    x: prevPosition.x + (body.position.x - prevPosition.x) * alpha,
    y: prevPosition.y + (body.position.y - prevPosition.y) * alpha,
    angle:
      prevPosition.angle + (body.angle - prevPosition.angle) * alpha,
  };
}

/**
 * 바디의 이전 위치를 저장하는 유틸리티.
 * 물리 스텝 직전에 호출한다.
 */
export function snapshotBodyPosition(body: Matter.Body) {
  return {
    x: body.position.x,
    y: body.position.y,
    angle: body.angle,
  };
}
```

### 왜 Fixed Timestep인가?

| 접근 방식 | 장점 | 단점 |
|-----------|------|------|
| 가변 타임스텝 | 구현 간단 | 비결정론적, 물리 불안정 |
| **고정 타임스텝** | **결정론적, 안정적** | accumulator 관리 필요 |
| 고정 + 보간 | 안정적 + 부드러운 렌더링 | 약간의 추가 메모리 |

무한 계단 게임에서는 캐릭터의 점프와 착지가 핵심 메카닉이므로, 고정 타임스텝으로 일관된 물리 동작을 보장하는 것이 필수적이다.

---

## 4. 충돌 감지 시스템 설계

### 충돌 이벤트 타입

| 이벤트 | 발생 시점 | 주요 용도 |
|--------|-----------|-----------|
| `collisionStart` | 두 바디가 처음 접촉 | 착지 감지, 점수 획득 |
| `collisionActive` | 접촉 중 매 프레임 | 지속 효과 (미끄러운 표면 등) |
| `collisionEnd` | 접촉 해제 | 공중 상태 진입 |

### 충돌 카테고리/마스크 설정

```typescript
// src/physics/collisionConfig.ts

import { BodyCategory } from './types';

/**
 * 충돌 필터 프리셋.
 * category: 이 바디가 속하는 카테고리
 * mask: 이 바디가 충돌하는 대상 카테고리
 */
export const CollisionFilter = {
  /** 캐릭터: 계단, 장애물, 벽과 충돌 */
  CHARACTER: {
    category: BodyCategory.CHARACTER,
    mask:
      BodyCategory.STAIR |
      BodyCategory.OBSTACLE |
      BodyCategory.WALL |
      BodyCategory.SENSOR,
    group: 0,
  },

  /** 계단: 캐릭터와만 충돌 */
  STAIR: {
    category: BodyCategory.STAIR,
    mask: BodyCategory.CHARACTER,
    group: 0,
  },

  /** 장애물: 캐릭터와만 충돌 */
  OBSTACLE: {
    category: BodyCategory.OBSTACLE,
    mask: BodyCategory.CHARACTER,
    group: 0,
  },

  /** 센서: 캐릭터와만 충돌 (물리 반응 없음) */
  SENSOR: {
    category: BodyCategory.SENSOR,
    mask: BodyCategory.CHARACTER,
    group: 0,
  },

  /** 벽: 캐릭터와만 충돌 */
  WALL: {
    category: BodyCategory.WALL,
    mask: BodyCategory.CHARACTER,
    group: 0,
  },
} as const;
```

### 충돌 핸들러 시스템

```typescript
// src/physics/useCollisionHandler.ts

import { useEffect, useCallback, useRef } from 'react';
import Matter from 'matter-js';
import { usePhysicsContext } from './PhysicsEngineProvider';
import { BodyLabel } from './types';

type CollisionPairHandler = (
  bodyA: Matter.Body,
  bodyB: Matter.Body,
  pair: Matter.Pair
) => void;

interface CollisionHandlerMap {
  [key: string]: CollisionPairHandler;
}

/**
 * 충돌 쌍을 라벨 기반으로 필터링하고 적절한 핸들러를 호출하는 hook.
 *
 * @example
 * useCollisionHandler('collisionStart', {
 *   'character:stair': (char, stair) => handleLanding(char, stair),
 *   'character:obstacle': (char, obs) => handleDeath(char, obs),
 *   'character:score-sensor': (char, sensor) => addScore(),
 * });
 */
export function useCollisionHandler(
  event: 'collisionStart' | 'collisionActive' | 'collisionEnd',
  handlers: CollisionHandlerMap
) {
  const { onCollision } = usePhysicsContext();
  const handlersRef = useRef(handlers);
  handlersRef.current = handlers;

  useEffect(() => {
    const unsubscribe = onCollision(event, (e) => {
      for (const pair of e.pairs) {
        const { bodyA, bodyB } = pair;
        const labelA = bodyA.label;
        const labelB = bodyB.label;

        // 양방향으로 키를 생성하여 핸들러를 찾는다
        const key1 = `${labelA}:${labelB}`;
        const key2 = `${labelB}:${labelA}`;

        const handler = handlersRef.current[key1];
        if (handler) {
          handler(bodyA, bodyB, pair);
          continue;
        }

        const handler2 = handlersRef.current[key2];
        if (handler2) {
          handler2(bodyB, bodyA, pair);
        }
      }
    });

    return unsubscribe;
  }, [event, onCollision]);
}
```

### 센서(Sensor) 바디 활용

센서는 충돌 이벤트는 발생하지만 물리적 반응(밀어내기)은 하지 않는 특수 바디이다. 게임에서 점수 영역, 사망 영역 등에 활용한다.

```typescript
// src/physics/createSensor.ts

import Matter from 'matter-js';
import { CollisionFilter } from './collisionConfig';
import { BodyLabel } from './types';

/**
 * 점수 획득 센서 생성.
 * 각 계단 위에 배치하여 캐릭터가 통과하면 점수를 준다.
 */
export function createScoreSensor(
  x: number,
  y: number,
  width: number
): Matter.Body {
  return Matter.Bodies.rectangle(x, y - 20, width, 10, {
    isSensor: true,
    isStatic: true,
    label: BodyLabel.SCORE_SENSOR,
    collisionFilter: CollisionFilter.SENSOR,
    render: {
      fillStyle: 'rgba(0, 255, 0, 0.3)',
    },
  });
}

/**
 * 사망 영역 센서 생성.
 * 화면 하단에 배치하여 캐릭터가 떨어지면 게임 오버.
 */
export function createDeathZone(
  x: number,
  y: number,
  width: number
): Matter.Body {
  return Matter.Bodies.rectangle(x, y, width, 50, {
    isSensor: true,
    isStatic: true,
    label: BodyLabel.DEATH_ZONE,
    collisionFilter: CollisionFilter.SENSOR,
    render: {
      fillStyle: 'rgba(255, 0, 0, 0.3)',
    },
  });
}
```

---

## 5. 물리 바디/월드 관리 전략

### 계단 바디 생성 (Static Body)

```typescript
// src/physics/bodies/stairBody.ts

import Matter from 'matter-js';
import { CollisionFilter } from '../collisionConfig';
import { BodyLabel } from '../types';

export interface StairConfig {
  x: number;
  y: number;
  width: number;
  height: number;
  /** 계단 방향: 'left' | 'right' */
  direction: 'left' | 'right';
  /** 계단 타입에 따른 마찰 계수 */
  friction?: number;
}

/**
 * 계단 물리 바디를 생성한다.
 * static body이므로 중력이나 힘의 영향을 받지 않는다.
 */
export function createStairBody(config: StairConfig): Matter.Body {
  const { x, y, width, height, direction, friction = 0.8 } = config;

  const body = Matter.Bodies.rectangle(x, y, width, height, {
    isStatic: true,
    label: BodyLabel.STAIR,
    friction,
    restitution: 0, // 반발 계수 0: 바운스 없음
    collisionFilter: CollisionFilter.STAIR,
    render: {
      fillStyle: direction === 'left' ? '#4a90d9' : '#d94a4a',
    },
  });

  // 커스텀 데이터 저장
  (body as any).gameData = {
    direction,
    scored: false, // 이미 점수를 획득했는지
    stairIndex: -1, // 풀링 시 인덱스
  };

  return body;
}
```

### 캐릭터 바디 생성 (Dynamic Body)

```typescript
// src/physics/bodies/characterBody.ts

import Matter from 'matter-js';
import { CollisionFilter } from '../collisionConfig';
import { BodyLabel } from '../types';

export interface CharacterConfig {
  x: number;
  y: number;
  width: number;
  height: number;
}

/**
 * 캐릭터 물리 바디를 생성한다.
 * dynamic body이므로 중력의 영향을 받는다.
 */
export function createCharacterBody(config: CharacterConfig): Matter.Body {
  const { x, y, width, height } = config;

  const body = Matter.Bodies.rectangle(x, y, width, height, {
    label: BodyLabel.CHARACTER,
    mass: 1,
    friction: 0.5,
    frictionAir: 0.01,
    restitution: 0, // 바운스 없음
    inertia: Infinity, // 회전 방지
    inverseInertia: 0,
    collisionFilter: CollisionFilter.CHARACTER,
    render: {
      fillStyle: '#ffcc00',
    },
  });

  // 캐릭터 상태 데이터
  (body as any).gameData = {
    isGrounded: false,
    canJump: true,
    facingDirection: 'right' as 'left' | 'right',
    jumpCount: 0,
  };

  return body;
}

/**
 * 캐릭터에 점프 힘을 적용한다.
 */
export function applyJump(
  body: Matter.Body,
  direction: 'left' | 'right',
  force: { x: number; y: number }
) {
  const xForce = direction === 'left' ? -Math.abs(force.x) : Math.abs(force.x);

  // 현재 속도를 초기화하고 점프 힘 적용
  Matter.Body.setVelocity(body, { x: 0, y: 0 });
  Matter.Body.applyForce(body, body.position, {
    x: xForce,
    y: -Math.abs(force.y), // 항상 위로
  });

  (body as any).gameData.isGrounded = false;
  (body as any).gameData.canJump = false;
}
```

### 오브젝트 풀링 (계단 재활용)

```typescript
// src/physics/StairPool.ts

import Matter from 'matter-js';
import { createStairBody, StairConfig } from './bodies/stairBody';

/**
 * 계단 바디 오브젝트 풀.
 * 빈번한 바디 생성/삭제를 방지하여 GC 부담을 줄인다.
 */
export class StairPool {
  private pool: Matter.Body[] = [];
  private active: Set<Matter.Body> = new Set();
  private world: Matter.World;

  constructor(world: Matter.World, initialSize: number = 30) {
    this.world = world;
    this.prewarm(initialSize);
  }

  /** 풀을 초기 바디로 채운다 */
  private prewarm(count: number) {
    for (let i = 0; i < count; i++) {
      const body = createStairBody({
        x: -9999,
        y: -9999,
        width: 80,
        height: 16,
        direction: 'left',
      });
      (body as any).gameData.stairIndex = i;
      this.pool.push(body);
    }
  }

  /**
   * 풀에서 계단 바디를 가져온다.
   * 풀이 비어있으면 새로 생성한다.
   */
  acquire(config: StairConfig): Matter.Body {
    let body: Matter.Body;

    if (this.pool.length > 0) {
      body = this.pool.pop()!;
      // 기존 바디를 재설정
      Matter.Body.setPosition(body, { x: config.x, y: config.y });
      Matter.Body.setAngle(body, 0);
      Matter.Body.setVelocity(body, { x: 0, y: 0 });

      // 바운드 재설정을 위해 vertices 업데이트
      const vertices = Matter.Vertices.fromPath(
        `0 0 ${config.width} 0 ${config.width} ${config.height} 0 ${config.height}`
      );
      Matter.Body.setVertices(body, vertices);
      Matter.Body.setPosition(body, { x: config.x, y: config.y });

      (body as any).gameData.direction = config.direction;
      (body as any).gameData.scored = false;
    } else {
      body = createStairBody(config);
    }

    // 월드에 추가
    Matter.Composite.add(this.world, body);
    this.active.add(body);

    // Sleeping 해제
    Matter.Sleeping.set(body, false);

    return body;
  }

  /**
   * 계단 바디를 풀에 반환한다.
   * 실제로 삭제하지 않고 재활용 가능 상태로 전환한다.
   */
  release(body: Matter.Body) {
    if (!this.active.has(body)) return;

    this.active.delete(body);
    Matter.Composite.remove(this.world, body);

    // 화면 밖으로 이동
    Matter.Body.setPosition(body, { x: -9999, y: -9999 });
    Matter.Sleeping.set(body, true);

    this.pool.push(body);
  }

  /** 활성 바디 수 */
  get activeCount(): number {
    return this.active.size;
  }

  /** 풀 내 대기 바디 수 */
  get pooledCount(): number {
    return this.pool.length;
  }

  /** 모든 바디를 정리한다 */
  clear() {
    this.active.forEach((body) => {
      Matter.Composite.remove(this.world, body);
    });
    this.active.clear();
    this.pool = [];
  }
}
```

### 오프스크린 바디 자동 제거

```typescript
// src/physics/useOffscreenCulling.ts

import { useEffect, useRef } from 'react';
import Matter from 'matter-js';
import { usePhysicsContext } from './PhysicsEngineProvider';
import { StairPool } from './StairPool';

interface CullingOptions {
  /** 화면 상단 기준 얼마나 위까지 유지할지 (px) */
  topMargin?: number;
  /** 화면 하단 기준 얼마나 아래까지 유지할지 (px) */
  bottomMargin?: number;
  /** 컬링 체크 주기 (프레임) */
  checkInterval?: number;
}

/**
 * 화면 밖 계단 바디를 자동으로 풀에 반환하는 hook.
 */
export function useOffscreenCulling(
  stairPool: StairPool,
  cameraY: React.MutableRefObject<number>,
  options: CullingOptions = {}
) {
  const { topMargin = 200, bottomMargin = 600, checkInterval = 10 } = options;
  const { world } = usePhysicsContext();
  const frameCountRef = useRef(0);

  useEffect(() => {
    const checkCulling = () => {
      frameCountRef.current++;
      if (frameCountRef.current % checkInterval !== 0) return;

      const viewportTop = cameraY.current - topMargin;
      const viewportBottom = cameraY.current + window.innerHeight + bottomMargin;

      const bodies = Matter.Composite.allBodies(world);
      for (const body of bodies) {
        if (body.label !== 'stair') continue;
        if (body.isStatic && body.position.y > viewportBottom) {
          stairPool.release(body);
        }
        if (body.isStatic && body.position.y < viewportTop) {
          stairPool.release(body);
        }
      }
    };

    // Matter.Events에 afterUpdate로 등록
    const engine = (world as any).engine;
    if (engine) {
      Matter.Events.on(engine, 'afterUpdate', checkCulling);
      return () => {
        Matter.Events.off(engine, 'afterUpdate', checkCulling);
      };
    }
  }, [world, stairPool, cameraY, topMargin, bottomMargin, checkInterval]);
}
```

### 바디 수면(Sleeping) 활성화

```typescript
// 엔진 생성 시 Sleeping 활성화
const engine = Matter.Engine.create({
  enableSleeping: true,
});

// 수면 이벤트 모니터링 (디버그 용도)
Matter.Events.on(engine, 'sleepStart', (event) => {
  const body = event.source as Matter.Body;
  console.debug(`[Sleep] ${body.label} 수면 시작`);
});

Matter.Events.on(engine, 'sleepEnd', (event) => {
  const body = event.source as Matter.Body;
  console.debug(`[Sleep] ${body.label} 수면 해제`);
});
```

---

## 6. 성능 최적화 전략

### 바디 수 상한 관리

```typescript
// src/physics/performanceConfig.ts

/**
 * 디바이스 성능에 따른 물리 설정 프리셋.
 */
export const PerformancePresets = {
  HIGH: {
    maxBodies: 200,
    enableSleeping: true,
    fixedTimestep: 1000 / 60,
    maxSubSteps: 5,
    stairPoolSize: 50,
    cullingCheckInterval: 10,
  },
  MEDIUM: {
    maxBodies: 150,
    enableSleeping: true,
    fixedTimestep: 1000 / 60,
    maxSubSteps: 3,
    stairPoolSize: 30,
    cullingCheckInterval: 5,
  },
  LOW: {
    maxBodies: 100,
    enableSleeping: true,
    fixedTimestep: 1000 / 60,
    maxSubSteps: 2,
    stairPoolSize: 20,
    cullingCheckInterval: 3,
  },
} as const;

/**
 * 간단한 디바이스 성능 감지.
 * hardwareConcurrency와 deviceMemory를 기반으로 판단한다.
 */
export function detectPerformanceTier(): keyof typeof PerformancePresets {
  const cores = navigator.hardwareConcurrency || 2;
  const memory = (navigator as any).deviceMemory || 4;

  if (cores >= 8 && memory >= 8) return 'HIGH';
  if (cores >= 4 && memory >= 4) return 'MEDIUM';
  return 'LOW';
}
```

### 매 프레임 setState 금지, ref 사용

React의 `setState`는 리렌더링을 유발하므로, 매 물리 프레임마다 호출하면 심각한 성능 저하를 일으킨다. 물리 상태는 반드시 `useRef`로 관리하고, 렌더링이 필요한 경우에만 선택적으로 setState를 호출한다.

```typescript
// src/physics/usePhysicsBody.ts

import { useRef, useEffect, useCallback } from 'react';
import Matter from 'matter-js';
import { usePhysicsContext } from './PhysicsEngineProvider';

interface BodyPosition {
  x: number;
  y: number;
  angle: number;
}

/**
 * 물리 바디의 위치를 추적하는 hook.
 * ref로 관리하여 불필요한 리렌더링을 방지한다.
 */
export function usePhysicsBody(body: Matter.Body | null) {
  const { engine } = usePhysicsContext();

  // 위치를 ref로 관리 (매 프레임 업데이트, 리렌더링 없음)
  const positionRef = useRef<BodyPosition>({ x: 0, y: 0, angle: 0 });

  // DOM 엘리먼트를 직접 조작하기 위한 ref
  const elementRef = useRef<HTMLDivElement | null>(null);

  useEffect(() => {
    if (!body) return;

    const updatePosition = () => {
      positionRef.current = {
        x: body.position.x,
        y: body.position.y,
        angle: body.angle,
      };

      // DOM 직접 조작 (setState 없이)
      if (elementRef.current) {
        const { x, y, angle } = positionRef.current;
        elementRef.current.style.transform =
          `translate(${x}px, ${y}px) rotate(${angle}rad)`;
      }
    };

    Matter.Events.on(engine, 'afterUpdate', updatePosition);
    return () => {
      Matter.Events.off(engine, 'afterUpdate', updatePosition);
    };
  }, [body, engine]);

  return { positionRef, elementRef };
}
```

### Broad-phase 최적화

Matter.js는 기본적으로 Grid 기반의 broad-phase 충돌 감지를 사용한다. 게임 특성에 맞게 Grid 크기를 조정하여 성능을 개선할 수 있다.

```typescript
// 엔진 생성 시 Broad-phase 설정
const engine = Matter.Engine.create({
  enableSleeping: true,
});

// Grid Broadphase 설정 (Matter.js 기본값)
// 게임 내 오브젝트 크기에 맞춰 버킷 크기 조정
engine.broadphase = {
  controller: (Matter as any).Grid,
  instance: (Matter as any).Grid.create({
    bucketWidth: 100, // 계단 너비에 맞춤
    bucketHeight: 100,
  }),
};
```

### 성능 모니터링

```typescript
// src/physics/usePhysicsStats.ts

import { useRef, useEffect, useCallback } from 'react';
import Matter from 'matter-js';
import { usePhysicsContext } from './PhysicsEngineProvider';

interface PhysicsStats {
  bodyCount: number;
  activeBodyCount: number;
  sleepingBodyCount: number;
  pairCount: number;
  fps: number;
  updateTime: number;
}

/**
 * 물리 엔진 성능 통계를 수집하는 hook.
 * DEV 모드에서만 활성화된다.
 */
export function usePhysicsStats() {
  const { engine, world } = usePhysicsContext();
  const statsRef = useRef<PhysicsStats>({
    bodyCount: 0,
    activeBodyCount: 0,
    sleepingBodyCount: 0,
    pairCount: 0,
    fps: 0,
    updateTime: 0,
  });

  const frameTimesRef = useRef<number[]>([]);

  useEffect(() => {
    if (!import.meta.env.DEV) return;

    const updateStats = () => {
      const bodies = Matter.Composite.allBodies(world);
      const sleeping = bodies.filter((b) => b.isSleeping);

      const now = performance.now();
      frameTimesRef.current.push(now);

      // 최근 60프레임의 FPS 계산
      const cutoff = now - 1000;
      frameTimesRef.current = frameTimesRef.current.filter(
        (t) => t > cutoff
      );

      statsRef.current = {
        bodyCount: bodies.length,
        activeBodyCount: bodies.length - sleeping.length,
        sleepingBodyCount: sleeping.length,
        pairCount: engine.pairs?.list?.length ?? 0,
        fps: frameTimesRef.current.length,
        updateTime: engine.timing.lastDelta,
      };
    };

    Matter.Events.on(engine, 'afterUpdate', updateStats);
    return () => {
      Matter.Events.off(engine, 'afterUpdate', updateStats);
    };
  }, [engine, world]);

  return statsRef;
}
```

---

## 7. 디버그 렌더러 구현

### Matter.Render 활용

```typescript
// src/physics/DebugRenderer.tsx

import React, { useEffect, useRef } from 'react';
import Matter from 'matter-js';
import { usePhysicsContext } from './PhysicsEngineProvider';

interface DebugRendererProps {
  /** 디버그 렌더러 표시 여부 */
  enabled?: boolean;
  /** 캔버스 너비 */
  width?: number;
  /** 캔버스 높이 */
  height?: number;
  /** 배경 투명도 */
  opacity?: number;
}

/**
 * Matter.js 내장 렌더러를 이용한 디버그 오버레이.
 * DEV 모드에서만 사용한다.
 */
export function DebugRenderer({
  enabled = import.meta.env.DEV,
  width = 390,
  height = 844,
  opacity = 0.7,
}: DebugRendererProps) {
  const { engine } = usePhysicsContext();
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const renderRef = useRef<Matter.Render | null>(null);

  useEffect(() => {
    if (!enabled || !canvasRef.current) return;

    const render = Matter.Render.create({
      canvas: canvasRef.current,
      engine,
      options: {
        width,
        height,
        wireframes: true,
        showSleeping: true,
        showCollisions: true,
        showVelocity: true,
        showAngleIndicator: true,
        showBounds: false,
        showIds: false,
        showShadows: false,
        background: 'transparent',
        wireframeBackground: 'transparent',
      },
    });

    Matter.Render.run(render);
    renderRef.current = render;

    return () => {
      Matter.Render.stop(render);
      renderRef.current = null;
    };
  }, [enabled, engine, width, height]);

  if (!enabled) return null;

  return (
    <canvas
      ref={canvasRef}
      width={width}
      height={height}
      style={{
        position: 'absolute',
        top: 0,
        left: 0,
        pointerEvents: 'none',
        opacity,
        zIndex: 9999,
      }}
    />
  );
}
```

### 커스텀 와이어프레임 렌더러

Matter.Render보다 가볍고, 게임 카메라와 동기화되는 커스텀 렌더러가 필요할 수 있다.

```typescript
// src/physics/CustomDebugRenderer.tsx

import React, { useEffect, useRef } from 'react';
import Matter from 'matter-js';
import { usePhysicsContext } from './PhysicsEngineProvider';
import { usePhysicsStats } from './usePhysicsStats';

interface CustomDebugRendererProps {
  enabled?: boolean;
  cameraY: React.MutableRefObject<number>;
}

/**
 * 커스텀 디버그 렌더러.
 * Canvas 2D로 바디 외곽선, 속도 벡터, 충돌 포인트를 직접 그린다.
 */
export function CustomDebugRenderer({
  enabled = import.meta.env.DEV,
  cameraY,
}: CustomDebugRendererProps) {
  const { engine, world } = usePhysicsContext();
  const canvasRef = useRef<HTMLCanvasElement>(null);
  const statsRef = usePhysicsStats();

  useEffect(() => {
    if (!enabled) return;

    const canvas = canvasRef.current;
    if (!canvas) return;
    const ctx = canvas.getContext('2d');
    if (!ctx) return;

    const draw = () => {
      const { width, height } = canvas;
      ctx.clearRect(0, 0, width, height);

      ctx.save();
      // 카메라 오프셋 적용
      ctx.translate(0, -cameraY.current);

      const bodies = Matter.Composite.allBodies(world);

      for (const body of bodies) {
        const vertices = body.vertices;

        // 바디 외곽선
        ctx.beginPath();
        ctx.moveTo(vertices[0].x, vertices[0].y);
        for (let i = 1; i < vertices.length; i++) {
          ctx.lineTo(vertices[i].x, vertices[i].y);
        }
        ctx.closePath();

        // 수면 상태에 따른 색상
        if (body.isSleeping) {
          ctx.strokeStyle = 'rgba(100, 100, 100, 0.5)';
        } else if (body.isSensor) {
          ctx.strokeStyle = 'rgba(0, 255, 0, 0.7)';
        } else if (body.isStatic) {
          ctx.strokeStyle = 'rgba(0, 150, 255, 0.8)';
        } else {
          ctx.strokeStyle = 'rgba(255, 200, 0, 0.9)';
        }

        ctx.lineWidth = 2;
        ctx.stroke();

        // 속도 벡터 (dynamic body만)
        if (!body.isStatic) {
          ctx.beginPath();
          ctx.moveTo(body.position.x, body.position.y);
          ctx.lineTo(
            body.position.x + body.velocity.x * 5,
            body.position.y + body.velocity.y * 5
          );
          ctx.strokeStyle = 'rgba(255, 0, 0, 0.8)';
          ctx.lineWidth = 2;
          ctx.stroke();
        }

        // 라벨 표시
        ctx.fillStyle = 'rgba(255, 255, 255, 0.8)';
        ctx.font = '10px monospace';
        ctx.fillText(body.label, body.position.x - 15, body.position.y - 10);
      }

      // 충돌 포인트 시각화
      const pairs = engine.pairs?.list ?? [];
      for (const pair of pairs) {
        if (!pair.isActive) continue;
        for (const contact of pair.activeContacts) {
          const vertex = contact.vertex;
          ctx.beginPath();
          ctx.arc(vertex.x, vertex.y, 4, 0, Math.PI * 2);
          ctx.fillStyle = 'rgba(255, 0, 255, 0.9)';
          ctx.fill();
        }
      }

      ctx.restore();

      // 통계 오버레이
      const stats = statsRef.current;
      ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
      ctx.fillRect(5, 5, 180, 100);
      ctx.fillStyle = '#00ff00';
      ctx.font = '12px monospace';
      ctx.fillText(`FPS: ${stats.fps}`, 10, 20);
      ctx.fillText(`Bodies: ${stats.bodyCount}`, 10, 35);
      ctx.fillText(`Active: ${stats.activeBodyCount}`, 10, 50);
      ctx.fillText(`Sleeping: ${stats.sleepingBodyCount}`, 10, 65);
      ctx.fillText(`Pairs: ${stats.pairCount}`, 10, 80);
      ctx.fillText(`dt: ${stats.updateTime.toFixed(2)}ms`, 10, 95);
    };

    Matter.Events.on(engine, 'afterUpdate', draw);
    return () => {
      Matter.Events.off(engine, 'afterUpdate', draw);
    };
  }, [enabled, engine, world, cameraY, statsRef]);

  if (!enabled) return null;

  return (
    <canvas
      ref={canvasRef}
      width={390}
      height={844}
      style={{
        position: 'absolute',
        top: 0,
        left: 0,
        pointerEvents: 'none',
        zIndex: 9999,
      }}
    />
  );
}
```

### 디버그 토글 컴포넌트

```typescript
// src/physics/DebugToggle.tsx

import React, { useState } from 'react';

interface DebugToggleProps {
  onToggle: (enabled: boolean) => void;
}

/**
 * 디버그 렌더러 온/오프 토글.
 * DEV 모드에서만 렌더링된다.
 */
export function DebugToggle({ onToggle }: DebugToggleProps) {
  const [enabled, setEnabled] = useState(false);

  if (!import.meta.env.DEV) return null;

  return (
    <button
      onClick={() => {
        const next = !enabled;
        setEnabled(next);
        onToggle(next);
      }}
      style={{
        position: 'fixed',
        bottom: 10,
        right: 10,
        zIndex: 10000,
        padding: '8px 12px',
        background: enabled ? '#ff4444' : '#444',
        color: '#fff',
        border: 'none',
        borderRadius: 4,
        cursor: 'pointer',
        fontSize: 12,
        fontFamily: 'monospace',
      }}
    >
      Physics Debug: {enabled ? 'ON' : 'OFF'}
    </button>
  );
}
```

---

## 8. 핵심 코드 예시

### usePhysics() 커스텀 Hook

```typescript
// src/physics/usePhysics.ts

import { useCallback, useRef, useEffect } from 'react';
import Matter from 'matter-js';
import { usePhysicsContext } from './PhysicsEngineProvider';
import { createStairBody, StairConfig } from './bodies/stairBody';
import { createCharacterBody, applyJump, CharacterConfig } from './bodies/characterBody';
import { createScoreSensor, createDeathZone } from './createSensor';
import { StairPool } from './StairPool';
import { BodyLabel } from './types';

interface UsePhysicsReturn {
  /** 캐릭터 바디 */
  characterRef: React.MutableRefObject<Matter.Body | null>;
  /** 계단 풀 */
  stairPoolRef: React.MutableRefObject<StairPool | null>;
  /** 캐릭터 초기화 */
  initCharacter: (config: CharacterConfig) => void;
  /** 캐릭터 점프 */
  jump: (direction: 'left' | 'right') => void;
  /** 계단 추가 */
  addStair: (config: StairConfig) => Matter.Body;
  /** 사망 영역 설정 */
  setupDeathZone: (y: number, width: number) => void;
  /** 모든 물리 바디 정리 */
  cleanup: () => void;
}

/**
 * 게임 물리를 통합 관리하는 메인 hook.
 * PhysicsEngineProvider 내부에서 사용한다.
 */
export function usePhysics(): UsePhysicsReturn {
  const { engine, world, addBody, removeBody } = usePhysicsContext();

  const characterRef = useRef<Matter.Body | null>(null);
  const stairPoolRef = useRef<StairPool | null>(null);
  const deathZoneRef = useRef<Matter.Body | null>(null);

  // 계단 풀 초기화
  useEffect(() => {
    stairPoolRef.current = new StairPool(world, 30);
    return () => {
      stairPoolRef.current?.clear();
    };
  }, [world]);

  // --- 캐릭터 초기화 ---
  const initCharacter = useCallback(
    (config: CharacterConfig) => {
      // 기존 캐릭터 제거
      if (characterRef.current) {
        removeBody(characterRef.current);
      }

      const body = createCharacterBody(config);
      addBody(body);
      characterRef.current = body;
    },
    [addBody, removeBody]
  );

  // --- 점프 ---
  const jump = useCallback(
    (direction: 'left' | 'right') => {
      const body = characterRef.current;
      if (!body) return;

      const gameData = (body as any).gameData;
      if (!gameData.isGrounded) return;

      applyJump(body, direction, {
        x: 0.005, // 수평 힘
        y: 0.008, // 수직 힘
      });

      gameData.facingDirection = direction;
      gameData.jumpCount++;
    },
    []
  );

  // --- 계단 추가 ---
  const addStair = useCallback(
    (config: StairConfig): Matter.Body => {
      const pool = stairPoolRef.current;
      if (!pool) {
        // 풀이 아직 초기화되지 않은 경우 직접 생성
        const body = createStairBody(config);
        addBody(body);
        return body;
      }
      return pool.acquire(config);
    },
    [addBody]
  );

  // --- 사망 영역 ---
  const setupDeathZone = useCallback(
    (y: number, width: number) => {
      if (deathZoneRef.current) {
        removeBody(deathZoneRef.current);
      }
      const zone = createDeathZone(width / 2, y, width);
      addBody(zone);
      deathZoneRef.current = zone;
    },
    [addBody, removeBody]
  );

  // --- 정리 ---
  const cleanup = useCallback(() => {
    if (characterRef.current) {
      removeBody(characterRef.current);
      characterRef.current = null;
    }
    if (deathZoneRef.current) {
      removeBody(deathZoneRef.current);
      deathZoneRef.current = null;
    }
    stairPoolRef.current?.clear();
  }, [removeBody]);

  return {
    characterRef,
    stairPoolRef,
    initCharacter,
    jump,
    addStair,
    setupDeathZone,
    cleanup,
  };
}
```

### 충돌 핸들러 등록 코드

```typescript
// src/game/useGameCollisions.ts

import { useCallback } from 'react';
import Matter from 'matter-js';
import { useCollisionHandler } from '../physics/useCollisionHandler';
import { BodyLabel } from '../physics/types';

interface GameCollisionHandlers {
  onLanding: (stairBody: Matter.Body) => void;
  onDeath: () => void;
  onScore: (sensorBody: Matter.Body) => void;
}

/**
 * 게임 내 충돌 이벤트를 처리하는 hook.
 */
export function useGameCollisions(handlers: GameCollisionHandlers) {
  const { onLanding, onDeath, onScore } = handlers;

  // --- 충돌 시작: 착지, 사망, 점수 ---
  useCollisionHandler('collisionStart', {
    // 캐릭터가 계단에 착지
    [`${BodyLabel.CHARACTER}:${BodyLabel.STAIR}`]: (
      charBody,
      stairBody,
      pair
    ) => {
      const gameData = (charBody as any).gameData;

      // 위에서 착지한 경우만 처리 (아래에서 부딪힌 건 무시)
      const normal = pair.collision.normal;
      if (normal.y < -0.5) {
        gameData.isGrounded = true;
        gameData.canJump = true;
        onLanding(stairBody);
      }
    },

    // 캐릭터가 장애물에 충돌
    [`${BodyLabel.CHARACTER}:${BodyLabel.OBSTACLE}`]: () => {
      onDeath();
    },

    // 캐릭터가 사망 영역에 진입
    [`${BodyLabel.CHARACTER}:${BodyLabel.DEATH_ZONE}`]: () => {
      onDeath();
    },

    // 캐릭터가 점수 센서 통과
    [`${BodyLabel.CHARACTER}:${BodyLabel.SCORE_SENSOR}`]: (
      _charBody,
      sensorBody
    ) => {
      const sensorData = (sensorBody as any).gameData;
      if (!sensorData?.scored) {
        sensorData.scored = true;
        onScore(sensorBody);
      }
    },
  });

  // --- 충돌 종료: 공중 상태 ---
  useCollisionHandler('collisionEnd', {
    [`${BodyLabel.CHARACTER}:${BodyLabel.STAIR}`]: (charBody) => {
      const gameData = (charBody as any).gameData;
      gameData.isGrounded = false;
    },
  });
}
```

### 게임 루프 통합 코드

```typescript
// src/game/GameScene.tsx

import React, { useEffect, useRef, useCallback } from 'react';
import { PhysicsEngineProvider } from '../physics/PhysicsEngineProvider';
import { usePhysics } from '../physics/usePhysics';
import { useGameCollisions } from './useGameCollisions';
import { DebugRenderer } from '../physics/DebugRenderer';
import { DebugToggle } from '../physics/DebugToggle';
import { detectPerformanceTier, PerformancePresets } from '../physics/performanceConfig';

/**
 * 게임 씬: 물리 엔진과 게임 로직을 통합한다.
 */
function GameSceneInner() {
  const {
    characterRef,
    initCharacter,
    jump,
    addStair,
    setupDeathZone,
    cleanup,
  } = usePhysics();

  const scoreRef = useRef(0);
  const cameraYRef = useRef(0);
  const gameOverRef = useRef(false);
  const containerRef = useRef<HTMLDivElement>(null);

  // --- 충돌 핸들러 ---
  const handleLanding = useCallback((stairBody: Matter.Body) => {
    // 카메라를 캐릭터 위치로 부드럽게 이동
    const charBody = characterRef.current;
    if (charBody) {
      cameraYRef.current = charBody.position.y - 400;
    }

    // 새 계단 생성 (무한 생성)
    const stairData = (stairBody as any).gameData;
    generateNextStairs(stairBody.position.y);
  }, [characterRef]);

  const handleDeath = useCallback(() => {
    if (gameOverRef.current) return;
    gameOverRef.current = true;
    console.log('Game Over! Score:', scoreRef.current);
    // 게임 오버 UI 표시 로직
  }, []);

  const handleScore = useCallback(() => {
    scoreRef.current += 1;
    // 점수 UI 업데이트 (DOM 직접 조작으로 리렌더 방지)
    const scoreEl = document.getElementById('score-display');
    if (scoreEl) {
      scoreEl.textContent = String(scoreRef.current);
    }
  }, []);

  useGameCollisions({
    onLanding: handleLanding,
    onDeath: handleDeath,
    onScore: handleScore,
  });

  // --- 계단 생성 ---
  const lastStairYRef = useRef(500);

  const generateNextStairs = useCallback(
    (currentY: number) => {
      // 현재 위치 위쪽으로 계단이 부족하면 추가 생성
      while (lastStairYRef.current > currentY - 800) {
        const direction = Math.random() > 0.5 ? 'left' : 'right';
        const x = direction === 'left' ? 120 : 270;

        addStair({
          x,
          y: lastStairYRef.current,
          width: 80,
          height: 16,
          direction,
        });

        lastStairYRef.current -= 60 + Math.random() * 40;
      }
    },
    [addStair]
  );

  // --- 초기화 ---
  useEffect(() => {
    // 캐릭터 생성
    initCharacter({
      x: 195,
      y: 450,
      width: 30,
      height: 40,
    });

    // 사망 영역 설정
    setupDeathZone(1000, 390);

    // 초기 계단 생성
    for (let i = 0; i < 10; i++) {
      const direction = i % 2 === 0 ? 'left' : 'right';
      const x = direction === 'left' ? 120 : 270;
      addStair({
        x,
        y: 500 - i * 80,
        width: 80,
        height: 16,
        direction,
      });
    }

    lastStairYRef.current = 500 - 10 * 80;

    return cleanup;
  }, [initCharacter, setupDeathZone, addStair, cleanup]);

  // --- 입력 처리 ---
  useEffect(() => {
    const handleKeyDown = (e: KeyboardEvent) => {
      if (gameOverRef.current) return;

      switch (e.key) {
        case 'ArrowLeft':
          jump('left');
          break;
        case 'ArrowRight':
          jump('right');
          break;
      }
    };

    const handleTouchStart = (e: TouchEvent) => {
      if (gameOverRef.current) return;

      const touch = e.touches[0];
      const halfWidth = window.innerWidth / 2;

      if (touch.clientX < halfWidth) {
        jump('left');
      } else {
        jump('right');
      }
    };

    window.addEventListener('keydown', handleKeyDown);
    window.addEventListener('touchstart', handleTouchStart);

    return () => {
      window.removeEventListener('keydown', handleKeyDown);
      window.removeEventListener('touchstart', handleTouchStart);
    };
  }, [jump]);

  // --- 카메라 추적 (rAF 기반, setState 없음) ---
  useEffect(() => {
    let rafId: number;

    const updateCamera = () => {
      rafId = requestAnimationFrame(updateCamera);

      const char = characterRef.current;
      if (!char || !containerRef.current) return;

      // 부드러운 카메라 추적
      const targetY = char.position.y - 400;
      cameraYRef.current += (targetY - cameraYRef.current) * 0.1;

      // DOM 직접 조작 (리렌더 없이 transform)
      containerRef.current.style.transform =
        `translateY(${-cameraYRef.current}px)`;
    };

    rafId = requestAnimationFrame(updateCamera);
    return () => cancelAnimationFrame(rafId);
  }, [characterRef]);

  return (
    <div
      style={{
        width: 390,
        height: 844,
        overflow: 'hidden',
        position: 'relative',
        background: '#1a1a2e',
      }}
    >
      {/* 점수 표시 */}
      <div
        style={{
          position: 'absolute',
          top: 20,
          left: '50%',
          transform: 'translateX(-50%)',
          color: '#fff',
          fontSize: 32,
          fontWeight: 'bold',
          zIndex: 100,
          fontFamily: 'monospace',
        }}
      >
        <span id="score-display">0</span>
      </div>

      {/* 게임 월드 컨테이너 */}
      <div ref={containerRef} style={{ position: 'absolute', width: '100%' }}>
        {/* 게임 오브젝트들은 물리 엔진이 DOM 직접 조작 */}
      </div>

      {/* 디버그 렌더러 */}
      <DebugRenderer />
    </div>
  );
}

/**
 * 게임 씬 래퍼: PhysicsEngineProvider로 감싼다.
 */
export function GameScene() {
  const tier = detectPerformanceTier();
  const preset = PerformancePresets[tier];

  return (
    <PhysicsEngineProvider
      config={{
        gravity: { x: 0, y: 1.2 },
        maxBodies: preset.maxBodies,
        enableSleeping: preset.enableSleeping,
        fixedTimestep: preset.fixedTimestep,
        maxSubSteps: preset.maxSubSteps,
      }}
    >
      <GameSceneInner />
    </PhysicsEngineProvider>
  );
}
```

### 파일 구조 요약

```
src/
├── physics/
│   ├── types.ts                    # 타입 정의, 상수
│   ├── PhysicsEngineProvider.tsx    # React Context + 엔진 관리
│   ├── usePhysics.ts               # 게임 물리 통합 hook
│   ├── useFixedTimestep.ts         # 고정 타임스텝 hook
│   ├── usePhysicsBody.ts           # 바디 위치 추적 hook
│   ├── useCollisionHandler.ts      # 충돌 핸들러 hook
│   ├── useOffscreenCulling.ts      # 오프스크린 컬링 hook
│   ├── usePhysicsStats.ts          # 성능 통계 hook
│   ├── collisionConfig.ts          # 충돌 카테고리/필터
│   ├── interpolate.ts              # 위치 보간 유틸
│   ├── performanceConfig.ts        # 성능 프리셋
│   ├── StairPool.ts                # 계단 오브젝트 풀
│   ├── createSensor.ts             # 센서 바디 팩토리
│   ├── DebugRenderer.tsx           # Matter.Render 디버그
│   ├── CustomDebugRenderer.tsx     # 커스텀 와이어프레임
│   ├── DebugToggle.tsx             # 디버그 토글 UI
│   └── bodies/
│       ├── stairBody.ts            # 계단 바디 팩토리
│       └── characterBody.ts        # 캐릭터 바디 팩토리
├── game/
│   ├── GameScene.tsx               # 게임 씬 (통합)
│   └── useGameCollisions.ts        # 게임 충돌 로직
└── ...
```

---

## 의존성 설치

```bash
bun add matter-js
bun add -d @types/matter-js
```

## 주요 고려사항 체크리스트

- [ ] Matter.js 버전 고정 (`0.19.x` 또는 `0.20.x`)
- [ ] 엔진 초기화는 컴포넌트 마운트 시 한 번만 수행
- [ ] 매 프레임 `setState` 호출 금지 - `useRef` + DOM 직접 조작
- [ ] 고정 타임스텝(60Hz) + accumulator 패턴 적용
- [ ] 오브젝트 풀링으로 GC 부담 최소화
- [ ] Sleeping 활성화로 비활성 바디 연산 절약
- [ ] 오프스크린 바디 자동 컬링
- [ ] 디바이스 성능에 따른 프리셋 자동 선택
- [ ] DEV 모드에서만 디버그 렌더러 활성화
- [ ] 충돌 카테고리/마스크로 불필요한 충돌 검사 제거
- [ ] 컴포넌트 언마운트 시 엔진/월드/이벤트 완전 정리
