---
layout: post
title: "계정 전환 스크립트에 박아둔 토큰은 처음부터 죽은 코드였다 - gh CLI 자격증명 위임 구조"
date: 2026-11-17 18:10:00 +0900
categories: [오류해결, GitHub CLI]
tags: [GitHub, GitHub CLI, Git, PowerShell, PAT, 자격증명]
---

## 문제 상황

GitHub 계정 두 개(개인용·업무용)를 `ghs <계정>` 한 줄로 전환하는 PowerShell 스크립트를 쓰고 있었다.
그런데 전환이 자유롭지 않았다. 특정 이름 하나로만 통하고, 계정 ID를 그대로 입력하면 터졌다.

```
ghs myname-work
# github-switch.ps1 : 'Account' 인수에 대한 인수 변환을 처리할 수 없습니다.
# 집합 "personal, work"에 속하지 않습니다.
```

고치려고 스크립트를 열었는데, 정작 문제는 전환 로직이 아니라 **자격증명을 심는 코드 전체가 애초에 동작하지 않고 있었다**는 쪽이었다.

## 원인 1 - ValidateSet이 별칭을 막고 있었다

```powershell
param(
    [Parameter(Mandatory=$true)]
    [ValidateSet("personal", "work")]
    [string]$Account
)
```

`ValidateSet`은 **정확히 일치하는 문자열만** 통과시킨다.
계정 ID(`myname-work`)와 스크립트가 받는 키(`work`)가 달랐으니, 기억하고 있는 이름을 쳐도 전부 검증 에러였다.

## 원인 2 - 토큰을 등록하는 코드를 아무도 읽지 않았다 (핵심)

스크립트에는 PAT를 평문으로 박아두고, 전환할 때마다 git 자격증명을 다시 등록하는 블록이 있었다.

```powershell
$rejectInput  = "protocol=https`nhost=github.com`n"
$approveInput = "protocol=https`nhost=github.com`nusername=$user`npassword=$token`n`n"

$rejectInput  | git credential reject  2>&1 | Out-Null
$approveInput | git credential approve 2>&1 | Out-Null
```

겉보기엔 그럴듯하다. 기존 자격증명을 지우고 새 계정 토큰을 넣는다.
그런데 전역 `.gitconfig`를 열어보니 이런 설정이 있었다.

```ini
[credential "https://github.com"]
	helper =
	helper = !'C:/Program Files/GitHub CLI/gh.exe' auth git-credential
```

`gh auth setup-git`이 심어두는 설정이다. 두 줄의 의미가 각각 다르다.

- 빈 `helper =` → **상위 스코프에서 누적된 helper 목록을 리셋한다.** system 설정의 `credential.helper = manager`(Windows 자격증명 관리자)가 여기서 지워진다.
- 다음 `helper = !gh auth git-credential` → github.com 자격증명 조회를 **gh CLI에 위임한다.**

git의 credential helper는 단일 값이 아니라 **목록**이고, 빈 문자열을 대입하면 그때까지 쌓인 목록이 비워진다. 그래서 github.com URL에 대해서는 최종적으로 gh 하나만 남는다.

결론적으로 git이 `github.com` 자격증명을 물을 때 Windows 자격증명 관리자가 아니라 **gh에게 묻고, gh는 keyring에 저장된 "활성 계정"의 토큰을 돌려준다.** 정리하면

- `git credential approve`로 밀어넣은 값은 **아무도 읽지 않는다**
- 스크립트에 박아둔 PAT 2개는 **한 번도 쓰이지 않았다** — 유출 위험만 지고 있던 죽은 코드
- 실제로 전환을 수행하는 건 `gh auth switch` 한 줄

### 어떤 helper가 실제로 쓰이는지 확인하기

```bash
git config --global --get-all credential.https://github.com.helper
```

빈 줄 하나 + `!... auth git-credential` 이 나오면 gh에 위임된 상태다.
`--show-origin`을 붙이면 어느 파일에서 온 설정인지까지 보인다.

```bash
git config --list --show-origin | grep credential
```

git이 실제로 돌려받는 자격증명을 직접 확인할 수도 있다.

```bash
printf 'protocol=https\nhost=github.com\n\n' | git credential fill
```

`username=` 이 gh의 활성 계정으로 나온다.
다만 이 명령은 **토큰을 평문으로 출력**하므로, 화면 공유 중이거나 로그가 남는 터미널에서는 쓰지 않는 게 좋다.

## 원인 3 - 두 계정의 커밋 이메일이 같았다

`user.name`만 계정별로 바꾸고 `user.email`은 두 계정이 같은 주소를 공유하고 있었다.
GitHub는 **커밋 작성자 이메일로 계정을 연결**하므로, 이 상태면 어느 계정으로 전환하든 커밋 귀속이 한쪽으로 쏠린다.

## 해결 방법

### 1. ValidateSet 제거 후 정규화해서 비교

입력을 소문자 + 영숫자만 남기고 비교하면 대소문자·하이픈·밑줄 차이를 전부 흡수할 수 있다.

```powershell
function Convert-ToKey([string]$value) {
    if ([string]::IsNullOrWhiteSpace($value)) {
        return ''
    }
    return ($value.ToLowerInvariant() -replace '[^a-z0-9]', '')
}
```

별칭 테이블도 정규화된 형태로 적어둔다.

```powershell
$accounts = @(
    @{
        id      = 'myname'
        email   = 'me@example.com'
        aliases = @('myname', 'personal', 'me')
    },
    @{
        id      = 'myname-work'
        email   = 'me@company.example.com'
        aliases = @('mynamework', 'work', 'company')
    }
)

$key    = Convert-ToKey $Account
$target = $accounts | Where-Object { $_.aliases -contains $key } | Select-Object -First 1
```

`myname-work`, `MYNAME-WORK`, `myname_work` 가 모두 `mynamework`로 정규화돼 같은 항목에 걸린다.

### 2. 토큰과 credential 블록을 전부 삭제하고 gh keyring을 단일 출처로

```powershell
git config --global user.name  $target.id
git config --global user.email $target.email

gh auth switch --user $target.id --hostname github.com
if ($LASTEXITCODE -ne 0) {
    Write-Host "[ERROR] gh 계정 전환 실패: $($target.id)" -ForegroundColor Red
    Write-Host "        gh auth login --hostname github.com" -ForegroundColor Yellow
    exit 1
}
```

자격증명이 gh에 위임돼 있으므로 토큰을 어디에도 심을 필요가 없다.

### 3. 에러를 숨기지 않고, 전환 결과를 검증

기존 스크립트는 이렇게 끝났다.

```powershell
gh auth switch --user $user --hostname github.com 2>$null
Write-Host "[OK] GitHub account switched: $user"
```

`2>$null`로 에러를 버리고 **무조건** `[OK]`를 찍는다. keyring에 로그인돼 있지 않아 전환이 실패해도 성공처럼 보였다.
실제 결과를 한 번 더 물어보게 바꿨다.

```powershell
$actual = (gh api user --jq '.login' 2>$null)
if ($actual -and $actual -ne $target.id) {
    Write-Host "[WARN] gh 활성 계정이 목표와 다르다: 목표=$($target.id), 실제=$actual" -ForegroundColor Yellow
}
```

### 4. 전환해도 안 바뀌는 두 가지 함정을 경고로 추가

전환은 성공했는데 push 계정이 그대로인 경우가 있다. 원인은 보통 둘 중 하나다.

**자격증명 위임이 깨진 경우** — `.gitconfig`의 helper 설정이 사라지면 Windows 자격증명 관리자에 남은 옛 토큰이 다시 쓰인다.

```powershell
$helpers = @(git config --global --get-all 'credential.https://github.com.helper' 2>$null)
if (-not ($helpers -match 'auth git-credential')) {
    Write-Host "[WARN] github.com 자격증명이 gh CLI에 위임돼 있지 않다." -ForegroundColor Yellow
    Write-Host "        복구: gh auth setup-git --hostname github.com" -ForegroundColor Yellow
}
```

**저장소 로컬 설정이 전역을 덮어쓰는 경우** — `git config --local user.email`이 박혀 있으면 전역 전환이 그 저장소에선 무효다. 이게 "전환했는데 커밋 작성자가 안 바뀐다"의 가장 흔한 원인이다.

```powershell
if ((git rev-parse --is-inside-work-tree 2>$null) -eq 'true') {
    $localEmail = (git config --local user.email 2>$null)
    if ($localEmail) {
        Write-Host "[WARN] 이 저장소에 로컬 user 설정이 있어 전역 전환이 적용되지 않는다." -ForegroundColor Yellow
    }
}
```

## 결과

- 별칭 아무 형태로나 전환된다 (`myname-work`, `MYNAME-WORK`, `work`, `company` 모두 동일)
- 스크립트 내 평문 토큰 0건
- 전환 실패가 성공으로 표시되지 않는다
- 양방향 전환 후 `git ls-remote origin HEAD`로 실제 인증까지 확인

## 덤으로 - 이미 노출된 토큰은 파일을 지워도 해소되지 않는다

하드코딩을 걷어내는 것과 **이미 평문으로 디스크에 있던 토큰을 처리하는 것**은 별개다. 파일을 지우고 백업을 지워도 토큰 문자열 자체는 여전히 유효하다. 결국 폐기(revoke)해야 끝난다.

순서에 함정이 하나 있었다. 스크립트에 박혀 있던 PAT가 **gh가 현재 쓰고 있는 토큰과 동일한 값**이었다. 비교해 보고 알았다.

```powershell
# 토큰 값을 화면에 띄우지 않고 일치 여부만 확인
$hard = [regex]::Matches((Get-Content ./github-switch.ps1 -Raw), 'ghp_[A-Za-z0-9]+') | ForEach-Object { $_.Value }
$live = (gh auth token)
[bool]($hard -contains $live)
```

`True`가 나왔다. 그러니 폐기하면 gh 로그인이 같이 끊긴다. 폐기 직후 재로그인이 필요하다.

```bash
gh auth login --hostname github.com
```

이때 인증 방식으로 **Login with a web browser**를 고르면 OAuth 토큰(`gho_`)이 발급된다.
`Paste an authentication token`을 골라 PAT를 다시 붙여넣으면 평문 토큰을 또 만들어 쓰는 셈이라, 애초의 문제로 돌아간다.

그리고 토큰 값은 파일에만 남는 게 아니다. 터미널 히스토리, 편집기 백업, 작업 로그처럼 생각하지 못한 곳에 사본이 남는다. 탐색은 패턴으로 한 번 훑는 게 빠르다.

```bash
grep -rl 'ghp_[A-Za-z0-9]' ~/scripts ~/.gitconfig 2>/dev/null
```

## PowerShell 5.1에서 .ps1 한글 주석이 깨진다면

고치는 과정에서 같이 걸린 문제. Windows PowerShell 5.1은 **BOM이 없는 `.ps1`을 ANSI(한국어 환경이면 cp949)로 읽는다.**
UTF-8로 저장한 한글 주석과 출력 문자열이 전부 깨진다. UTF-8 **BOM 포함**으로 저장하면 해결된다.

```bash
# BOM 확인 - ef bb bf 로 시작해야 한다
head -c 3 github-switch.ps1 | od -An -tx1
```

기존 파일은 BOM이 없었는데도 멀쩡해 보였던 이유는, 깨지는 대상이 주석뿐이고 출력 문자열은 전부 ASCII였기 때문이다. 한글 출력을 추가하는 순간 드러난다.

## 정리

고장 난 줄 알았던 건 전환 로직이었지만, 실제로 고장 난 건 **"설정을 심는 코드"와 "그 설정을 읽는 주체"의 불일치**였다.
자격증명이 안 먹는다는 증상을 만나면, 심는 쪽을 의심하기 전에 **누가 읽는지**를 먼저 확인하는 게 빠르다. git에서는 `git config --list --show-origin | grep credential` 한 줄이면 된다.
