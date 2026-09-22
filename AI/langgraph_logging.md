# LangGraph 로깅

로깅은 프로그램 실행 중 발생한 이벤트를 기록하는 작업이다. LangGraph에서는 Node 시작·완료, 선택된 경로, 외부 호출, 예외와 traceback을 기록해 실행 흐름을 추적하고 장애 원인을 분석할 수 있다.

## logging 구성 요소

- **Logger**: 로그 이벤트를 기록하는 객체
- **Handler**: 로그를 화면, 파일 등 특정 목적지로 전달
- **Formatter**: 시각, 레벨, Logger 이름, 메시지 등의 출력 형식 정의
- **Level**: 기록할 중요도의 기준

하나의 Logger에 여러 Handler를 연결하면 같은 이벤트를 화면과 파일에 서로 다른 형식이나 기준으로 남길 수 있다.

## 로그 레벨

```text
DEBUG < INFO < WARNING < ERROR < CRITICAL
```

DEBUG는 상세 진단 정보, INFO는 정상적인 진행 상황, WARNING은 주의가 필요한 전환, ERROR는 요청 실패, CRITICAL은 서비스 운영이 어려운 문제에 사용한다. 이벤트가 출력되려면 Logger와 해당 Handler의 기준을 모두 통과해야 한다.

## 화면과 파일 로그 분리

개발 중에는 화면에 진행 상황을 보여주고, 운영에서는 파일에 상세한 실행 기록을 남길 수 있다. Logger는 DEBUG로 설정하고 화면 Handler는 INFO, 파일 Handler는 DEBUG로 설정하면 상세 로그는 파일에만 기록할 수 있다.

설정 코드를 여러 번 실행할 수 있는 환경에서는 기존 Handler를 제거하고 닫아 중복 출력과 파일 핸들 누적을 방지해야 한다.

## 로그 로테이션

`RotatingFileHandler`는 일정 크기마다 파일을 교체한다. `maxBytes`는 교체 기준 크기이고, `backupCount`는 보관할 이전 파일 수다. 로그 보존 기간과 파일 수를 정해 저장 공간을 관리한다.

## 예외와 traceback

`logger.error()`는 메시지를 기록하고, `logger.exception()`은 `except` 블록 안에서 현재 예외의 traceback을 함께 기록한다. 기록과 예외 처리는 별개이므로, 호출자에게 실패를 전달하거나 복구 로직을 실행하려면 별도로 `raise`하거나 오류 상태를 반환해야 한다.

## Node 로깅 기준

각 Node에서 Graph 이름, Thread ID, Node 이름, 시작·완료 시각, 외부 호출 소요 시간, 선택된 경로, 오류 유형을 일정한 형식으로 기록하면 실행 흐름을 추적하기 쉽다. 사용자 입력, 계좌 정보, 인증 토큰 같은 민감 정보는 마스킹하거나 식별자로 대체한다.

## 운영 설계 기준

- `print()` 대신 Logger와 Level을 사용해 출력 기준을 분리한다.
- 요청을 추적할 수 있는 correlation ID를 함께 남긴다.
- 로그 파일의 크기와 보존 기간을 제한한다.
- 예외 기록에 traceback과 필요한 실행 Context를 포함한다.
- 개인정보와 비밀 값을 기록하지 않는다.
