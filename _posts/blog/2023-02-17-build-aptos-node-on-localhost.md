---
title: Localhost에 Aptos Node 실행하기
categories: [web3, aptos]
tags: [web3, move, aptos]
date: 2023-02-17 23:17:00 +0900
---
# Intro
요즘들어 [Aptos 블록체인](https://aptoslabs.com/)을 열심히 공부해보고 있습니다. Layer1 블록체인이며 흔히들 말하는 Smart Contract는 Move라는 언어로 작성되고요. 코딩하기에 앞서서 테스트를 하기 위한 네트워크를 Localhost에 구성해보았고 그 절차를 정리해보았습니다.

# How to build network on localhost? ([참고](https://aptos.dev/nodes/local-testnet/run-a-local-testnet/))
1. [aptos-core](https://github.com/aptos-labs/aptos-core)의 repository를 clone합니다.
   ```bash
   git clone https://github.com/aptos-labs/aptos-core.git
   ```
2. `aptos-core` directory 로 이동하고,
   ```bash
   cd aptos-core
   ```
3. 아래와 같이 `scripts/dev_setup.sh` 파일을 실행시켜 줍니다.
   ```bash
   ./scripts/dev_setup.sh
   ```
4. cargo env 파일이 최신화되었으므로 source 커멘드로 적용시켜줍니다.
   ```bash
   source ~/.cargo/env
   ```
5. `cargo`를 이용하여 소스코드를 빌드하고 실행합니다.
   ```bash
   CARGO_NET_GIT_FETCH_WITH_CLI=true cargo run -p aptos-node -- --test # 소스코드 수정이 이루어질 때마다 이 명령어로 build 및 실행 해주기
   
   [Output]
       Finished dev [unoptimized + debuginfo] target(s) in 0.64s
        Running `target/debug/aptos node run-local-testnet --with-faucet --faucet-port 8081 --force-restart --assume-yes`
   Completed generating configuration:
      Log file: "/Users/user/works/personal/aptos/.aptos/testnet/validator.log"
      Test dir: "/Users/user/works/personal/aptos/.aptos/testnet"
      Aptos root key path: "/Users/user/works/personal/aptos/.aptos/testnet/mint.key"
      Waypoint: 0:a1df7bb0778100a5161c533000571f54d230ef76d666e34f9c0baa66337112bb
      ChainId: testing
      REST API endpoint: http://0.0.0.0:8080
      Metrics endpoint: http://0.0.0.0:9101/metrics
      Aptosnet Fullnode network endpoint: /ip4/0.0.0.0/tcp/6181

   Aptos is running, press ctrl-c to exit

   Faucet is running.  Faucet endpoint: 0.0.0.0:8081
   ```
   ![build-aptos-network.png](/posts/build-aptos-network.png)

6. 사용자들에게 테스트용 APTs를 제공해줄 수 있도록 Faucet 도 연결해줍니다.
   - 5번 과정을 무사히 완료하였다면 위 로그에서 `Aptos root key path`라는 내용이 있으며,
   - 이 경로를 복사해서 `--mint-key-file-path`의 옵션 값으로 입력합니다. (나머지 옵션은 아래와 동일)
     ```bash
     cargo run --package aptos-faucet -- \
         --chain-id TESTING \
         --mint-key-file-path /Users/user/works/personal/aptos/.aptos/testnet/mint.key \
         --address 0.0.0.0 \
         --port 80 \
         --server-url http://127.0.0.1:8080
     ```
    ![build-aptos-faucet.png](/posts/build-aptos-faucet.png)

7. 잘 구축되었는지 테스트 해보기
   - Localnet에 계정 생성
      ```shell
      aptos init --assume-yes --network custom --rest-url http://localhost:8080 --faucet-url http://localhost:8081
      ```
      ```text
      Configuring for profile default
      Configuring for network Custom
      Using command line argument for rest URL http://localhost:8080/
      Using command line argument for faucet URL http://localhost:8081/
      Enter your private key as a hex literal (0x...) [Current: None | No input: Generate new key (or keep one if present)]

      No key given, generating key...
      Account c0f351a65435de74f7eb6d27920dcb8f3e233ac8d4a016eae83b1f9151f24679 doesn't exist, creating it and funding it with 100000000 Octas
      Account c0f351a65435de74f7eb6d27920dcb8f3e233ac8d4a016eae83b1f9151f24679 funded successfully

      ---
      Aptos CLI is now set up for account c0f351a65435de74f7eb6d27920dcb8f3e233ac8d4a016eae83b1f9151f24679 as profile default!  Run `aptos --help` for more information about commands
      {
      "Result": "Success"
      }
      ```
   - Faucet으로 테스트 APTs 발급 받기
      ```shell
      aptos account fund-with-faucet --url http://localhost:8080 --faucet-url http://localhost:8081 --account 0xc0f351a65435de74f7eb6d27920dcb8f3e233ac8d4a016eae83b1f9151f24679
      ```
      ```text
      {
        "Result": "Added 100000000 Octas to account ff4fc660a1f2f36cd63ecd5e850b26f04daaee15d982b6299933001049331b88"
      }
      ```

