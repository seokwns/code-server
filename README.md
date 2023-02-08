# code-server
code-server 설치 과정입니다.







### 서론

노트북을 들고다니기엔 무리가 있어 아이패드로 코딩을 해야하는 상황이 왔습니다. 앱스토어에 있던 'code' 라는 앱은 데스크탑의 vs-code와 상당히 비슷한 환경이었으나 모바일-PC 환경 전환이 용이하지 못하였고, 버그가 많아서 사용하는데 불편함이 많았습니다. 그래서 아이패드에서 코딩을 할 수 있는 환경 구축을 위해 처음 생각한 내용은 '원격 데스크탑' 프로그램을 이용하는 것이었습니다.

원격 데스크탑은 집에 있는 PC를 그대로 사용가능 하였지만, 네트워크 연결이 매끄럽지 못하였고 해상도 문제가 심하여 사용하기 힘들었습니다. 그래서 다른 방법을 찾게된게 바로 code-server를 통한 vs-code 사용하기입니다.

code-server를 사용하면, 웹에서 vs-code를 즉각적으로 사용가능하고, 모바일-PC 환경 전환이 용이합니다. 또한 마이크로소프트에서 지원하는 vs-code 웹 버전에서는 지원하지 않는 extension들도 수동으로 설치가 가능하여 PC 환경과 거의 동일하게 구동이 가능하였습니다.



설치 환경은 다음과 같습니다.

- 구글 클라우드 플랫폼 VM 인스턴스
  - 머신 유형: e2-micro (AMD Rome / x86/64)
  - 메모리 1GB
  - 30GB 디스크

- OS: Ubuntu 22.04 LTS







### 설치

저는 간편한 설치스크립트를 사용하였습니다.

우선 기본 패키지 업데이트를 진행하였습니다.

```bash
sudo apt-get update
sudo apt-get upgrade
```

업데이트 후 code-server를 설치하였습니다.

```bash
curl -fsSL https://code-server.dev/install.sh | sh
```

이로써 code-server의 설치는 완료하였습니다. 다음으로는 방화벽 설정입니다. 어디서든 접근이 가능해야 하므로 포트를 열어줍니다.

```bash
sudo iptables -I INPUT 5 -i ens3 -p tcp --dport 80 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -I INPUT 5 -i ens3 -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
sudo iptables -I INPUT 5 -i ens3 -p tcp --dport 8080 -m state --state NEW,ESTABLISHED -j ACCEPT
```

기본적으로 HTTP(80), HTTPS(443), code-server에서 사용할 포트(8080) 를 열어줍니다.







### code-server 설정

code-server를 서비스로 실행하기 위해 systemctl로 enable 시킵니다.

```bash
sudo systemctl enable --now code-server@$USER
```

외부 접속을 위해서는 config.yaml 을 수정해야 합니다.

```bash
sudo vim ~/.config/code-server/config.yaml
```

내용을 아래와 같이 수정합니다.

```yaml
bind-addr: 0.0.0.0:8080
auth: password
password: 비밀번호
cert: false
```

설정을 완료하였으면 서비스를 재시작 해줍니다. 재시작 이후 아이피:포트번호 로 접속이 가능해집니다.

```bash
sudo systemctl restart --now code-server@$USER
```

이로써 기본적인 구동과정은 마무리 됩니다. 하지만 데이터 암호화를 위해 cert를 설정할 필요가 있습니다.







### cert 보안 설정

config.yaml 파일에서 cert를 true로 변경해줍니다.

```yaml
bind-addr: 0.0.0.0:8080
auth: password
password: 비밀번호
cert: true
```

저의 경우, 위의 방법으로는 구동되지 않았습니다. 

"error RSA PRIVATE KEY not found from openssl output" 이라는 오류가 뜨면서 code-server를 실행할 수 없었습니다. 

[[Bug\]: error RSA PRIVATE KEY not found from openssl output · Issue #5162 · coder/code-server (github.com)](https://github.com/coder/code-server/issues/5162)

저와 동일한 에러 환경에 있는 해당 깃허브 이슈를 발견하여

https://github.com/coder/code-server/issues/5162#issuecomment-1214183806

위 방법을 적용하기로 했습니다. mkcert 깃허브로 가서 설치를 해주었습니다.

```bash
sudo apt install libnss3-tools
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
sudo cp mkcert-v*-linux-amd64 /usr/local/bin/mkcert
sudo pacman -Syu mkcert
```

완료되었습니다. mkcert 설정을 시작합니다.

```bash
mkcert -install
mkcert {your_machine_ip} 127.0.0.1
```

이후 config.yaml을 수정합니다.

```bash
nano ~/.config/code-server/config.yaml
```

이후 아래 부분을 추가합니다.

```
cert: path_to_your_cert_folder/cert_name.pem
cert-key: path_to_your_cert_folder/cert_name-key.pem
```

여기서 파일 경로가 중요합니다. bash에서 code-server로 실행할 경우,  /home/$user/..... 의 절대경로가 아닌 상대경로로 입력해야 합니다. 하지만, systemctl로 실행해 둘 경우에는 절대경로로 적어야 합니다. 저는 백그라운드에서 계속 실행하기 위해서 절대경로를 입력해주었습니다. (해당 오류를 sudo systemctl status code-server@$USER 을 지속적으로 관찰하면서 알게 되었습니다. 직접 실행할 땐 아무 문제 없었지만, systemctl 로 이동시키는 순간 cert 키 파일을 찾을 수 없다는 오류가 나타나 절대경로로 수정해주어 해결하였습니다.)

이후 code-server을 재시작하면 됩니다.



cert 발급과 관련된 정보는 금방 찾았고 이렇게 깃허브 이슈를 통해 해결하는 과정이 처음이라 생소하였지만 하나하나 배우는 의미가 쏠쏠한 시간들이었습니다.







### extension 수동설치

code-server로 구동시킨 vs-code에서는 c/c++ extension을 사용할 수 없습니다. 따라서 보다 쉬운 컴파일 환경을 구축하기 위해서 수동으로 설치하였습니다.

우선 c/c++ extension 깃허브로 가서 배포 자료 중 리눅스 환경의 자료를 다운받고 옮겨줍니다.

[microsoft/vscode-cpptools: Official repository for the Microsoft C/C++ extension for VS Code. (github.com)](https://github.com/microsoft/vscode-cpptools)

위 링크에서 다운받았습니다. 해당 운영체제에 맞는 파일을 다운받으면 됩니다. 이 후 code-server 명령어를 통해 설치하면 됩니다.

```bash
code-server --install-extension <vsix 파일 위치>
```

이외 다른 extension 또한 동일한 방법으로 설치하면 적용됩니다.
