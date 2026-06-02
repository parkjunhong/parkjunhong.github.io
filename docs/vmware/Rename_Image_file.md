가상 머신(VM)이 완전히 종료(Power Off)된 상태인지 확인한 후 작업을 진행해야 합니다.

### 1. 삭제해도 안전한 파일 목록

VMware가 실행 중 발생한 오류나 로그를 기록한 파일들로, 가상 머신의 실제 구동이나 데이터와는 무관하므로 삭제가 가능합니다.

* **로그 파일 (`.log`)**: `vmware.log`, `vmware-0.log`, `vmware-1.log`, `vmware-2.log`
* **크래시 덤프 파일 (`.dmp`)**: `vmware-vmx.dmp`, `vmware-vmx-0.dmp`, `vmware-vmx-1.dmp`
* **압축된 코어 덤프 파일 (`.gz`)**: `vmmcores-2.gz`, `vmmcores-3.gz`
* **스코어보드 파일 (`.scoreboard`)**: `Ubuntu 20.04-x64-bit.scoreboard` 및 숫자가 붙은 모든 파일 (VM 실행 중 생성되는 프로세스 추적용 파일이며, 정상 종료 상태라면 삭제해도 무방합니다.)

---

### 2. VM 이름 및 파일명 변경 방법

VMware에서 디렉토리 내의 파일명을 직접 변경할 경우, 설정 파일들이 기존 이름을 참조하고 있어 오류가 발생합니다. 인플레이스(In-place)로 이름을 변경하려면 파일명 변경과 함께 설정 파일 내부의 텍스트도 수정해야 합니다.

**사전 작업**

1. VMware Workstation에서 해당 VM을 완전히 종료합니다 (Suspend 상태 불가).
2. VMware Workstation 인벤토리에서 해당 VM을 제거합니다 (해당 VM 우클릭 → `Remove` 선택, **Delete from Disk를 선택하지 않도록 주의**).

**1단계: 파일 이름 일괄 변경**
첨부된 이미지가 Windows 환경이므로 PowerShell을 활용하면 40개가 넘는 분할 가상 디스크 파일(`-s0xx.vmdk`)의 이름을 한 번에 변경할 수 있습니다.
해당 VM이 위치한 폴더로 이동하여 PowerShell을 열고 아래 명령어를 실행합니다.

```powershell
Get-ChildItem -Filter "Ubuntu 20.04-x64-bit*" | Rename-Item -NewName { $_.Name -replace 'Ubuntu 20.04-x64-bit', 'Ubuntu-x64' }

```

**2단계: 설정 파일 내부 텍스트 수정**
이름이 변경된 파일 중 메타데이터 및 디스크립터 역할을 하는 파일들을 텍스트 편집기(VS Code, Notepad++ 등)로 열어 내부의 참조 경로를 업데이트해야 합니다.

1. **`Ubuntu-x64.vmx` 수정**
* 파일을 열고 `Ubuntu 20.04-x64-bit` 문자열을 모두 찾아 `Ubuntu-x64`로 일괄 변경(Replace All) 합니다. (주요 변경 대상: `displayName`, `nvram`, `extendedConfigFile`, `.vmdk` 참조 경로 등)


2. **`Ubuntu-x64.vmxf` 수정**
* XML 형태의 메타데이터 파일입니다. 동일하게 `Ubuntu 20.04-x64-bit`를 `Ubuntu-x64`로 일괄 변경합니다.


3. **`Ubuntu-x64.vmdk` 수정 (메인 디스크 디스크립터)**
* 용량이 GB 단위인 분할 파일(`-s0xx.vmdk`)이 아닌, **가장 용량이 작은(수 KB) 메인 `.vmdk` 파일**을 텍스트 편집기로 엽니다.
* 파일 내부의 `Extent description` 섹션에 분할된 42개의 파일명이 리스트업되어 있습니다. 이 파일명들의 `Ubuntu 20.04-x64-bit` 부분을 모두 `Ubuntu-x64`로 변경하고 저장합니다.



**3단계: 인벤토리 재등록**

1. 파일 수정이 모두 완료되면 VMware Workstation을 실행합니다.
2. `File` → `Open` 메뉴를 통해 이름이 변경된 `Ubuntu-x64.vmx` 파일을 선택하여 인벤토리에 추가합니다.
3. VM을 최초로 부팅할 때 복사 또는 이동 여부를 묻는 팝업창이 나타납니다. 기존 네트워크 설정(MAC 주소)을 그대로 유지하기 위해 반드시 "I Moved It"을 선택합니다.
