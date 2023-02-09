---
title: 2022 MOVEment CTF Writeups
categories: [web3, move]
tags: [web3, ctf, move, aptos]
date: 2023-02-09 23:54:00 +0900
---
# 1. Info
2022년 12월에 [movebit](https://twitter.com/MoveBit_)이 주체하는 [CTF Movement](https://ctfmovement.movebit.xyz/)가 열렸었다. 
물론 나는 뒤늦게 이런 CTF가 열린것을 확인하였고, devnet에 문제 파일이 그대로 보존되고 있어서 때늦은 풀이를 해보고 있다.
[문제 난이도](https://ctfmovement.movebit.xyz/challenges)는 꽤 다양하게 나온듯 해서, move languae와 aptos가 초심인 나에게는 적절하고 재밌는 문제들이었다.
![2022_ctf_movement_challenges](/posts/2022_ctf_movement_challenges.png)

# 2. Writeups
## 1) checkin
- 문제 내용
  - This is a simple challenge, follow the steps below to complete the challenge. The goal is calling the `get_flag()` function to trigger a Flag event, and submit the transaction hash to get the flag. You can reach the contract code here: [movebit/ctfmovement-1](https://github.com/movebit/ctfmovement-1).
  - Deployment Contract: [0xdec0b6cf75d38b9da4922cc983810ef436560bbe55e85f2b80d657ff834b3c9f::checkin](https://fullnode.devnet.aptoslabs.com/v1/accounts/0xdec0b6cf75d38b9da4922cc983810ef436560bbe55e85f2b80d657ff834b3c9f/module/checkin)
  - Deployment Hash: [0x7f8d39b633c291e498fa490be34db086082e5b52b7050d1988df904f168cfc0f](https://fullnode.devnet.aptoslabs.com/v1/transactions/by_hash/0x7f8d39b633c291e498fa490be34db086082e5b52b7050d1988df904f168cfc0f)
  - Source code
    ```move
    module ctfmovement::checkin {
        use std::signer;
        use aptos_framework::account;
        use aptos_framework::event;

        struct FlagHolder has key {
            event_set: event::EventHandle<Flag>,
        }

        struct Flag has drop, store {
            user: address,
            flag: bool
        }

        public entry fun get_flag(account: signer) acquires FlagHolder {
            let account_addr = signer::address_of(&account);
            if (!exists<FlagHolder>(account_addr)) {
                move_to(&account, FlagHolder {
                    event_set: account::new_event_handle<Flag>(&account),
                });
            };

            let flag_holder = borrow_global_mut<FlagHolder>(account_addr);
            event::emit_event(&mut flag_holder.event_set, Flag {
                user: account_addr,
                flag: true
            });
        }
    }
    ```
- 풀이
  - Step1) aptos에서 사용할 Account를 생성한다.
    ```bash
    aptos init --assume-yes --network custom --rest-url https://fullnode.devnet.aptoslabs.com --faucet-url https://faucet.devnet.aptoslabs.com
    ```
    ```
    Aptos CLI is now set up for account 1405b44526f5681853f6c4bfac983e950f8536f392491fd02213c235c11d67fb as profile default!  Run `aptos --help` for more information about commands
    {
      "Result": "Success"
    }
    ```
  - Step2) 테스트 코인(faucet)을 발급받는다.
    ```bash
    aptos account fund-with-faucet --url https://fullnode.devnet.aptoslabs.com --faucet-url https://faucet.devnet.aptoslabs.com --account
    ```
    ```
    {
      "Result": "Added 100000000 Octas to account d831a5e9b93c7bcaab0c5c8ae7a887a6daa2dee9972676456bda986a130fecad"
    }
    ```
  - Step3) 문제에서 제공한 contract의 `get_flag()` 함수를 호출한다. 위 소스코드를 보다시피 `get_flag()`만 호출하면 다른 조건 없이 `emit_event`가 수행된다.
    ```bash
    aptos move run --function-id '0xdec0b6cf75d38b9da4922cc983810ef436560bbe55e85f2b80d657ff834b3c9f::checkin::get_flag' --assume-yes
    ```
    ```
    {
      "Result": {
        "transaction_hash": "0xe7c31b92eca1a21d9dc757dc253e19548c8b3ae1a726617841e47a43317ea59a",
        "gas_used": 909,
        "gas_unit_price": 100,
        "sender": "1405b44526f5681853f6c4bfac983e950f8536f392491fd02213c235c11d67fb",
        "sequence_number": 0,
        "success": true,
        "timestamp_us": 1675952688137771,
        "version": 14127201,
        "vm_status": "Executed successfully"
      }
    }
    ```
  - Step4) transaction_hash 를 문제 사이트에 입력을 하면 아래와 같이 정답(flag)를 알려준다.
    ![2022_ctf_movement_checkin_1](/posts/2022_ctf_movement_checkin_1.png)
    ![2022_ctf_movement_checkin_2](/posts/2022_ctf_movement_checkin_2.png)

# 3. To be continue
- Next time...
