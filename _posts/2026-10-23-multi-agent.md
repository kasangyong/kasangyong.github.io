---
title: "LLM 리뷰어 대신 컴파일러를: 에이전트 6개가 3라운드 도는 코드 생성 파이프라인"
date: 2026-10-23 11:20:00 +0900
categories: [AI, Agent]
tags: [multi-agent, gemini, llm, prompt-engineering, python, nodejs, esbuild]
image:
  path: /assets/img/posts/multi-agent/pipeline-light.png
  alt: 6개 에이전트가 3라운드를 도는 파이프라인 구조도
---

과제 한 줄을 넣으면 Gemini 기반 에이전트 6개가 순서대로 세 라운드를 돌며 프로젝트 파일을 만들고, 2라운드가 끝난 시점에 실제 도구로 문법과 의존성을 검사해 그 에러를 3라운드에 넘기는 파이썬 시스템을 만들었다.

| 항목 | 값 |
|---|---|
| 에이전트 | 6개 (Idea, Architect, Developer, Designer, Reviewer, Orchestrator) |
| 라운드 / API 호출 | 3라운드 / 18회 |
| 모델 | `gemini-3.6-flash` (초기 버전은 로컬 Ollama `gemma2:2b`) |
| 빌드 검사 | 5종 (package.json 파싱, 존재, `node --check`, `esbuild`, `npm install --package-lock-only`) |
| 컨텍스트 제한 | 에이전트 출력 1건당 4,000자 |
| Round 1 응답 시간 | 193.3초 (실행 예시 1회) |
| 코드 | 파이썬 402줄, 외부 의존성 `google-genai` 1개 |

## 1. 구조: 역할 6개, 파일을 쓰는 건 3개

![파이프라인 구조도](/assets/img/posts/multi-agent/pipeline-light.png){: .light }
![파이프라인 구조도](/assets/img/posts/multi-agent/pipeline-dark.png){: .dark }
_주황 테두리 에이전트의 출력에서만 파일을 뽑는다_

에이전트는 모두 `BaseAgent`를 상속하고, 차이는 이름, 이모지, 시스템 프롬프트뿐이다.

```python
class DeveloperAgent(BaseAgent):
    def __init__(self):
        bt = chr(96) * 3
        super().__init__(
            "Developer Agent",
            "💻",
            f"You write code files. FORMAT: FILE: path\n{bt}language\ncode\n{bt}. In Round 2-3, IMPROVE code based on Reviewer feedback. State improvements. Use className for styles."
        )
```

| 순서 | 에이전트 | 시스템 프롬프트 요지 | 파일 추출 |
|---|---|---|---|
| 1 | Idea | 기능 제안. 2~3라운드에는 새 아이디어 추가 또는 다듬기 | X |
| 2 | Architect | 구조 설계. 2~3라운드에는 피드백 반영해 개선 | X |
| 3 | Developer | `FILE:` 형식으로 코드 작성, 스타일은 `className` | O |
| 4 | Designer | `FILE:` 형식으로 CSS 작성 | O |
| 5 | Reviewer | 전체 출력 검토와 구체적 피드백 | X |
| 6 | Orchestrator | 종합. 3라운드에 `FILE: README.md` 작성 | O |

역할 경계는 프롬프트보다 **파일 추출 필터**에서 강제된다. `FileGenerator`는 세 에이전트 출력만 본다. Reviewer가 리뷰 중에 고친 코드를 `FILE:` 블록으로 붙여도 디스크에는 안 쓰인다.

```python
if agent_name in ['Developer Agent', 'Designer Agent', 'Orchestrator Agent']:
    files = self.extract_files_from_text(str(output))
```

## 2. 상태: 라운드 × 에이전트 딕셔너리 하나

메시지 큐나 에이전트 간 직접 호출은 없다. 상태는 `SharedMemory.data` 하나이고 모양은 `{round: {agent_name: output}}`다.

```python
class SharedMemory:
    MAX_CONTEXT_CHARS = 4000

    # ... 생략: __init__, get_round, _truncate, get_all

    def add(self, round_num, agent_name, output):
        if round_num not in self.data:
            self.data[round_num] = {}
        self.data[round_num][agent_name] = output

    def get_context_for_agent(self, current_round, current_agent_index):
        """Build context for an agent based on current round and agent index."""
        context = ""
        for r in range(1, current_round + 1):
            round_data = self.get_round(r)
            if r < current_round:
                context += f"\n=== ROUND {r} COMPLETE OUTPUTS ===\n"
                for agent, output in round_data.items():
                    context += f"\n{agent}:\n{self._truncate(output)}\n"
            elif r == current_round:
                context += f"\n=== ROUND {r} (Current Round) ===\n"
                for idx, (agent, output) in enumerate(round_data.items()):
                    if idx < current_agent_index:
                        context += f"\n{agent}:\n{self._truncate(output)}\n"
        return context
```

규칙은 두 줄이다.

- 이전 라운드는 **전부** 본다.
- 현재 라운드는 **자기보다 앞 순서만** 본다. 딕셔너리 삽입 순서가 곧 실행 순서라서 `idx < current_agent_index`로 자를 수 있다.

이 구조 덕분에 빌드 검증 결과도 특별한 채널 없이 들어간다. 검증이 실패하면 Round 2 딕셔너리에 `'Build Validation'`이라는 일곱 번째 "에이전트"로 추가된다. Round 3의 모든 에이전트는 Round 2를 전부 보니까 자동으로 이 항목을 받는다.

### 컨텍스트는 호출마다 늘어난다

![호출별 컨텍스트 항목 수](/assets/img/posts/multi-agent/context-light.png){: .light }
![호출별 컨텍스트 항목 수](/assets/img/posts/multi-agent/context-dark.png){: .dark }
_`get_context_for_agent()` 규칙으로 센 항목 수. 오른쪽 축은 항목당 잘림 상한 4,012자를 곱한 값_

Round 3의 Orchestrator는 Round 1 여섯 건, Round 2 여섯 건, Build Validation 한 건, Round 3 앞 순서 다섯 건으로 최대 18건을 받는다. 처음에는 그대로 이어 붙였고, 큰 과제에서는 컨텍스트 윈도우를 넘길 수 있었다. 그래서 항목마다 잘랐다.

```python
def _truncate(self, output):
    text = str(output)
    if len(text) <= self.MAX_CONTEXT_CHARS:
        return text
    half = self.MAX_CONTEXT_CHARS // 2
    return f"{text[:half]}\n...(중략)...\n{text[-half:]}"
```

앞 2,000자와 뒤 2,000자를 남긴다. `FILE:` 마커가 몰리는 앞부분과 결론부를 둘 다 지키려고 앞뒤를 남겼다. 잘린 결과는 구분 문자열 12자를 더해 4,012자다. 잘림은 컨텍스트를 만들 때만 적용되고 `data`의 원본은 그대로라서 파일 생성과 `final_result.md`에는 영향이 없다. 18건 × 4,000자면 본문만 72,000자가 상한이다.

## 3. 라운드 진행

```python
def run(self, task):
    for round_num in range(1, 4):
        for idx, agent in enumerate(self.agents):
            context = self.memory.get_context_for_agent(round_num, idx)

            if round_num == 1:
                prompt = f"Task: {task}"
            else:
                prompt = f"Task: {task}\n\nThis is Round {round_num}. Review previous rounds and IMPROVE your work. State what you changed and why."

            output = agent.call_model(prompt, context)
            self.memory.add(round_num, agent.name, output)
            time.sleep(2)

        if round_num == 2:
            self._run_build_validation()

    self._write_final_result()
    # ... 생략: FileGenerator(clean=True)로 파일 생성, 최종 검증 리포트 출력
```

출력(print)과 시간 측정 줄은 뺐다.

| 단계 | 하는 일 |
|---|---|
| Round 1 | 과제만 받고 순차 실행. 앞 순서 출력이 컨텍스트 |
| Round 2 | Round 1 전체 + "개선하고 무엇을 바꿨는지 말하라" |
| 빌드 검증 | Round 1~2의 `FILE:` 블록을 `output/.preview/`에 써 보고 검사 |
| Round 3 | Round 2와 같은 프롬프트. 검증 실패 시 에러 원문이 컨텍스트에 있다 |
| 생성 | `output/project/`를 비우고 Round 1→3 순서로 파일 기록 |
| 최종 검증 | 같은 검사를 한 번 더. 리포트만 출력 |

순차 구조라 피드백에는 지연이 있다. Developer는 Reviewer보다 앞 순서라서 같은 라운드의 리뷰를 못 본다. Developer 시스템 프롬프트의 "Reviewer feedback"은 항상 **이전 라운드** 리뷰다. Round 1 코드에 대한 리뷰는 Round 2에서, Round 2 코드의 빌드 에러는 Round 3에서 반영된다.

검증은 Round 2 뒤에만 피드백으로 돌아간다. 에러를 받아 고칠 기회는 Round 3 한 번이다. 마지막 검증은 Round 3에서 새로 깨진 것을 놓치지 않으려고 넣었고, 고칠 라운드가 없으니 리포트만 낸다.

파일 생성은 세 라운드 출력을 순서대로 디스크에 쓴다. 같은 경로면 뒤에 쓴 것이 이긴다. Round 3에서 `src/App.jsx`를 다시 냈다면 그게 최종본이다.

## 4. 프롬프트 설계

모든 호출은 같은 틀로 조립된다.

```python
def call_model(self, prompt, context=""):
    full_prompt = f"{self.system_prompt}\n\nContext from other agents:\n{context}\n\nYour task:\n{prompt}"
```

API의 system 역할을 따로 쓰지 않고 한 문자열로 보낸다. 대화 이력도 없다. 에이전트는 매 호출마다 상태 없이 처음부터 읽고, 기억은 전부 `context` 문자열이 대신한다.

시스템을 만들 때 쓴 지시문이 두 개 있다.

| 파일 | 에이전트 프롬프트 형태 |
|---|---|
| `agent_prompt.txt` (295줄) | Developer/Designer에는 "CRITICAL FORMAT" 규칙과 완성 예시 코드, Orchestrator에는 `FORMAT:`과 README 예시를 넣은 긴 프롬프트 |
| `feedback_prompt.txt` | 에이전트마다 한 문장. 2~3라운드 행동만 추가 |

실제 코드에 들어간 건 `feedback_prompt.txt`의 한 문장짜리다. 형식 예시는 프롬프트에 한 줄(`FILE: path` + 펜스)로만 남았다.

긴 예시의 흔적은 산출물에 있다. 커밋해 둔 `output/project/package.json`은 `agent_prompt.txt`의 예시와 내용이 같고(`"name": "game-portal"`), `components.css`도 예시의 `.game-card` 규칙으로 시작한다. `App.jsx`는 카운터 앱인데 패키지 이름은 게임 포털이다. 어느 실행의 결과인지 기록해 두지 않았다.

남은 규칙은 셋이다.

- **출력 형식**: `FILE: 경로` 다음 줄에 코드펜스. 파서가 이 형식만 알아본다.
- **역할 분리 힌트**: Developer에게 "Use className for styles". 스타일은 Designer의 CSS 파일로 가게 한다.
- **변경 설명 요구**: 2~3라운드 프롬프트마다 "State what you changed". 다음 에이전트가 무엇이 바뀌었는지 컨텍스트에서 바로 읽게 한다.

Idea 프롬프트의 "add NEW ideas"는 의도한 확장이다. 그 대가로 간단한 과제도 라운드를 거치며 기능이 크게 늘어날 수 있다.

## 5. 파일 추출: 한 번도 파일이 안 생기던 버그

최초 버전은 매 실행 `⚠️ No files generated`로 끝났다. 필터는 `'Developer'`를 찾는데 runner가 저장하는 키는 `agent.name`인 `'Developer Agent'`였다. 조건이 절대 참이 될 수 없었다.

```python
# 수정 전
if agent_name in ['Developer', 'Designer', 'Orchestrator']:
# 수정 후
if agent_name in ['Developer Agent', 'Designer Agent', 'Orchestrator Agent']:
```

이 버그를 고치면서 정규식도 완화했다. LLM이 형식을 미묘하게 어겨도 잡기 위해서다.

```python
bt = chr(96) * 3
# 수정 전
pattern = r'FILE:\s*([^\n]+)\n' + bt + r'(?:\w+)?\n(.*?)' + bt
# 수정 후
pattern = r'FILE:\s*([^\n`]+)\n+\s*' + bt + r'(?:\w+)?[ \t]*\n(.*?)' + bt
# 경로 후처리: filepath.strip().strip('`\'"')
```

두 정규식에 입력을 직접 넣어 보면 이렇다.

| 입력 형태 | 수정 전 | 수정 후 |
|---|---|---|
| `FILE: src/App.jsx` 바로 아래 펜스 | `src/App.jsx` | `src/App.jsx` |
| 경로와 펜스 사이 빈 줄 | 못 잡음 | `src/App.jsx` |
| 펜스의 언어 태그 뒤 공백 | 못 잡음 | `src/App.jsx` |
| 경로를 따옴표로 감쌈 | `"src/App.jsx"` | `src/App.jsx` |
| 경로를 백틱으로 감쌈 | `` `src/App.jsx` `` | 못 잡음 |
| `**FILE: src/App.jsx**` | `src/App.jsx**` | `src/App.jsx**` |

빈 줄과 공백은 해결됐다. 하지만 백틱을 허용하려던 수정이 오히려 백틱 경로를 놓치게 만들었다. 경로 캡처 `` [^\n`]+ ``가 백틱을 허용하지 않아서, 경로가 백틱으로 시작하면 한 글자도 못 먹고 매치가 실패한다. 백틱을 벗기는 뒤의 `strip`에는 도달하지 못한다.

마크다운 굵게 처리는 둘 다 `App.jsx**`라는 경로를 만든다. Windows에서 `*`는 파일 이름에 쓸 수 없어서 `write_text`가 `OSError: [Errno 22]`를 던진다. 이 예외는 잡지 않는다. 그 블록이 Round 1~2에 있으면 12회 호출 뒤 빌드 검증 단계에서, Round 3에만 있으면 18회 호출 뒤 파일 생성 단계에서 프로그램이 죽는다.

## 6. 빌드 검사 5종

Reviewer는 코드를 실행하지 않고 LLM 판단만으로 평가한다. 리뷰어도 LLM이라 없는 패키지를 알아보기 어렵다고 봤다. 그래서 판정을 실제 도구로 바꿨다.

| # | 검사 | 도구 | 조건 | 막는 것 |
|---|---|---|---|---|
| 1 | `package.json` 파싱 | `json.loads` | 파일이 있을 때 | 끝 쉼표, 주석 등으로 깨진 JSON |
| 2 | `package.json` 존재 | 파일 확인 | `.js/.jsx/.ts/.tsx`가 있는데 없을 때 | package.json 자체가 없는 결과물. import와 dependencies 대조는 하지 않는다 |
| 3 | JS 문법 | `node --check` | `node`가 있을 때 | `.js` 파일 문법 오류 |
| 4 | JSX 문법 | `npx --yes esbuild` | `npx`가 있을 때 | 닫히지 않은 태그 같은 JSX 오류 |
| 5 | 의존성 실재 | `npm install --package-lock-only` | `npm`이 있고 1~4가 전부 통과했을 때 | 레지스트리에 없는 패키지 이름이나 버전 |

2번은 실제 실행에서 import한 패키지가 어디에도 선언되지 않아 npm install조차 할 수 없는 결과물이 나와서 넣었다. 5번은 `--package-lock-only`라서 `node_modules`를 내려받지 않고 의존성 트리만 해석한다.

```python
if package_json.exists() and not issues and shutil.which('npm'):
    try:
        result = subprocess.run(
            'npm install --package-lock-only',
            cwd=project_dir, capture_output=True, text=True, encoding='utf-8', errors='replace',
            timeout=60, shell=True
        )
        if result.returncode != 0:
            issues.append(f"npm install 실패:\n{result.stderr.strip()[-1500:]}")
    except subprocess.TimeoutExpired:
        issues.append("npm install 시간 초과 (60초)")
```

`not issues` 조건 때문에 5번은 문법이 다 맞아야 돈다. 네트워크를 쓰는 가장 비싼 검사를 앞의 오류가 있을 때 건너뛰는 구조다. 대신 피드백 기회가 Round 2 뒤 한 번뿐이라, 문법 오류와 가짜 패키지가 같이 있으면 Round 3은 문법 오류만 보고 받는다.

도구가 없으면 그 검사만 건너뛴다. Node.js가 없어도 파이프라인은 끝까지 돈다.

### 에이전트가 실제로 받는 메시지

`ProjectValidator`에 일부러 깨뜨린 프로젝트를 넣어 돌렸다 (Windows, Node v24.18.0).

`package.json` 없이 닫는 태그가 빠진 `App.jsx`만 둔 경우 (esbuild 경고와 위치 표시 일부 생략):

```text
package.json이 생성되지 않았습니다. 코드에서 실제로 사용한 모든 의존성을 포함해 FILE: package.json 형식으로 반드시 출력하세요.
---
JSX 문법 오류:
X [ERROR] Unexpected end of file before a closing "div" tag

    a/src/App.jsx:5:0:
      5 │ 
        │ ^
        ╵ </div>
```

레지스트리에 없는 패키지를 선언한 경우 (뒷부분 생략):

```text
npm install 실패:
npm error code E404
npm error 404 Not Found - GET https://registry.npmjs.org/react-super-counter-hooks-zz - Not found
```

2번 메시지는 진단이 아니라 **지시문**이다. "무엇이 틀렸다"에서 멈추지 않고 `FILE: package.json` 형식으로 다시 내라는 행동까지 적었다. 이 문자열은 가공 없이 Round 3 에이전트 프롬프트에 들어간다. Runner는 리포트 앞에 "Round 3에서 반드시 수정하세요"를 붙여 메모리에 넣는다.

esbuild 경로에는 노이즈 제거가 하나 붙어 있다. `npx` 래퍼가 실패할 때 자체 Node 스택 트레이스를 stderr에 덧붙여서, `\nnode:child_process`나 `\nError: Command failed:` 앞에서 자른다.

### 검사끼리 가리는 경우

`package.json`이 깨진 상태에서 `.js`에도 문법 오류를 넣으면, `node --check`는 JS 오류 대신 이렇게 보고한다 (스택 일부 생략).

```text
Error: Invalid package config …\c\package.json.
    at getPackageScopeConfig (node:internal/modules/package_json_reader:193:33)
  code: 'ERR_INVALID_PACKAGE_CONFIG'
```

Node가 모듈 형식을 정하려고 가장 가까운 `package.json`을 먼저 읽기 때문이다. `package.json`이 정상이면 같은 파일에서 `SyntaxError: Unexpected token ';'`이 줄 번호와 함께 나온다. 이 경우 Round 3은 JS 오류의 위치를 모른 채 고쳐야 한다.

### 통과해도 안 돌아가는 것

커밋해 둔 `output/project/`를 같은 검사기에 넣으면 **5종 모두 통과**한다. 그런데 이 결과물은 실행되지 않는다.

| 빠진 것 | 결과 |
|---|---|
| `scripts.dev` | 콘솔이 안내하는 `npm run dev`가 바로 실패 |
| `react-dom`, 진입점(`main.jsx`, `index.html`) | `App` 컴포넌트를 마운트할 곳이 없다 |
| CSS import | `App.jsx`가 `main.css`를 불러오지 않는다 |

esbuild는 `--bundle` 없이 파일별로 변환만 하고, `node --check`도 import를 따라가지 않는다. 그래서 검사 5종은 "파일 하나하나가 문법적으로 맞고 선언한 패키지가 존재한다"까지만 보장한다. 프로젝트로서 조립되는지는 보지 않는다.

## 7. 실패 기록

개발하면서 만난 실패와 수정이다.

| 증상 | 원인 | 수정 |
|---|---|---|
| 매 실행 파일 0개 | 필터 `'Developer'` vs 저장 키 `'Developer Agent'` | 키 일치 (5장) |
| 형식 준수와 누적 컨텍스트 처리 부족 | `gemma2:2b` (2B 파라미터) | Gemini API로 교체 |
| 앞뒤가 안 맞는 프로젝트 | 이전 실행의 낡은 파일이 새 결과와 섞임 | `FileGenerator(clean=True)` |
| 백엔드가 죽어도 18회 호출을 다 쓰고 조용히 실패 | 사전 확인 없음 | 시작 전에 `GEMINI_API_KEY` 확인 후 즉시 종료. 키 존재만 확인한다. 키가 유효한지는 보지 않는다 |
| 한글 Windows 콘솔에서 이모지 출력 크래시 | cp949 `UnicodeEncodeError` | `sys.stdout.reconfigure(encoding='utf-8')` |
| `npm` 실행 시 `FileNotFoundError` | Windows의 `npm`은 `.cmd` 배치 파일 | `shell=True` |
| esbuild 출력 디코딩 실패 | 박스 문자를 cp949로 디코딩 | `encoding='utf-8', errors='replace'` |

마지막 두 줄은 검증 결과를 직접 확인할 때도 다시 만났다. 검사기 출력을 cp949 콘솔에 그대로 `print`하면 esbuild의 `╵`(U+2575)에서 같은 `UnicodeEncodeError`가 난다. `main.py`가 stdout을 먼저 UTF-8로 바꿔 두는 이유다.

## 8. 비용

![Round 1 에이전트별 응답 시간](/assets/img/posts/multi-agent/round1-timing-light.png){: .light }
![Round 1 에이전트별 응답 시간](/assets/img/posts/multi-agent/round1-timing-dark.png){: .dark }
_코드를 쓰는 Developer가 Round 1의 44%를 차지한다_

과제 하나에 API 호출은 재시도를 빼면 고정 18회다. 과제가 쉬워도 줄지 않는다. 라운드를 일찍 끝내는 조건이 없기 때문이다.

| 항목 | 값 | 근거 |
|---|---|---|
| 호출 수 | 18 | 6 에이전트 × 3라운드 |
| Round 1 응답 합 | 193.3초 | 실행 예시 1회 (Developer 85.8초) |
| 고정 대기 | 36초 | 호출마다 `time.sleep(2)` × 18 |
| 재시도 대기 | 호출당 최대 3초 | 1초, 2초 후 세 번째 실패면 포기 |
| 입력 크기 | 호출마다 증가, 최대 18건 | 2장 그래프 |

토큰 사용량과 금액은 측정하지 않았다.

재시도는 이렇게 생겼다.

```python
for attempt in range(3):
    try:
        response = self.client.models.generate_content(
            model=self.model,
            contents=full_prompt
        )
        return response.text
    except Exception as e:
        if attempt < 2:
            time.sleep(2 ** attempt)
        else:
            return f"Error: {e}"
```

세 번 다 실패하면 예외를 올리지 않고 `"Error: ..."` 문자열을 **그 에이전트의 출력으로** 돌려준다. runner는 이걸 정상 출력처럼 메모리에 넣고, 뒤 에이전트들은 에러 문자열을 동료의 결과물로 읽는다. 파이프라인은 멈추지 않지만 그 라운드의 해당 역할은 빈 칸이 된다.

## 9. 한계와 다음 단계

문서에 정리해 둔 한계:

- **Round 3 뒤에는 되돌릴 라운드가 없다.** 최종 검증은 리포트만 내고 자동으로 고치지 않는다.
- **JSX는 문법만 본다.** 타입 검사나 렌더링은 하지 않는다.
- **범위가 불어난다.** "새 아이디어를 추가하라"는 지시 때문에 단순한 과제도 기능이 늘 수 있다.
- **컨텍스트는 항목당 4,000자로 잘린다.** 긴 코드의 가운데는 다음 에이전트가 못 본다.

다시 돌려 보며 찾은 결함:

- **`.preview` 폴더는 비우지 않는다.** `clean=True`는 `output/project/`에만 적용돼서, 이전 실행의 낡은 파일이 섞이는 문제가 `.preview`에는 남아 있다.
- **파일은 세 라운드의 합집합이다.** Round 1에서 만든 파일을 Round 3에서 이름을 바꿔도 옛 파일이 같이 남는다.
- **Orchestrator도 코드 파일을 덮어쓸 수 있다.** 추출 대상이고, 같은 라운드 안에서는 Orchestrator가 마지막이라 같은 경로면 Orchestrator 쪽이 남는다.
- **`.ts/.tsx`는 존재 여부에만 쓰인다.** 문법 검사 대상이 아니다.
- **esbuild 검사는 Windows 전제다.** 인자 리스트와 `shell=True`를 같이 쓰는데, POSIX에서는 `--yes esbuild …`는 셸 위치 인자가 되어 `npx`만 인자 없이 실행된다. 오류로 끝나든 60초 시간 초과든 `[]`를 돌려주므로 JSX 검사가 통과로 처리된다.
- **정규식 완화가 백틱 경로를 놓친다** (5장).

검사 5종이 바꾼 건 피드백의 종류다. Reviewer의 "이 부분을 개선하면 좋겠다"는 의견 옆에, 줄 번호가 붙은 컴파일러 에러와 404가 난 패키지 이름이 들어간다. 다만 그 메시지는 한 번만 전달되고, "파일이 문법적으로 맞다"는 "프로젝트가 돌아간다"가 아니다. 커밋해 둔 결과물이 검사를 전부 통과하고도 `npm run dev`에서 멈춘다는 점이 그 차이를 보여준다.

코드: [github.com/kasangyong/multi_agent](https://github.com/kasangyong/multi_agent)
