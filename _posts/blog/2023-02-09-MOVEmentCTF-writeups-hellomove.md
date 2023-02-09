---
title: 2022 MOVEment CTF Writeups
categories: [web3, move]
tags: [web3, ctf, move, aptos]
date: 2023-02-10 00:20:00 +0900
---
# 1. Info
2022년 12월에 [movebit](https://twitter.com/MoveBit_)이 주체하는 [CTF Movement](https://ctfmovement.movebit.xyz/)가 열렸었다. 
물론 나는 뒤늦게 이런 CTF가 열린것을 확인하였고, devnet에 문제 파일이 그대로 보존되고 있어서 때늦은 풀이를 해보고 있다.
[문제 난이도](https://ctfmovement.movebit.xyz/challenges)는 꽤 다양하게 나온듯 해서, move languae와 aptos가 초심인 나에게는 적절하고 재밌는 문제들이었다.
![2022_ctf_movement_challenges](/posts/2022_ctf_movement_challenges.png)

# 2. Writeups: `Hello Move`
- 문제 내용
  - This is a challenge that combines three questions, come to try it. Follow the steps below to complete the challenge. The goal is calling the get_flag() function to trigger a Flag event, and submit the transaction hash to get the flag. You can reach the contract code here: [movebit/ctfmovement-2](https://github.com/movebit/ctfmovement-2).
  - Deployment Contract: [0xc400473d4225e27a1b7a934d05bfe31a746605a59cf13928427f123238cf8f26::hello_move](https://fullnode.devnet.aptoslabs.com/v1/accounts/0xc400473d4225e27a1b7a934d05bfe31a746605a59cf13928427f123238cf8f26/module/hello_move)
  - Deployment Hash: [0xb4270980f341a0d98eaae2885a80dc48007c9d0b25d3439fb31db484ae042083](https://fullnode.devnet.aptoslabs.com/v1/transactions/by_hash/0xb4270980f341a0d98eaae2885a80dc48007c9d0b25d3439fb31db484ae042083)
  - Source Code
    ```move
    module ctfmovement::hello_move {
        // use std::hash;
        // use std::vector;
        // use sui::event;
        use std::signer;
        use std::vector;

        use aptos_std::aptos_hash;
        use aptos_framework::event::{Self, EventHandle};
        use aptos_framework::account;

        const Initialize_balance : u8 = 10;

        struct Challenge has key, store {
            balance: u8,
            q1: bool,
            q2: bool,
            q3: bool,
            flag_event_handle: EventHandle<Flag>
        }

        struct Flag has store, drop {
            user: address,
            flag: bool
        }

        public fun init_challenge(account: &signer) {
            let addr = signer::address_of(account);
            let handle =  account::new_event_handle<Flag>(account);
            assert!(!exists<Challenge>(addr), 0);
            move_to(account, Challenge {
                balance: Initialize_balance,
                q1: false,
                q2: false,
                q3: false,
                flag_event_handle: handle
            })
        }

        entry public fun hash(account: &signer, guess: vector<u8>) acquires Challenge{
            let borrow_guess = &mut guess;
            assert!(vector::length(borrow_guess) == 4, 0);
            vector::push_back(borrow_guess, 109);
            vector::push_back(borrow_guess, 111);
            vector::push_back(borrow_guess, 118);
            vector::push_back(borrow_guess, 101);

            if (aptos_hash::keccak256(guess) == x"d9ad5396ce1ed307e8fb2a90de7fd01d888c02950ef6852fbc2191d2baf58e79") {
                let res = borrow_global_mut<Challenge>(signer::address_of(account));
                if (!res.q1) {
                    res.q1 = true;
                }
            }
        }

        public entry fun discrete_log(account: &signer, guess: u128) acquires Challenge {
            if (pow(10549609011087404693, guess, 18446744073709551616) == 18164541542389285005) {
                let res = borrow_global_mut<Challenge>(signer::address_of(account));
                if (!res.q2) {
                    res.q2 = true;
                }
            }
        }

        public entry fun add(account: &signer, choice: u8, number: u8) acquires Challenge {
            let res = borrow_global_mut<Challenge>(signer::address_of(account));
            assert!(number <= 5, 0);
            if (choice == 1) {
                res.balance = res.balance + number;
            } else if (choice == 2) {
                res.balance = res.balance * number;
            } else if (choice == 3) {
                res.balance = res.balance << number;
            };

            if (!res.q3 && res.balance < Initialize_balance) {
                res.q3 = true;
            }
        }

        public entry fun get_flag(account: &signer) acquires Challenge {
            let addr = signer::address_of(account);
            let res = borrow_global_mut<Challenge>(addr);
            if (res.q1 && res.q2 && res.q3) {
                event::emit_event(&mut res.flag_event_handle, Flag {
                    user: signer::address_of(account),
                    flag: true
                })
            }
        }

        public fun pow(g: u128, x: u128, p: u128): u128 {
            let ans = 1;
            g = g % p;
            while (x != 0) {
                if ((x & 1) == 1) {
                    ans = ((ans % p) * (g % p)) % p;
                };
                x = x >> 1;
                g = (g * g) % p;
            };
            ans
        }
    }
    ```
- 풀이
  - 푸는중..
