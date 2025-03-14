# sudo rm -rf agentic_security

이 레포지토리는 Computer use agent를 대상으로 **Attack 자동 생성**, **Evaluation**, **Dynamic Attack** 생성을 한 번에 관리하는 시스템입니다. **Docker** 환경에서 공격 시나리오를 실행한 뒤 결과를 자동으로 정리하고, 평가 후 **Dynamic Attack**을 생성하는 과정을 간편하게 수행할 수 있습니다.

![SUDO Attack Framework Architecture](sudo_figure.png)

---

## 목차

1. [개요](#개요)  
2. [폴더 구조](#폴더-구조)  
3. [준비 사항](#준비-사항)  
4. [사용 방법](#사용-방법)  
5. [상세 워크플로우](#상세-워크플로우)  
6. [참고 사항](#참고-사항)  

---

## 개요

- **Attack Generation**:  
  공격 JSON을 생성하고, **Scene Change Task**를 삽입.  
  결과물은 `computer_use_demo/data/` 폴더에 복사.  
  이후 **Docker**로 공격을 실행해 로그를 생성.
- **Evaluation**:  
  생성된 로그를 `eval/logs/`로 이동하여 평가.  
  수치 계산 등을 진행.
- **Dynamic Attack**:  
  평가 결과를 활용해 **동적 공격**(Dynamic Attack)을 추가 생성.

모든 단계는 **`main.py`** 단일 스크립트로 자동화됨.

---

## 폴더 구조

```plaintext
sudo
├── main.py                # 전체 파이프라인을 관리하는 메인 스크립트
├── attack
│   ├── attack_generation.py
│   └── result.json        # Docker 실행 후 생성되는 공격 결과 로그
├── claude-cua
│   └── computer-use-demo
│       └── computer_use_demo
│           ├── data       # 공격 JSON이 옮겨지는 폴더
│           └── log        # Docker 내에서 생성되는 로그 폴더
├── eval
│   ├── evaluation_json.py # 평가 로직 (로그 파일 → 점수 추출)
|   ├── calculate_score.py # 평가 로직 (점수 → 수치 계산)
│   └── logs               # 공격 결과 로그 최종 저장 (이후 평가)
├── dynamic_attack
│   └── dynamic_attack.py  # Dynamic Attack 처리
├── formatter
│   ├── auto-scene
│   └── csv2json
│       └── convert_format.py
├── .env                   # (필요 시) 환경 변수를 담을 수도 있음
├── .gitignore
├── LICENSE
├── pyproject.toml
└── README.md              # 바로 이 파일
```

주의: 실제 Docker 마운트 경로(-v 옵션)와 로컬 폴더 구조를 일치시켜야 함(현재 적용함.)

## 준비 사항
0. conda create -n sudo python=3.10
<<아직 정리 다 안됨>>
1. 필요 패키지 설치: pip install -r requirements.txt
2. Docker
Docker가 설치되어 있고, 명령줄에서 docker run을 실행할 수 있어야 합니다.
3. 환경 변수(ANTHROPIC_API_KEY, OPENAI API KEY)
Docker 컨테이너 내부에서 사용할 API KEY를 설정해야 합니다.
예) export ANTHROPIC_API_KEY="YOUR_KEY_HERE"
4. Computer use agent 대상 공격 전, Victim account 생성 및 Attacker 로그인


## 사용 방법
main.py 는 CLI 인자로 공격 생성/평가/동적 공격을 분리 실행하거나, 한 번에 처리.

1. Static Attack만 실행

```bash
python main.py --attack ./attack/result/result.csv dynamic_response_round_1
python3 main.py --attack-gen o1_static o1  #[명령어 완료, 고정]
python3 main.py --docker-run #[명령어 완료, 고정]
python main.py --formatter claude3.7_dynamic_1 dynamic_response_round_1 #[csv이름, column이름: 명령어 완료, 고정]

```
* 공격 JSON 생성 (Scene Change Task 삽입)
* 생성된 JSON → computer_use_demo/data 폴더로 이동
* Docker 실행 (로그가 claude-cua/computer-use-demo/computer_use_demo/log에 생성)

2. Evaluation만 실행

```bash
python main.py --evaluate deharm_claude3.7_static #[명령어 완료, 고정]
```
* Docker 결과(attack/result.json)를 eval/logs로 이동
* evaluation_json.py 스크립트 실행 → 수치 계산 등

3. Dynamic Attack만 실행

```bash
python3 main.py --dynamic 
```
* 평가 결과(eval/logs 폴더 등)를 기반으로 Dynamic Attack 생성

4. 전체 파이프라인 자동 실행 (순서대로 Attack → Evaluate → Dynamic)


```bash
python main.py --all ./attack/result/result.csv dynamic_response_round_1 deharm_claude3.7_static #[3번째 인자 고정]

```
* 공격 생성 → 공격 진행 → 평가 → 동적 공격 순으로 자동 실행.

## 상세 워크플로우
1. Attack Generation
* attack_generation.py가 공격 JSON을 생성하고,
Scene Change Task 등을 삽입(필요 시 formatter/auto-scene 기능 활용).
* 완료된 JSON을 computer-use-demo/computer_use_demo/data 폴더로 자동 이동.

2. Docker 컨테이너 실행
* main.py 내부 run_attack() 단계에서 Docker를 실행.
* 공격 JSON을 기반으로 실제 공격 시뮬레이션 진행.
* 공격 로그(result.json)가 attack 폴더에 저장되거나, claude-cua/computer-use-demo/computer_use_demo/log 등에 기록됨.

3. Evaluation
* main.py가 평가 단계를 실행(--evaluate).
* attack/result.json → eval/logs 폴더로 자동 이동 후,
* evaluation_json.py를 통해 로그를 읽어 평가 및 수치 계산 수행.

4. Dynamic Attack
* dynamic_attack/dynamic_attack.py에서 평가 결과를 활용해 동적 공격을 생성.


## 참고 사항
* Docker 경로 설정이 실제 로컬 경로와 잘 맞는지 확인하기. (-v $(pwd)/...:/home/...)
* ANTHROPIC_API_KEY가 정상 전달되는지 체크. 필요 시 shell=True 대신 환경 변수를 직접 subprocess.run()에 전달할 수도 있다.

## 라이선스 및 기여
* 이 프로젝트는 LICENSE 파일의 규정을 따릅니다.
* 버그 리포트, 기능 제안 등은 자유롭게 이슈를 등록하거나 PR을 날려주세요.