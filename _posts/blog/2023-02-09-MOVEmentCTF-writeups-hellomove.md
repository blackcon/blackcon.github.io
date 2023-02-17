---
title: (작성중) 2022 MOVEment CTF Writeups (Hello Move)
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
- Source Code (move 언어이지만 highlight 지원이 안되어 rust로 표기함)
  ```rust
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
          let borrow_guess = &mut guess;     // 인자로 전달받은 guess를 borrow_guess로 전달한다.
          assert!(vector::length(borrow_guess) == 4, 0);  // guess의 길이는 4글자라는 것을 알 수 있다.
          vector::push_back(borrow_guess, 109); // borrow_guess 백터 뒤에 push_back을 하여 109를 입력한다. (ascii: 'm')
          vector::push_back(borrow_guess, 111); // borrow_guess 백터 뒤에 push_back을 하여 111를 입력한다. (ascii: 'o')
          vector::push_back(borrow_guess, 118); // borrow_guess 백터 뒤에 push_back을 하여 118를 입력한다. (ascii: 'v')
          vector::push_back(borrow_guess, 101); // borrow_guess 백터 뒤에 push_back을 하여 101를 입력한다. (ascii: 'e')

          // guess는 [?,?,?,?,m,o,v,e] 로 추정되며, 
          // hash값이 'd9ad5396ce1ed307e8fb2a90de7fd01d888c02950ef6852fbc2191d2baf58e79'인 값을 찾아야 한다.
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
          assert!(number <= 5, 0);  // number는 5 이하의 숫자입니다.
          if (choice == 1) {
              res.balance = res.balance + number;
          } else if (choice == 2) {
              res.balance = res.balance * number;
          } else if (choice == 3) {
              res.balance = res.balance << number;
          };

          if (!res.q3 && res.balance < Initialize_balance) { // Initialize_balance값은 10이며,
                                                             // res.balance가 Initialize_balance 보다 작아야한다.
                                                             // 위 choice 조건문 중에서 2번을 택하면 해결된다.
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
  - 이번에는 `checkin`문제와 달리 get_flag를 하기 위해서 `res.q1`, `res.q2`, `res.q3` 모두 true여야 한다.
  - 각 변수의 value를 true로 만들기 위해서는 다른 함수들의 문제들을 풀어야 할 듯 하다.
    - Step1) res.q1을 true로 바꾸기 (`hash`함수에 주석을 달아 두었다.)
      ```python
      import sha3

      for i in range(256):
          for j in range(256):
              for k in range(256):
                  for l in range(256):
                      guess = bytes([i,j,k,l,109,111,118,101])
                      if sha3.keccak_256(guess).hexdigest() == "d9ad5396ce1ed307e8fb2a90de7fd01d888c02950ef6852fbc2191d2baf58e79":
                          print( guess )
                          exit()
      # answer: goodmove
      ```
    - Step2) res.q2를 true로 바꾸기
      ```python
      from sympy.ntheory import discrete_log
      result = discrete_log(18446744073709551616, 18164541542389285005, 10549609011087404693)  
      print(result)
      answer: result = 3123592912467026955
      ```
    - Step3) res.q3를 true로 바꾸기 (`add`함수에 주석을 달아 두었다.)
      ```python
      def solv_add( choice, number ):
          init_bal = 10
          bal = 10
          if choice == 1:
              bal = bal + number
          elif choice == 2:
              bal = bal * number
          elif choice == 3:
              bal = bal << number

          if bal < init_bal:
              print( choice, number  )

      for i in [1,2,3]:
          for number in range(0, 6):
              solv_add(i, number)

      # answer: choice = 2, number = 0
      ```
  - 실제 모듈로 값을 전달하기
    > 앞어서 res.q1 ~ res.q3를 true로 셋팅할 수 있는 값을 찾아냈다. 이 값들을 문제의 module로 호출해야하는데, 이에 앞어 할 일이 하나 있다. 위 문제코드를 보면 `init_challenge`라는 함수가 있는데 이 함수를 호출하여 challnger를 활성화 시켜주어야 한다. 하지만 `init_challenge`는 `entry` 키워드가 없는 함수이기에 `aptos move run` 명령어로 직접적으로 호출을 할 수 없다.
       
