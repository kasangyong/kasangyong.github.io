---
title: "OpenClaw 기여기: 이전 PR이 놓친 인자 하나"
date: 2026-11-23 10:00:00 +0900
categories: [Open Source, TypeScript]
tags: [openclaw, open-source, typescript, vitest, regression-test, bug-fix]
image:
  path: /assets/img/posts/openclaw-doctor-fix/thumb-poster.jpg
  alt: 세 가지 설정이 서로 다른 점검 결과를 내는 OpenClaw 기여기 소개 애니메이션
---

GitHub 스타 39만 개가 넘는 오픈소스 에이전트 OpenClaw에 버그 수정 PR(#84424)을 냈고, 2026년 7월 8일에 머지됐다. 버그 탐색부터 수정, 테스트, PR 작성까지 Claude Code로 진행했다.

찾은 버그는 진단 명령 `openclaw doctor`가 사용자의 설정을 무시하고 엉뚱한 기준으로 경고하던 문제다. 에러도 크래시도 없이 숫자만 틀리는 버그라, 이 문제를 만든 PR의 리뷰와 CI도 그대로 통과했다. 이 글은 버그 자체와 함께, AI로 만든 수정이 큰 프로젝트에 머지되기까지 어떤 증거를 갖췄는지를 정리한다.

| 항목 | 값 |
|---|---|
| 저장소 | [openclaw/openclaw](https://github.com/openclaw/openclaw) |
| PR | [#84424 fix(doctor): honor per-agent bootstrap profile in size check](https://github.com/openclaw/openclaw/pull/84424) |
| 기간 | 2026-05-20 제출, 2026-07-08 머지 (squash) |
| 내 커밋 | 2개, 파일 4개 (소스 2, 테스트 2) |
| 메인테이너 커밋 | 1개, 안내 문구 한 줄 보완 |
| 검증 | 수정 전후 출력 비교 3개 시나리오, 수정 전 코드에서 실패하는 회귀 테스트 |
| 도구 | Claude Code (탐색, 수정, 테스트, PR 작성). PR 본문 첫 줄에 AI 사용을 밝혔다 |
| 검증 환경 | Windows 11 + Node 24 (PR), Linux (메인테이너) |

{% include embed/video.html src='/assets/img/posts/openclaw-doctor-fix/intro.mp4' poster='/assets/img/posts/openclaw-doctor-fix/intro-poster.jpg' title='OpenClaw 기여기 소개: 버그를 찾아 고치고 머지되기까지' %}

## 1. bootstrap 파일과 글자 수 예산

OpenClaw 에이전트는 세션을 시작할 때 작업 폴더의 `AGENTS.md` 같은 파일을 읽어 컨텍스트에 넣는다. 이 파일들을 bootstrap 파일이라고 부른다. 파일이 너무 길면 컨텍스트를 다 먹으니 상한이 있다.

| 설정 | 뜻 |
|---|---|
| `bootstrapMaxChars` | 파일 하나에 넣을 최대 글자 수 |
| `bootstrapTotalMaxChars` | 모든 파일을 합친 최대 글자 수 |

원래 이 값은 `agents.defaults` 아래 전역으로 하나만 둘 수 있었다. PR #69966이 `agents.list[]` 아래에 에이전트마다 따로 정하는 기능을 넣었다. 그 PR은 실행기, CLI, 대화 압축, `/context` 진단까지 에이전트 ID를 넘기도록 고쳤다.

`openclaw doctor`는 설정과 작업 폴더를 점검해 문제를 알려 주는 명령이다. bootstrap 파일이 상한을 넘으면 "몇 % 잘린다, 어느 설정을 조정하라"고 안내한다. #69966은 이 경로를 빠뜨렸다.

## 2. 버그: 에이전트 ID가 넘어가지 않았다

수정 전 `noteBootstrapFileSize`는 이랬다.

```ts
const workspaceDir = resolveAgentWorkspaceDir(cfg, resolveDefaultAgentId(cfg));
const bootstrapMaxChars = resolveBootstrapMaxChars(cfg);
const bootstrapTotalMaxChars = resolveBootstrapTotalMaxChars(cfg);
const { bootstrapFiles, contextFiles } = await resolveBootstrapContextForRun({
  workspaceDir,
  config: cfg,
});
```

에이전트 ID를 구해서 작업 폴더를 찾는 데만 쓰고, 정작 예산을 구하는 세 함수에는 넘기지 않는다. 세 함수는 에이전트 ID가 `undefined`면 조용히 `agents.defaults` 값으로 돌아간다. 그래서 에러가 나지 않는다. 에이전트별로 상한을 다르게 정한 사용자에게 doctor가 전역 기준으로 계산한 퍼센트와, 엉뚱한 설정을 고치라는 안내를 보여 줄 뿐이다.

기존 테스트도 이걸 못 잡았다. 세 함수를 목(mock)으로 바꿔 두고 화면에 찍히는 문자열만 확인했기 때문이다. 에이전트 ID를 넘기든 안 넘기든 목은 같은 값을 돌려주니 출력도 같았다.

고친 코드는 에이전트 ID를 한 번 구해서 세 곳에 모두 넘긴다.

```ts
const defaultAgentId = resolveDefaultAgentId(cfg);
const workspaceDir = resolveAgentWorkspaceDir(cfg, defaultAgentId);
const bootstrapMaxChars = resolveBootstrapMaxChars(cfg, defaultAgentId);
const bootstrapTotalMaxChars = resolveBootstrapTotalMaxChars(cfg, defaultAgentId);
const { bootstrapFiles, contextFiles } = await resolveBootstrapContextForRun({
  workspaceDir,
  config: cfg,
  agentId: defaultAgentId,
});
```

안내 문구도 에이전트별 설정(`agents.list[].bootstrapMaxChars`)을 먼저 가리키고, 전역 값은 그 설정이 없을 때 쓰이는 값으로 적도록 바꿨다.

## 3. 수정 전후 비교

OpenClaw는 외부 기여자에게 "실제 동작 증명"을 요구한다. 단위 테스트가 통과한다는 것만으로는 받지 않는다. 그래서 25,000자짜리 `AGENTS.md`를 둔 임시 작업 폴더를 만들고, 전역 상한을 20,000자로 둔 채 에이전트별 상한만 바꿔 가며 doctor가 실제로 쓰는 함수를 돌렸다.

| 시나리오 | 에이전트별 상한 | 수정 전 | 수정 후 |
|---|---|---|---|
| A. 전역 설정만 | 없음 | 20% 잘림 (20,000자 기준) | 20% 잘림 (변화 없음) |
| B. 에이전트 상한이 더 큼 | 40,000자 | 20% 잘림 (틀린 경고) | 경고 없음 |
| C. 에이전트 상한이 더 작음 | 10,000자 | 20% 잘림 (틀린 경고) | 60% 잘림 (10,000자 기준) |

수정 전에는 세 시나리오가 글자 하나 다르지 않은 같은 경고를 냈다. B는 파일이 실제 예산 안에 들어가는데 잘린다고 경고했고, C는 실제로는 60%가 잘리는데 20%라고 알렸다. 수정 후에는 B와 C가 실제 런타임이 쓰는 예산을 따른다. 전역 설정만 쓰는 A는 그대로라서 기존 사용자에게는 바뀌는 게 없다.

## 4. 회귀 테스트: 출력이 아니라 인자를 본다

버그의 정체가 "인자 하나를 빠뜨렸다"였으니 테스트도 그걸 확인하게 짰다. 기본 에이전트 ID를 `custom-agent`로 바꿔 놓고, 세 함수가 그 ID를 받았는지를 확인한다.

```ts
it("threads the default agent id through bootstrap size resolution", async () => {
  resolveDefaultAgentId.mockReturnValueOnce("custom-agent");
  // ...
  await noteBootstrapFileSize({} as OpenClawConfig);
  expect(resolveBootstrapMaxChars).toHaveBeenCalledWith(expect.anything(), "custom-agent");
  expect(resolveBootstrapTotalMaxChars).toHaveBeenCalledWith(expect.anything(), "custom-agent");
  expect(resolveBootstrapContextForRun).toHaveBeenCalledWith(
    expect.objectContaining({ agentId: "custom-agent" }),
  );
});
```

이 테스트가 버그를 실제로 잡는지도 확인했다. 소스만 `main` 버전으로 되돌리고 돌리면 3개 중 2개가 실패하고, 수정본에서는 3개 모두 통과한다. 포맷 검사, `oxlint`, 타입 검사(`tsgo:core`)도 통과시킨 뒤 제출했다.

## 5. 같은 실수가 한 곳 더

doctor에는 사람이 읽는 안내문을 찍는 경로 말고, 점검 결과를 구조화된 데이터로 돌려주는 상태 점검(`doctor-core-checks.ts`)도 있다. 여기도 같은 세 함수를 에이전트 ID 없이 부르고 있었다. 같은 날 두 번째 커밋으로 이 경로도 고쳤다.

이쪽 테스트는 목 없이 실제 파일로 확인한다. 15,000자 `AGENTS.md`는 전역 상한 20,000자 안에는 들어가지만, 기본 에이전트의 상한 10,000자는 넘는다. 수정 후에는 이 경우에 경고가 나오고, 고치는 방법으로 `agents.list[].bootstrapMaxChars`를 안내한다.

## 6. 머지까지

PR 양식은 길다. 원인 분석, 회귀 테스트 계획, 보안 영향, 호환성, 그리고 "검증하지 않은 것"까지 적어야 한다. 이 중 마지막 항목에는 두 가지를 적었다.

- doctor는 기본 에이전트의 작업 폴더만 본다. 다른 에이전트까지 점검하는 건 이 PR 범위 밖이다.
- 상태 폴더까지 다 갖춘 환경에서 `openclaw doctor` CLI를 직접 돌리지는 않았다. doctor 흐름이 부르는 같은 함수를 직접 호출해 확인했다.

머지는 제출 7주 뒤였다. 리뷰 봇이 변경을 요약하고 "메인테이너 리뷰 필요"로 분류한 뒤, 메인테이너가 최신 `main`에 리베이스하면서 전체 예산 안내 문구가 빠지지 않게 한 줄을 보완했다. PR에 확인하지 않았다고 적은 CLI 실행도 메인테이너가 Linux에서 채웠다. 15,000자 파일과 에이전트 상한 10,000자로 실제 `openclaw doctor --non-interactive`를 돌려 에이전트별 상한이 적용되는 것을 보였고, CI는 69개 검사를 통과했다. 그 뒤 squash로 머지됐다.

## 7. AI로 만든 PR이 머지되려면

큰 오픈소스는 검증되지 않은 AI PR을 반기지 않는다. OpenClaw가 외부 기여자에게 실제 동작 증명을 따로 요구하는 것도 그래서다. 이 PR이 갖춘 것을 다시 모으면 다섯 가지다.

| 갖춘 것 | 이 PR에서 |
|---|---|
| 범위를 하나로 좁힘 | 에이전트 ID 전달과 안내 문구만 바꿨다. 예산 계산, 경고 기준값, 이미 맞게 고쳐진 다른 호출 지점은 건드리지 않았다고 PR에 적었다 |
| 고쳤다는 증거 | 소스를 `main`으로 되돌리면 실패하는 회귀 테스트 |
| 사용자가 보는 결과 | 단위 테스트 대신 doctor 출력을 수정 전후로 나란히 놓은 3개 시나리오 |
| 확인하지 않은 것 | 다른 에이전트 점검, 실제 CLI 실행을 범위 밖이나 미검증으로 밝혔다. 뒤의 것은 메인테이너가 Linux에서 채웠다 |
| AI 사용 공개 | PR 본문 첫 줄 |

메인테이너의 머지 코멘트에는 PR이 낸 Windows 증명과 메인테이너가 돌린 Linux 증명이 나란히 적혀 있다.
