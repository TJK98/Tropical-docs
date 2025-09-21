## OpenSSL을 통해 JWT key 발급 받기


'openssl rand -base64 64' 명령어를 사용하는 방법은 운영체제에 따라 다소 차이가 있습니다. 윈도우에서는 OpenSSL을 별도로 설치해야 하는 경우가 많지만, macOS와 리눅스에서는 보통 기본적으로 설치되어 있어 바로 사용할 수 있습니다.

### **macOS 및 리눅스 (터미널)**

macOS와 리눅스는 대부분 OpenSSL이 내장되어 있으므로 별도의 설치 과정 없이 터미널에서 바로 명령어를 실행할 수 있습니다.

1.  **터미널 열기**:

      * **macOS**: "응용 프로그램 \> 유틸리티 \> 터미널" 또는 Spotlight(Command + Space)에서 "터미널"을 검색하여 실행합니다.
      * **리눅스**: "Ctrl + Alt + T" 단축키를 사용하거나 애플리케이션 메뉴에서 "터미널"을 찾아 실행합니다.

2.  **명령어 입력**: 터미널 창에 다음 명령어를 입력하고 Enter 키를 누릅니다.

    ```bash
    openssl rand -base64 64
    ```

3.  **결과 확인**: 명령어를 실행하면 64바이트의 난수가 Base64로 인코딩된 문자열이 화면에 출력됩니다. 이 문자열이 바로 키입니다.

-----

### **윈도우 (PowerShell 또는 CMD)**

윈도우는 기본적으로 OpenSSL이 설치되어 있지 않으므로 먼저 설치해야 합니다.

1.  **OpenSSL 설치**:

      * Shining Light Productions와 같은 신뢰할 수 있는 소스에서 윈도우용 OpenSSL 설치 파일을 다운로드합니다.
      * 설치 파일을 실행하고 안내에 따라 설치를 진행합니다. 이 때 **"the OpenSSL binaries to the PATH"** 옵션을 선택하여 환경 변수에 자동으로 경로를 추가하면 편리합니다.
      * 만약 환경 변수 설정을 놓쳤다면, 수동으로 OpenSSL 실행 파일(`openssl.exe`)이 있는 경로(예: `C:\Program Files\OpenSSL-Win64\bin`)를 시스템 PATH 환경 변수에 추가해야 합니다.

2.  **명령어 실행**:

      * PowerShell 또는 명령 프롬프트(CMD)를 엽니다.
      * 다음 명령어를 입력하고 Enter 키를 누릅니다.

    <!-- end list -->

    ```bash
    openssl rand -base64 64
    ```

3.  **결과 확인**: macOS 및 리눅스와 동일하게, Base64로 인코딩된 난수 문자열이 출력됩니다.