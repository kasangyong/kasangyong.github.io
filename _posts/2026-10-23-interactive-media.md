---
title: "손짓과 스크롤로 움직이는 웹 인터랙션 17가지"
date: 2026-10-23 12:00:00 +0900
categories: [Web, Graphics]
tags: [webgl2, glsl, threejs, mediapipe, web-audio, typescript, intersectionobserver]
math: true
image:
  path: /assets/img/posts/interactive-media/thumb-poster.jpg
  alt: 비스듬히 누운 인터랙티브 미디어 페이지 위로 화면 틀이 오르내리고, 틀 안 데모만 움직이는 모습
---

포인터, 스크롤, 소리, 카메라와 센서를 입력으로 받는 웹 인터랙션 기법 17가지를 스크롤 한 페이지에 데모로 모았다. 각 데모는 페이지 안에서 바로 돌다가 화면 밖으로 나가면 알아서 멈춘다.

**라이브 데모: [kasangyong.github.io/interactive-media](https://kasangyong.github.io/interactive-media/)** (데스크톱 Chrome / Edge 권장)

| 항목 | 값 |
|---|---|
| 데모 | 17개, 4개 챕터 + 인터랙션 역사 가로 타임라인 |
| 동시 활성 WebGL 데모 | 최대 3개 |
| 유체 시뮬레이션 | 속도장 짧은 변 128, 잉크 768, 압력 야코비 20회 |
| 손 인식 | MediaPipe Hand Landmarker, 두 손 21관절, GPU 실패 시 CPU |
| 사운드 구체 | 정점 319,740 → 65,340, 오디오 섹션 74fps → 120fps(주사율 상한) |
| 스택 | TypeScript strict, Three.js r186, WebGL2, Web Audio, Vite 8 |
| 코드 / 테스트 | `src` TypeScript 5,167줄, Vitest 20개, Playwright 스모크 3개 × 데스크톱·모바일 |

{% include embed/video.html src='/assets/img/posts/interactive-media/intro.mp4' poster='/assets/img/posts/interactive-media/intro-poster.jpg' title='인터랙티브 미디어 소개: 커서가 유체를 휘젓고 스크롤이 3D 카메라를 밀고, 화면 밖으로 나간 데모는 멈추는 한 페이지' %}

## 1. 데모 17개가 한 페이지에서 버티는 구조

{% include embed/video.html src='/assets/img/posts/interactive-media/explainer.mp4' poster='/assets/img/posts/interactive-media/explainer-poster.jpg' title='스크롤로 near 범위에 들어온 데모가 시작되고, 포인터 입력이 한 프레임의 패스를 지나 소용돌이가 되며, 멀어지면 pause와 unmount로 자원을 돌려주는 생명주기' loop=true %}

데모가 한꺼번에 돌면 GPU가 버티지 못한다. 브라우저는 WebGL 컨텍스트 수에 상한을 두고, 카메라·마이크·소리는 사용자 동작이 있어야 시작할 수 있다. 이걸 데모마다 따로 처리하는 대신 모든 데모가 메서드 세 개짜리 계약을 따르게 하고, 생명주기는 `DemoHost` 하나에 맡겼다.

```ts
export interface Demo {
  pause(): void;
  resume(): void;
  unmount(): void;
}

export interface DemoFactory {
  create(container: HTMLElement): Demo | Promise<Demo>;
}
```

데모 하나는 `meta.ts`(제목, 설명, 필요 권한)와 `index.ts`(`create`)가 든 폴더 하나로 이뤄진다. 설명 텍스트는 바로 그리고, 무거운 `index.ts`는 화면에 가까워졌을 때 `import()`로 받아 온다. 레지스트리에 한 줄만 넣으면 해당 챕터에 블록이 생긴다.

### 두 개의 IntersectionObserver

| 관찰자 | rootMargin | 들어오면 | 나가면 |
|---|---|---|---|
| near | `15% 0px` | 로드 후 시작, 또는 resume | pause |
| far | `150% 0px` | | unmount (GPU 버퍼, 노드, 스트림 해제) |

pause는 렌더 루프와 오디오, 카메라 트랙만 멈추고 상태는 그대로 들고 있다. 자원을 돌려주는 unmount는 화면 1.5개 이상 멀어졌을 때만 한다. 두 경계를 이렇게 띄워 두면 스크롤을 조금 되돌렸을 때 다시 로드할 일이 없다.

상태는 `idle → (gated) → loading → active ⇄ paused → idle`로 움직인다. 로드나 생성이 실패하면 `failed`로 넘어가 데모 자리에 안내문을 그리고, WebGL을 못 쓰는 경우만 빼고 "다시 시도" 버튼을 붙인다.

### 로딩 중에 지나쳐 버리는 경우

동적 import와 `create`는 비동기다. 받는 사이에 사용자가 스크롤로 지나가 버리면 이미 unmount된 슬롯에 데모가 뒤늦게 붙는다. 그래서 슬롯마다 토큰을 두고 `unload`가 토큰을 올리게 했다. `await`에서 돌아왔는데 토큰이 바뀌어 있으면 결과를 버린다.

```ts
private async start(slot: Slot): Promise<void> {
  const token = ++slot.token;
  slot.state = 'loading';
  // ... 마운트 요소 생성 생략
  if (this.isGL(slot)) this.enforceGLLimit(slot);

  let demo: Demo;
  try {
    const factory = await this.registry[slot.id].load();
    if (token !== slot.token) return;
    demo = await factory.create(mountEl);
  } catch (err) {
    if (token !== slot.token) return;
    // ... failed 처리 생략
    return;
  }

  if (token !== slot.token) {
    demo.unmount();
    return;
  }
  slot.demo = demo;
  if (slot.near) {
    slot.state = 'active';
  } else {
    demo.pause();
    slot.state = 'paused';
  }
}
```

토큰은 `create`가 끝난 뒤에 한 번 더 확인한다. 이 시점의 데모는 GPU 자원과 카메라 스트림을 이미 잡고 있어서, 그냥 버리면 자원이 새고 `unmount()`를 꼭 불러 줘야 한다. 이 경로는 단위 테스트 `unmount requested while loading → demo unmounted on resolve, not active`로 고정해 두었다.

### WebGL 컨텍스트 상한 3개

`requires: ['webgl']`인 데모는 loading, active, paused 상태일 때 컨텍스트를 하나씩 잡고 있는 것으로 센다. 새 GL 데모를 시작하기 전에 이 수가 3개를 넘으면 화면에서 가장 먼 것부터 해제한다. 상한은 DemoHost가 관리하는 데모에만 적용되고, 히어로 캔버스는 여기에 들어가지 않는다.

```ts
private enforceGLLimit(keep: Slot): void {
  const gl = this.glSlots();
  while (gl.length > this.maxActiveGL) {
    let farthest: Slot | undefined;
    let max = -Infinity;
    for (const s of gl) {
      if (s === keep) continue;
      // 화면 근처(near) 슬롯은 최후의 수단. 해제하면 다시 들어오기 전까지 빈 채로 보인다
      const d = distanceToViewport(s.el) - (s.near ? 1e6 : 0);
      if (d > max) {
        max = d;
        farthest = s;
      }
    }
    if (!farthest) break;
    this.unload(farthest);
    // IO 는 상태 변화 때만 알리므로, 다시 들어올 때 재로드되도록 near 해제
    farthest.near = false;
    farthest.el.classList.remove('is-near');
    gl.splice(gl.indexOf(farthest), 1);
  }
}
```

처음에는 뷰포트까지의 거리만 봤다. 이러면 화면 근처(near 범위 안)에 있는 데모도 뽑힐 수 있고, 해제된 데모는 다시 들어오기 전까지 사용자 눈앞에서 빈 칸으로 남는다. 그래서 near 슬롯에 `1e6`만큼 벌점을 줘서 마지막까지 버티게 했다. 기준값도 `-1`에서 `-Infinity`로 바꿨는데, 남은 후보가 모두 near이면 점수가 전부 음수가 되기 때문이다. `-1`에서 시작하면 아무도 뽑히지 않고 상한을 넘긴 채로 끝난다.

해제한 뒤 `near`를 직접 끄는 줄도 빠지면 안 된다. IntersectionObserver는 교차 상태가 바뀔 때만 콜백을 부르기 때문에, 화면 근처에 있던 슬롯을 강제로 내려 버리면 다시 들어와도 `enter`가 오지 않는다. 컨텍스트는 Three.js에서는 `forceContextLoss()`로, 원시 WebGL2에서는 `WEBGL_lose_context`로 바로 돌려준다.

### 권한 게이트

카메라·마이크·모션·소리가 필요한 데모는 버튼을 눌러야 시작한다. 한 번 허용한 요구사항은 `granted`에 넣어 두고, 같은 요구사항만 기다리던 다른 데모의 게이트도 함께 푼다. 예를 들어 "소리 켜기"를 한 번 누르면 사운드 챕터의 다른 데모는 버튼 없이 시작된다. 클릭 핸들러 안에서 `onUnlock`을 부르므로 `AudioContext.resume()`이 사용자 제스처 안에서 처리되고, iOS 모션 권한 요청도 이 자리에서 미리 띄운다.

## 2. Stable Fluids를 프래그먼트 셰이더로 돌린 유체 시뮬레이션

![커서를 따라 소용돌이치는 유체](/assets/img/posts/interactive-media/fluid.gif)
_커서 속도가 속도장에 힘으로, 색이 잉크로 들어간다_

Jos Stam의 Stable Fluids를 WebGL2 프래그먼트 셰이더로 구현했고, 구조는 Pavel Dobryakov의 WebGL-Fluid-Simulation(MIT)을 참고해 단순하게 줄였다. 격자 한 칸이 텍스처의 텍셀 하나에 대응하고, 셰이더 패스 하나가 격자 전체에 연산 하나를 적용한다.

비압축성 나비에-스토크스 방정식에서 점성항을 뺀 형태를 푼다.

$$
\frac{\partial \mathbf{u}}{\partial t} = -(\mathbf{u}\cdot\nabla)\mathbf{u} - \nabla p + \mathbf{f}, \qquad \nabla\cdot\mathbf{u} = 0
$$

### 텍스처 구성

| 필드 | 형식 | 해상도(16:9) | 필터 |
|---|---|---|---|
| 속도 `velocity` | RG16F, 더블 버퍼 | 228 × 128 | LINEAR |
| 압력 `pressure` | R16F, 더블 버퍼 | 228 × 128 | NEAREST |
| 발산, 회전 | R16F | 228 × 128 | NEAREST |
| 잉크 `dye` | RGBA16F, 더블 버퍼 | 1365 × 768 | LINEAR |

짧은 변은 `SIM_RES = 128`, `DYE_RES = 768`로 두고 긴 변은 화면비에 맞춰 늘린다. 물리는 거친 격자에서 풀고, 눈에 보이는 잉크만 6배 촘촘한 격자에서 옮기는 식이다. 잉크 쪽에서 속도장을 선형 보간으로 읽기 때문에 두 해상도가 달라도 상관없다.

속도에는 음수가 들어가서 8비트 렌더 타깃에는 담기지 않는다. `EXT_color_buffer_float`나 `EXT_color_buffer_half_float`가 없는 환경에서는 `WebGLUnavailableError`를 던지고, `DemoHost`가 이를 "WebGL을 사용할 수 없다"는 안내로 바꿔 그린다. 또 한 패스 안에서 같은 텍스처를 읽으면서 쓸 수는 없으니, 읽기용과 쓰기용 FBO 두 장을 두고 패스마다 `swap()`한다.

### 한 프레임의 패스 순서

| 순서 | 패스 | 하는 일 |
|---|---|---|
| 1 | curl | 속도장의 회전 $\omega$ |
| 2 | vorticity | 소용돌이 보강 힘 더하기 |
| 3 | divergence | $\nabla\cdot\mathbf{u}$ |
| 4 | clear | 이전 압력에 0.8을 곱해 초기값으로 |
| 5 | pressure × 20 | 야코비 반복으로 $\nabla^2 p = \nabla\cdot\mathbf{u}$ |
| 6 | gradient | $\mathbf{u} \leftarrow \mathbf{u} - \nabla p$ |
| 7 | advection × 2 | 속도, 잉크를 속도장 따라 이류 |
| 8 | display | 잉크를 화면에 |

Stam의 기본 순서와 달리 이 구현은 투영을 이류보다 먼저 한다.

그리기 호출은 한 프레임에 28번이고 그중 26번이 228 × 128 격자에서 돈다. 커서 입력(splat)은 이 루프와 별개로 `pointermove` 핸들러에서 속도와 잉크 텍스처에 바로 더한다.

### 이류는 거꾸로 따라가서 읽는다

```glsl
uniform sampler2D uVelocity; uniform sampler2D uSource; uniform vec2 texel; uniform float dt; uniform float dissipation;
void main(){
  vec2 coord = vUv - dt * texture(uVelocity, vUv).xy * texel;
  outColor = texture(uSource, coord) / (1.0 + dissipation * dt);
}
```

semi-Lagrangian 방식으로, 각 텍셀이 속도를 거꾸로 따라가며 "dt 전에 나는 어디 있었나"를 찾고 그 자리의 값을 선형 보간으로 읽어 온다.

$$
q^{n+1}(\mathbf{x}) = \frac{q^{n}(\mathbf{x} - \Delta t\,\mathbf{u}(\mathbf{x}))}{1 + d\,\Delta t}
$$

값을 앞으로 밀어 보내지 않고 기존 값 사이를 보간만 하기 때문에 새 값이 이웃 값의 범위를 벗어나지 않고, dt가 커도 발산하지 않는다. Stable Fluids라는 이름의 "stable"이 이 성질에서 나왔다. 이와 별도로 렌더 루프에서 dt를 최대 0.05초로 잘라 두어서, 프레임이 늦게 와도 한 스텝에 크게 밀리지 않는다.

분모의 $1 + d\,\Delta t$가 감쇠 항이다. 60fps에서 1초 뒤에 남는 양을 계산해 보면 속도($d = 0.25$)는 $(1 + 0.25/60)^{-60} \approx 0.78$, 잉크($d = 0.9$)는 $(1 + 0.9/60)^{-60} \approx 0.41$이다. 그래서 소용돌이는 오래 돌고, 색은 그보다 빨리 옅어진다.

### 압력 투영으로 발산 지우기

이류와 힘을 거친 속도장은 보통 발산이 0이 아니어서, 어떤 칸에는 유체가 모이고 어떤 칸에서는 빠져나간다. 헬름홀츠 분해로 속도장을 "발산 없는 부분 + 기울기"로 나눈 뒤 기울기를 빼면 비압축 조건이 맞는다. 그 기울기가 압력이고, 압력은 포아송 방정식을 풀어 구한다.

$$
\nabla^2 p = \nabla\cdot\mathbf{u} \;\;\Rightarrow\;\; p_{i,j} \leftarrow \frac{p_{i-1,j} + p_{i+1,j} + p_{i,j-1} + p_{i,j+1} - (\nabla\cdot\mathbf{u})_{i,j}}{4}
$$

```glsl
// divergence
float L = texture(uVelocity, vL).x; float R = texture(uVelocity, vR).x;
float T = texture(uVelocity, vT).y; float B = texture(uVelocity, vB).y;
vec2 C = texture(uVelocity, vUv).xy;
if (vL.x < 0.0) L = -C.x; if (vR.x > 1.0) R = -C.x;
if (vT.y > 1.0) T = -C.y; if (vB.y < 0.0) B = -C.y;
outColor = vec4(0.5 * (R - L + T - B), 0.0, 0.0, 1.0);

// pressure (야코비 1회)
outColor = vec4((L + R + B + T - div) * 0.25, 0.0, 0.0, 1.0);

// gradient
vec2 vel = texture(uVelocity, vUv).xy - vec2(R - L, T - B);
```

야코비 반복에서는 한 칸의 새 값이 이웃의 이전 값에만 의존하므로, 텍셀마다 따로 계산하는 프래그먼트 셰이더와 잘 맞는다. 다만 수렴이 느려 20회로는 완전히 풀리지 않는다. 그래서 압력을 매 프레임 0에서 시작하는 대신 이전 프레임 압력에 0.8을 곱해 초기값으로 쓴다. 유체는 프레임 사이에 크게 변하지 않으니 이전 해에서 출발하는 편이 낫다.

발산 패스에서는 경계를 처리할 때 벽 밖의 속도를 안쪽 속도의 반대 부호로 둔다. 벽을 막힌 면으로 보고 발산을 계산하니, 벽 쪽으로 가는 흐름은 압력으로 되밀린다.

중심 차분의 $\tfrac{1}{2}$은 발산 패스에만 있고 기울기 패스에는 없다. 결국 기울기를 중심 차분의 2배로 빼는 셈이다. 참고 구현(Dobryakov)과 같은 스케일이며, 따로 맞춰 보지는 않았다.

### 소용돌이 보강

이류 때마다 선형 보간을 거치면서 수치 확산이 생기고, 격자 몇 칸 크기의 작은 소용돌이는 금방 뭉개진다. 이걸 vorticity confinement로 보완한다. 회전이 강한 곳에 회전을 키우는 힘을 다시 넣어 주는 방식이다.

$$
\omega = \frac{\partial v}{\partial x} - \frac{\partial u}{\partial y}, \qquad
\mathbf{N} = \frac{\nabla|\omega|}{\lVert\nabla|\omega|\rVert}, \qquad
\mathbf{f} = \varepsilon\,(N_y\,\omega,\; -N_x\,\omega)
$$

```glsl
float L = texture(uCurl, vL).x; float R = texture(uCurl, vR).x;
float T = texture(uCurl, vT).x; float B = texture(uCurl, vB).x;
float C = texture(uCurl, vUv).x;
vec2 force = 0.5 * vec2(abs(T) - abs(B), abs(R) - abs(L));
force /= length(force) + 0.0001;
force *= curl * C; force.y *= -1.0;
vec2 vel = texture(uVelocity, vUv).xy + force * dt;
```

2차원에서 $\mathbf{N} \times \omega\hat{z}$는 `force`의 x 성분에 $\partial|\omega|/\partial y$, y 성분에 $\partial|\omega|/\partial x$를 넣고 y에 $-1$을 곱해서 계산한다. 세기 $\varepsilon$은 `CURL = 28`이다.

### 입력과 표시

커서가 움직이면 직전 위치와의 차이에 6000을 곱해 속도로 넣고, 시안과 바이올렛 사이의 색을 잉크로 넣는다. 둘 다 가우시안 모양으로 더한다. 입력이 2.5초 동안 없으면 1.1초마다 임의 위치에 한 번씩 자동으로 뿌려서 화면이 멈춘 것처럼 보이지 않게 했다.

```glsl
vec3 raw = max(texture(uTexture, vUv).rgb, 0.0);
// 노출 톤매핑: 옅은 잉크도 보이고 진한 곳은 하얗게 타지 않게
vec3 c = 1.0 - exp(-raw * 4.0);
// ... 밀도 기울기로 얕은 음영을 넣는 부분 생략
// 비-premultiplied 캔버스에서 가산 합성처럼 보이도록 색을 알파로 나눈다
float a = clamp(max(c.r, max(c.g, c.b)) * 1.15, 0.0, 1.0);
outColor = vec4(min(c / max(a, 0.001), 1.0), a);
```

잉크는 계속 쌓이므로 값이 1을 넘는다. 이걸 $1 - e^{-4x}$로 눌러 옅은 잉크는 살리고, 진한 곳이 하얗게 날아가는 건 막았다. 캔버스는 투명해서 뒤쪽 섹션의 배경색이 비쳐 보여야 한다. 그래서 밝기를 알파로 쓰고 색을 알파로 나눠, 어두운 배경 위에 빛이 더해지는 것처럼 보이게 했다.

## 3. 두 손이 각자 다른 값을 쥐는 손 인식

![두 손 제스처로 디더링과 회전을 조절하는 조형물](/assets/img/posts/interactive-media/hand-brutalism.gif)
_왼손 엄지·검지 간격이 디더링, 오른손 손가락 선의 기울기가 회전. 캡처용 손 영상은 로컬 이미지 모델로 생성해 합성한 입력이다_

손 인식을 쓰는 데모는 두 개다. "손 추적"에서는 검지 끝으로 궤적을 그리고, 엄지와 검지를 집으면 불꽃이 튄다. "두 손으로 조각하는 브루탈리즘"은 Genuary의 TouchDesigner 작업에서 아이디어를 얻은 데모로, 두 손이 각각 다른 파라미터를 맡는다. 영상은 브라우저 안에서만 처리하고 기기 밖으로 내보내지 않는다.

### 파이프라인

| 단계 | 처리 |
|---|---|
| 카메라 | `getUserMedia`, 1280×720 이상적, 버튼을 눌러야 시작 |
| 모델 로드 | Hand Landmarker(float16), WASM, GPU 델리게이트 → 실패 시 CPU |
| 검출 | 새 영상 프레임일 때만 `detectForVideo`, 단조 증가 타임스탬프 |
| 좌표 | 정규화 좌표를 거울 반전, cover 맞춤으로 화면 좌표에 |
| 제스처 | 손 크기로 정규화한 거리, 각도 변화량 |
| 평활 | 목표값을 향한 지수 평활 |
| 출력 | Three.js 장면 회전, 디더링 후처리 uniform |

로더는 두 데모가 같이 쓴다.

```ts
// 패키지 버전과 반드시 같은 wasm 을 써야 한다
const WASM = 'https://cdn.jsdelivr.net/npm/@mediapipe/tasks-vision@1.0.1/wasm';

/** 손 관절 21개 검출기. GPU 델리게이트가 안 되는 환경이면 CPU 로 한 번 더 시도 */
export async function loadHandLandmarker(numHands = 2): Promise<HandLandmarker> {
  const fileset = await FilesetResolver.forVisionTasks(WASM);
  const make = (delegate: 'GPU' | 'CPU') =>
    HandLandmarker.createFromOptions(fileset, {
      baseOptions: { modelAssetPath: MODEL, delegate },
      runningMode: 'VIDEO',
      numHands,
    });
  return make('GPU').catch(() => make('CPU'));
}

/** detectForVideo 는 타임스탬프가 단조 증가해야 한다 */
export function monotonicClock(): () => number {
  let last = 0;
  return () => (last = Math.max(last + 1, Math.round(performance.now())));
}
```

처음에는 `detectForVideo(video, performance.now())`를 그대로 넘기고 GPU 델리게이트만 시도했다. 그런데 `VIDEO` 모드는 직전 호출보다 큰 타임스탬프를 요구한다. 그래서 밀리초로 반올림한 값이 겹치면 1씩 밀어 엄격하게 증가하도록 고쳤고, CPU 폴백도 이때 같이 넣었다. 그래도 모델이 안 뜨면 손 추적은 `DemoHost`의 "다시 시도" 폴백으로 넘어가고 브루탈리즘은 마우스 조작으로 남는다.

### 같은 프레임을 두 번 추론하지 않는다

```ts
if (video.readyState >= 2 && video.currentTime !== lastVideoTime) {
  lastVideoTime = video.currentTime;
  const r = landmarker.detectForVideo(video, clock());
  hands = r.landmarks.map((hand) => hand.map(map));
}
```

렌더 루프는 `requestAnimationFrame`이라 주사율만큼 도는데, 카메라 프레임은 대개 그보다 드물게 들어온다. 그래서 `video.currentTime`이 바뀐 경우에만 추론하고 그 사이 프레임은 마지막 결과로 그린다. `detectForVideo`는 결과를 동기로 돌려주기 때문에 추론 시간이 그 프레임의 렌더 시간에 고스란히 얹힌다. 같은 영상 프레임을 다시 추론하지 않기만 해도 프레임 시간이 준다.

### 제스처는 손 크기로 나눈다

손이 카메라에 가까울수록 엄지와 검지 사이의 픽셀 거리도 커지기 때문에, 이 거리를 손목(0)에서 가운데 손가락 뿌리(9)까지의 길이로 나눠서 쓴다.

```ts
// 핀치: 엄지 끝(4)과 검지 끝(8) 거리를 손 크기(0→9)로 정규화
const handSize = Math.hypot(pts[0].x - pts[9].x, pts[0].y - pts[9].y) || 1;
const pinch = Math.hypot(pts[4].x - tip.x, pts[4].y - tip.y) / handSize < 0.28;
if (pinch && !pinched[hi]) {
  // 불꽃 60개 생성 (생략)
}
pinched[hi] = pinch;
```

불꽃은 핀치 상태가 바뀌는 순간에만 나오므로, 집은 채로 있어도 매 프레임 터지지는 않는다.

브루탈리즘에서는 같은 정규화 거리를 디더 강도에, 엄지에서 검지로 가는 선의 각도 변화를 회전에 쓴다.

```ts
// 손 개수가 바뀌면 '오른손' 이 다른 손으로 바뀔 수 있으므로 회전 기준 각도를 초기화
if (hands.length !== lastHandCount) lastAngle = null;
lastHandCount = hands.length;
// 화면 왼쪽 손 = 디더, 오른쪽 손 = 회전 (손 라벨 대신 위치로 구분해 혼동을 없앤다)
const sorted = [...hands].sort((a, b) => a[0].x - b[0].x);
const left = sorted.length === 2 ? sorted[0] : sorted[0][0].x < 0.5 ? sorted[0] : null;
const right = sorted.length === 2 ? sorted[1] : sorted[0][0].x >= 0.5 ? sorted[0] : null;
if (left) {
  const size = Math.hypot(left[0].x - left[9].x, left[0].y - left[9].y) || 1;
  const ratio = Math.hypot(left[4].x - left[8].x, left[4].y - left[8].y) / size;
  control.targetDither = Math.min(1, Math.max(0, (ratio - 0.15) / 0.85));
}
if (right) {
  const a = Math.atan2(right[8].y - right[4].y, right[8].x - right[4].x);
  if (lastAngle !== null) control.targetRot += Math.atan2(Math.sin(a - lastAngle), Math.cos(a - lastAngle)) * 1.6;
  lastAngle = a;
} else lastAngle = null;
```

왼손과 오른손은 MediaPipe의 손 라벨 대신 화면에서의 손목 x 좌표로 나눈다. 거울 반전을 거친 좌표에서는 라벨과 화면 위치가 엇갈리기 쉬운데, 위치로 나누면 그런 혼동이 아예 생기지 않는다.

회전에는 절대 각도 대신 프레임 사이 변화량을 누적한다. 덕분에 손을 뗐다가 다시 대도 조형물이 튀지 않고, 손목을 여러 번 돌려 계속 감을 수도 있다. 변화량은 `atan2(sin Δ, cos Δ)`로 $(-\pi, \pi]$ 안에 접는다. 이렇게 해 두면 각도가 $\pi$에서 $-\pi$로 넘어가는 순간 한 바퀴를 거꾸로 도는 일이 없다.

손이 하나에서 둘로 늘면 "오른쪽 손"이 다른 손으로 바뀔 수 있다. 그러면 이전 각도와 새 손의 각도 차이가 그대로 회전으로 들어가 튄다. 그래서 손 개수가 바뀌면 기준 각도를 버리도록 고쳤다.

### 평활은 일부러 늦게

랜드마크 좌표는 프레임마다 조금씩 흔들린다. 그래서 uniform에 바로 넣지 않고 목표값만 갱신하며, 실제 값은 그 목표를 지수적으로 따라가게 했다.

```ts
control.dither += (control.targetDither - control.dither) * Math.min(1, dt * 8);
control.rot += (control.targetRot - control.rot) * Math.min(1, dt * 6);
```

$k\,dt \ll 1$이면 $(1-k\,dt)^{1/dt}\approx e^{-k}$라서, 시간 상수는 프레임레이트와 거의 상관없이 $1/k$초에 가깝다(60fps에서 약 116ms, 158ms). 극한값으로 보면 디더는 125ms, 회전은 167ms 늦게 따라온다. 흔들림을 거르는 대가로 이만큼 지연을 받아들였다. `Math.min(1, ...)`은 안전장치인데, 루프가 dt를 0.05초로 자르기 때문에 실제로 걸릴 일은 없다. 반대로 손 추적 데모의 궤적은 평활 없이 원시 좌표를 그대로 쓰고, 1.4초에 걸쳐 흐려지게만 했다.

### 비동기 초기화 중의 경쟁

카메라 연결과 모델 다운로드에는 수 초가 걸리기도 한다. 그동안 사용자가 스크롤로 지나치면 `pause`가, 더 멀리 가면 `unmount`가 먼저 불린다.

```ts
const lm = await loadHandLandmarker(2);
if (disposed) {
  lm.close();
  return;
}
landmarker = lm;
mode = 'camera';
// 로딩 중 화면을 벗어났다면 카메라를 켜 둔 채 두지 않는다
if (paused) {
  video.pause();
  setStreamEnabled(stream, false);
}
```

`await`마다 `disposed`를 확인하는 코드는 대부분 처음부터 있었다. 리뷰 뒤에 추가한 건, 로딩이 끝난 시점에 이미 일시정지 상태라면 트랙을 꺼서 카메라 표시등이 켜진 채로 남지 않게 하는 처리다. pause에서는 `track.enabled = false`로 트랙을 비활성화하고(브라우저에 따라 표시등도 꺼진다), unmount에서 `track.stop()`과 `landmarker.close()`로 완전히 놓아준다.

### 브루탈리즘의 디더링 후처리

조형물은 판, 기둥, 덩어리 34개를 시드를 고정한 난수로 쌓아 만든 Three.js 장면이다. 장면을 먼저 렌더 타깃에 그린 다음, 전체 화면 사각형 하나에 4×4 Bayer 순서 디더링을 입혀 내보낸다.

```glsl
float bayer4(vec2 p){
  int x = int(mod(p.x, 4.0));
  int y = int(mod(p.y, 4.0));
  float m[16] = float[16](0.,8.,2.,10.,12.,4.,14.,6.,3.,11.,1.,9.,15.,7.,13.,5.);
  return (m[x + y * 4] + 0.5) / 16.0;
}
void main(){
  // 렌더 타깃은 선형 색공간. 화면 출력용 sRGB 로 근사 변환
  vec3 base = pow(texture2D(tScene, vUv).rgb, vec3(1.0 / 2.2));
  vec2 cell = floor(gl_FragCoord.xy / uPix);
  vec3 blockCol = pow(texture2D(tScene, (cell + 0.5) * uPix / uRes).rgb, vec3(1.0 / 2.2));
  float l = dot(blockCol, vec3(0.299, 0.587, 0.114));
  float on = step(bayer4(cell), l);
  // 점은 각 블록 안에서 작은 정사각형으로. 디더가 강할수록 점묘처럼 보인다
  vec2 f = fract(gl_FragCoord.xy / uPix) - 0.5;
  float pixOn = on * step(max(abs(f.x), abs(f.y)), 0.36);
  vec3 dithered = vec3(pixOn) * (0.55 + 0.45 * l);
  gl_FragColor = vec4(mix(base, dithered, uDither), 1.0);
}
```

화면을 `uPix` 크기의 블록으로 나누고, 블록 중심 한 점의 밝기를 Bayer 임계값과 비교해 점을 켜거나 끈다. Bayer 행렬은 임계값이 4×4 안에 고르게 퍼지게 배치돼 있어서 같은 밝기의 넓은 면은 일정한 점 패턴이 된다. 블록 크기를 `Math.round(3 * dpr)`로 잡았기 때문에 고해상도 화면에서도 CSS 픽셀 기준 점 크기는 비슷하다. 왼손으로 조절하는 `uDither`는 원본과 디더 결과를 섞는 비율이다.

카메라를 켜기 전에는 같은 두 값을 마우스로 조절할 수 있다(X는 회전, Y는 디더). Playwright 스모크에서는 Chrome의 가짜 카메라 장치를 붙인 채 버튼을 누르고, 모델을 받아 `INPUT CAMERA`로 바뀔 때까지 에러가 나지 않는지 본다.

## 4. 사운드 반응 구체, 재 보니 정점이 문제였다

![킥과 베이스에 맞춰 일렁이는 검은 구체](/assets/img/posts/interactive-media/audio-sphere.gif)
_저음은 부피, 중음은 주름, 고음은 떨림으로 바뀐다_

브라우저가 직접 연주하는 루프를 `AnalyserNode`(FFT 512)로 분석하고, 그 결과로 구체의 형태를 바꾼다.

### 박자는 오디오 클록으로 예약한다

`setInterval`에서 노트를 바로 울리면 메인 스레드가 바쁠 때마다 박자가 흔들린다. 그래서 룩어헤드 스케줄러가 25ms마다 깨어나, 앞으로 0.12초 안에 울릴 16분음표를 오디오 클록 시각으로 미리 예약하게 했다. 타이머는 0.12초 창 안에서만 깨어나면 조금 늦어도 괜찮다. 실제 발음 시각은 오디오 스레드가 지킨다.

```ts
const tick = () => {
  // 백그라운드 탭 스로틀로 밀렸으면 지난 노트를 몰아 치지 않고 현재로 점프
  if (next < c.currentTime) next = c.currentTime + 0.02;
  while (next < c.currentTime + 0.12) {
    onStep(step, next);
    step++;
    next += spb;
  }
};
```

첫 줄은 나중에 추가했다. 이 줄이 없으면 백그라운드 탭에서 타이머가 스로틀됐다가 돌아올 때 `while`이 그동안 밀린 노트를 한꺼번에 예약해 버려서 소리가 한 번에 터진다. 지금은 밀렸을 때 과거 노트를 버리고 현재 시각으로 건너뛴다.

분석 쪽은 bin 1~6을 저음, 6~40을 중음, 40~120을 고음으로 묶어 평균을 낸다. 세 값은 각각 구체 크기, 변위 진폭, 노이즈 주파수와 시간 속도로 들어간다.

### 74fps에서 120fps로

구체 표면은 버텍스 셰이더에서 3D 심플렉스 노이즈로 밀어낸다. 변위 함수 한 번에 노이즈가 2옥타브 들어가고, 법선을 다시 구하려고 이웃한 두 점에서도 변위를 계산한다.

```glsl
float disp(vec3 p){
  float n = snoise(p * uFreq + vec3(0.0, uTime * 0.25, uTime * 0.1));
  n += 0.5 * snoise(p * uFreq * 2.1 - vec3(uTime * 0.2));
  float bump = smoothstep(0.55, 1.0, dot(normalize(p), uPointer)) * uPointerForce;
  return n * uAmp + bump;
}
void main(){
  vec3 n = normalize(position);
  vec3 t = orthogonal(n);
  vec3 b = normalize(cross(n, t));
  float eps = 0.01;
  vec3 p0 = position + n * disp(position);
  vec3 p1 = position + t * eps; p1 += normalize(p1) * disp(p1);
  vec3 p2 = position + b * eps; p2 += normalize(p2) * disp(p2);
  vec3 dn = normalize(cross(p1 - p0, p2 - p0));
  vDisp = disp(position);
  // ... 이하 생략
}
```

세 점 × 2옥타브라서 정점당 6회가 필요하다. 해석적 법선을 유도하는 대신 유한 차분으로 구한 대가다. 코드만 보면 `vDisp`의 `disp(position)`이 중복돼 8회를 호출하지만, 인자가 p0과 같아서 컴파일러가 합칠 수 있으므로 6회로 셌다.

오디오 섹션에서 사운드 구체와 신시사이저 패드가 같이 돌면 74fps까지 떨어졌고, 원인은 구체의 정점 수였다. Three.js의 `IcosahedronGeometry`는 인덱스가 없는 지오메트리라서 삼각형마다 정점 3개를 따로 가진다. detail이 $d$면 삼각형은 $20(d+1)^2$개다.

| detail | 삼각형 | 정점 | 정점당 노이즈(중복 제외) | 노이즈 호출(중복 제외) |
|---|---|---|---|---|
| 72 | 106,580 | 319,740 | 6 | 1,918,440 |
| 32 | 21,780 | 65,340 | 6 | 392,040 |

detail을 32로 낮추자 정점이 4.9분의 1로 줄었고, 같은 구간이 120fps(주사율 상한)로 돌아왔다.

이 수정을 하면서 화면 전체를 덮는 무거운 장면의 기기 픽셀 비율도 1.5로 묶었다. 현재 이 상한을 쓰는 건 히어로, 사운드 구체, 기기 기울기, 노이즈 필드, 브루탈리즘이다. DPR 2 화면이면 픽셀 수가 $(1.5/2)^2 = 0.5625$배로 약 44% 줄어든다. 다만 이 변경의 fps는 따로 재지 않았다.

## 5. 나머지 데모

| 챕터 | 데모 | 기법 |
|---|---|---|
| Pointer | 커서 파티클 트레일 | 포인터 속도를 입자 밀도와 색으로, Canvas 2D, 오브젝트 풀 |
| Pointer | 자석 버튼 & 물리 드래그 | Pointer Capture, 던진 속도를 이어받는 충돌·탄성 |
| Pointer | 멀티터치 제스처 | 두 손가락의 거리·각도·중점을 변환 하나로, 휠 폴백 |
| Scroll | 스크롤 연동 3D 카메라 | 진행도 0→1을 `CatmullRomCurve3` 위 위치로, `position: sticky` |
| Scroll | 패럴랙스 레이어 | 레이어별 깊이 계수, 포인터 기울기 |
| Scroll | 스크롤텔링 | 진행도 구간별 SVG 도형 전환, 180점 극좌표 보간 |
| Scroll | 텍스트 모핑 | `getImageData`로 글자를 4,200점 샘플링, 점마다 시차 이동 |
| Sound | 신시사이저 패드 | 펜타토닉 4×4, Oscillator + BiquadFilter, ADSR |
| Sound | 마이크 비주얼라이저 | `getUserMedia`, FFT 2048, 로그 스케일 원형 스펙트럼 |
| Sound | 공간 음향 | `PannerNode` HRTF, 평면에서 음원 이동 |
| Sensor | 웹캠 셰이더 필터 | VideoTexture, 아스키·열화상·픽셀·글리치 프래그먼트 셰이더 |
| Sensor | 손 추적 | 3장 참고 |
| Sensor | 기기 기울기 장면 | `DeviceOrientationEvent`, iOS 권한, 데스크톱은 커서 |
| Sensor | 노이즈 필드 & 셀룰러 오토마타 | 심플렉스 흐름장 위 입자 3,200개, 라이프 게임 |

![가로 스크롤 타임라인](/assets/img/posts/interactive-media/timeline.jpg)
_마지막 섹션. 1963년 Sketchpad부터 2020년대 온디바이스 ML까지, 세로 스크롤이 가로 트랙을 넘긴다_

스크롤 연동 데모와 타임라인은 GSAP ScrollTrigger의 pin을 쓰지 않고 `position: sticky`와 진행도 계산으로 만들었다. pin은 고정할 요소를 감싼 뒤 여백을 계산해 끼워 넣는다. 데모를 동적으로 붙였다 뗐다 하는 페이지라서, 레이아웃이 튀지 않도록 이 부분은 CSS에 맡겼다. 섹션 배경은 고정 레이어 하나의 색을 스크롤 위치에 따라 보간하기 때문에 경계 없이 바뀐다.

## 6. 검증

| 종류 | 내용 |
|---|---|
| Vitest | DemoHost 생명주기 7개(로드 중 해제, GL 상한, near 우선 보호, 실패 격리, 게이트 공유 해제 등), 권한 오류 분류 5개, 스테이지 2개, 색 보간 6개 |
| Playwright | 데스크톱(1440×900)·모바일(Pixel 7)에서 페이지 끝까지 반 화면씩 스크롤하며 게이트를 누르고 확인 |
| CI | `main` 푸시 시 lint, typecheck, 단위 테스트, 빌드 후 GitHub Pages 배포 |

스모크 테스트에서 보는 항목은 이렇다.

- `[data-demo]`가 17개다.
- 콘솔 에러와 같은 출처 HTTP 오류가 0이다. CDN 네트워크 실패와 헤드리스 오디오 장치 메시지만 예외로 둔다.
- 스크롤하는 동안 `activeGLCount()`가 한 번도 3을 넘지 않는다.
- `failed`가 된 데모가 없고, 모델 다운로드가 걸리는 손 추적을 빼면 모든 데모가 한 번은 active나 paused가 된다.
- 끝까지 내려간 뒤 첫 데모는 paused나 idle이다.
- 타임라인 섹션 중간으로 스크롤하면 트랙이 실제로 가로로 100px 넘게 이동한다.
- 페이지에 가로 넘침이 없다(`scrollWidth - innerWidth ≤ 0`).

가짜 카메라·마이크 장치는 여러 브라우저가 동시에 열면 서로 막히기 때문에 워커는 1개로 고정했다.

## 7. 한계와 다음 단계

- 프레임레이트를 잰 곳은 사운드 구체 하나뿐이다. 유체(프레임당 28패스)나 손 인식의 추론 시간은 측정하지 않았다.
- 손 인식 추론이 메인 스레드에서 동기로 돌아서 카메라 프레임마다 그 시간만큼 렌더가 밀린다. 다음에는 Worker로 옮기거나 추론 주기를 렌더와 떼어 놓을 생각이다.
- 유체의 압력 야코비 20회는 완전히 수렴하지 않고, 기울기 패스의 스케일도 맞춰 보지 않았다. 둘 다 눈에 보이는 결과만 보고 정했다.
- 반정밀도 float 렌더 타깃이 없는 기기에서는 유체가 돌지 않는다. 8비트 텍스처에 부호를 인코딩하는 폴백은 만들지 않았다.
- Playwright 스모크는 CI에 넣지 않고 로컬에서만 돌린다.

코드: [github.com/kasangyong/interactive-media](https://github.com/kasangyong/interactive-media)
