# Hangul (Korean) Input Support for kmscon

## Overview

kmscon에 libhangul 기반 한글 입력 기능을 추가하였다.
기존 키 입력 경로(evdev → XKB → keysym → VTE → PTY)에서 VTE 앞 단계에
한글 조합 레이어를 삽입하여, 초성+중성+종성 조합이 완료된 문자를 PTY에
직접 UTF-8로 전송하는 방식이다.

## Dependencies

- **libhangul** (>= 0.2.0) - `pkg-config` name: `libhangul`
- Debian/Ubuntu: `sudo apt install libhangul-dev`

## Build

libhangul이 시스템에 설치되어 있으면 자동으로 감지된다(auto).

```sh
meson setup build
ninja -C build
```

명시적으로 활성화/비활성화:

```sh
meson setup build -Dhangul=enabled   # 강제 활성화 (없으면 에러)
meson setup build -Dhangul=disabled  # 비활성화
```

빌드 요약에서 확인:

```
Input Methods
  hangul              : true
```

`config.h`에 `BUILD_ENABLE_HANGUL`이 정의되며, 모든 한글 관련 코드는
`#ifdef BUILD_ENABLE_HANGUL` 가드로 감싸져 있어 비활성화 시 기존 동작에
영향을 주지 않는다.

## Configuration

### Command-line options

| Option | Default | Description |
|---|---|---|
| `--hangul` | off | 한글 입력 기능 활성화 |
| `--hangul-keyboard <id>` | `2` | 한글 키보드 레이아웃 |
| `--grab-hangul-toggle <grab>` | `Hangul` | 한/영 전환 단축키 |

### Config file (`/etc/kmscon/kmscon.conf`)

```
hangul
hangul-keyboard=2
grab-hangul-toggle=Hangul
```

`hangul` 한 줄만 추가하면 나머지는 기본값으로 동작한다.

### Keyboard layouts

| ID | Layout |
|---|---|
| `2` | 두벌식 (표준) |
| `2y` | 두벌식 옛한글 |
| `3f` | 세벌식 최종 |
| `39` | 세벌식 390 |
| `3s` | 세벌식 순아래 |

## Usage

1. kmscon 콘솔(tty)에서 **Hangul 키**를 눌러 한글 모드로 전환
2. 영문 키 입력 시 한글 조합이 진행됨
   - 예: `r` → ㄱ, `r` `k` → 가, `r` `k` `s` → 간
3. 조합 중인 문자는 커서 위치에 밑줄과 함께 표시(preedit)
4. **Hangul 키**를 다시 눌러 영문 모드로 복귀

### Key behavior in Hangul mode

| Key | Behavior |
|---|---|
| 영문자 (a-z) | libhangul로 전달, 한글 조합 진행 |
| Shift + 영문자 | libhangul로 전달 (쌍자음/이중모음) |
| Backspace | 조합 중이면 자소 단위 삭제, 아니면 일반 backspace |
| Enter, Tab, 방향키, Esc | 진행 중 조합을 커밋 후 원래 기능 수행 |
| Ctrl/Alt 조합 | 진행 중 조합을 커밋 후 원래 기능 수행 |
| Hangul 키 | 한/영 모드 전환 |

### Custom toggle key

Hangul 키가 없는 키보드에서는 다른 키를 지정할 수 있다:

```
grab-hangul-toggle=<Alt>space
```

또는 Right Alt 키:

```
grab-hangul-toggle=ISO_Level3_Shift
```

## Modified Files

### Build system

- **`meson.options`** - `hangul` feature option 추가 (type: feature, value: auto)
- **`meson.build`** - libhangul dependency, feature loop entry, summary section
- **`src/meson.build`** - kmscon 바이너리에 libhangul 조건부 링크

### Configuration

- **`src/kmscon_conf.h`** - `kmscon_conf_t` 구조체에 필드 추가
  - `bool hangul` - 기능 활성화 플래그
  - `char *hangul_keyboard` - 키보드 레이아웃 ID
  - `struct conf_grab *grab_hangul_toggle` - 한/영 전환 키
- **`src/kmscon_conf.c`** - 옵션 정의, 기본값, help 텍스트

### Core implementation

- **`src/kmscon_terminal.c`** - 전체 한글 입력 로직
  - `#include <hangul.h>` (조건부)
  - `struct kmscon_terminal`에 `hangul_ic`, `hangul_mode`, `preedit_ch` 필드 추가
  - Helper functions:
    - `hangul_commit_ucs4()` - UCS-4 문자열을 UTF-8로 변환하여 PTY에 쓰기
    - `hangul_flush()` - 진행 중 조합을 커밋
    - `hangul_update_preedit()` - preedit 문자 갱신
    - `hangul_should_passthrough()` - 바이패스 키 판별
    - `hangul_process_key()` - 한글 키 처리 메인 로직
  - `draw_hangul_preedit()` - 커서 위치에 조합 중 문자를 밑줄과 함께 렌더링
  - `input_event()` - 한/영 토글 및 한글 키 처리 삽입
  - `kmscon_terminal_register()` - HangulInputContext 초기화
  - `terminal_destroy()` - HangulInputContext 해제
  - `session_event()` - 세션 비활성화 시 조합 플러시

## Architecture

```
Key Press
  │
  ▼
input_event()
  │
  ├─ grab check (scroll, zoom, rotate, ...)
  │
  ├─ Hangul toggle key? ──▶ toggle hangul_mode, flush if leaving
  │
  ├─ hangul_mode ON?
  │   │
  │   ├─ hangul_process_key()
  │   │   ├─ Backspace → hangul_ic_backspace()
  │   │   ├─ Passthrough key → hangul_flush() + return false
  │   │   └─ Printable → hangul_ic_process()
  │   │       ├─ commit string → hangul_commit_ucs4() → PTY
  │   │       └─ preedit string → preedit_ch → redraw
  │   │
  │   └─ consumed? return : fall through to VTE
  │
  └─ tsm_vte_handle_keyboard() (default path)
```
