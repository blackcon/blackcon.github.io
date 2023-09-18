---
title: Miniconda 설치 및 사용법 
categories: [Tools, Miniconda]
tags: [Python, Miniconda]
date: 2023-09-18 12:50:0 +0900
---
# Miniconda 란?
- Python을 사용하는 방법은 매우 다양하다. local system에 python을 설치한 후 그냥 실행을 한다거나, `venv`를 실행한다거나, `[anaconda](https://anaconda.org/anaconda/python)`를 사용하거나.
- 이 중에서 `Miniconda`라는 프로그램을 사용하는 방법도 존재한다.
- 하여 Miniconda란, `[anaconda](https://anaconda.org/anaconda/python)` 를 최소화한 프로그램으로써 작업하고자하는 프로젝트별로 python의 버전을 설정할 수 있으며, 모듈 또한 각 프로젝트 환경에 종속되어 설치할 수 있다.
- 이러한 기능으로 프로젝트별 모듈 관리하기가 편하며, 간단한 명령어를 이용하여 project switching 하기도 용이하다.

# Miniconda 설치법
## 1. 설치 스크립트 다운로드
- [다운로드 사이트](https://docs.conda.io/projects/miniconda/en/latest/)에서 설치하고자 하는 환경을 다운로드받으면 된다. Windows, MacOS, Linux 모두 지원하고 있다.
- Mac OS M1
    ```bash
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-arm64.sh
    ```
- MacOS Intel
    ```bash
    wget https://repo.anaconda.com/miniconda/Miniconda3-latest-MacOSX-x86_64.sh
    ```
- Linux
    ```bash
    wget https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh
    ```
## 2. 설치 스크립트에 실행권한 주기
```bash
chmod +x Miniconda3-latest-Linux-x86_64.sh
```
## 3. 설치 스크립트 실행하기
- 스크립트 실행
    ```bash
    ./Miniconda3-latest-Linux-x86_64.sh
    ```
- 설치 시작
    ```bash
    $ ./Miniconda3-latest-Linux-x86_64.sh

    Welcome to Miniconda3 py311_23.5.2-0

    In order to continue the installation process, please review the license
    agreement.
    Please, press ENTER to continue
    >>> [엔터 입력하기]
    ```
- Licence 동의
    ```bash
    In order to continue the installation process, please review the license
    agreement.
    Please, press ENTER to continue
    >>>
    ======================================
    End User License Agreement - Miniconda
    ======================================

    Copyright 2015-2023, Anaconda, Inc.

    All rights reserved under the 3-clause BSD License:

    This End User License Agreement (the "Agreement") is a legal agreement between you and
    Anaconda, Inc. ("Anaconda") and governs your use of Miniconda.

    ... 중략 ...

    Last updated March 21, 2022


    Do you accept the license terms? [yes|no]
    [no] >>>  yes 
    ```
- Miniconda 설치경로 지정하기
    ```bash
    Miniconda3 will now be installed into this location:
    /Users/user/miniconda3

    - Press ENTER to confirm the location
    - Press CTRL-C to abort the installation
    - Or specify a different location below

    [/Users/user/miniconda3] >>>
    ```
- Miniconda 초기화 하기
    ```bash
    Do you wish the installer to initialize Miniconda3
    by running conda init? [yes|no]
    [no] >>> yes
    no change     /Users/user/miniconda3/condabin/conda
    no change     /Users/user/miniconda3/bin/conda
    no change     /Users/user/miniconda3/bin/conda-env
    no change     /Users/user/miniconda3/bin/activate
    no change     /Users/user/miniconda3/bin/deactivate
    no change     /Users/user/miniconda3/etc/profile.d/conda.sh
    no change     /Users/user/miniconda3/etc/fish/conf.d/conda.fish
    no change     /Users/user/miniconda3/shell/condabin/Conda.psm1
    no change     /Users/user/miniconda3/shell/condabin/conda-hook.ps1
    no change     /Users/user/miniconda3/lib/python3.11/site-packages/xontrib/conda.xsh
    no change     /Users/user/miniconda3/etc/profile.d/conda.csh
    modified      /Users/user/.bashrc

    ==> For changes to take effect, close and re-open your current shell. <==

    If you'd prefer that conda's base environment not be activated on startup,
    set the auto_activate_base parameter to false:

    conda config --set auto_activate_base false

    Thank you for installing Miniconda3!
    ```
- .bashrc 최신화 하기 (이 작업을 해야 conda 명령어를 사용할 수 있다.)
    ```bash
    $ source ~/.bashrc
    (base) $ conda
    usage: conda [-h] [-V] command ...

    conda is a tool for managing and deploying applications, environments and packages.
    ```

# 4. Conda 명령어 사용해보기.
## 4.1) 신규 프로젝트 생성
- 기본 명령기
    ```bash
    (base) ➜  ~ conda create -n {프로젝트 이름} python={파이썬 버전}
    ```
- 생성해보기
    ```bash
    (base) ➜  ~ conda create -n conda-test python=3.11
    Retrieving notices: ...working... done
    Collecting package metadata (current_repodata.json): done
    Solving environment: done


    ==> WARNING: A newer version of conda exists. <==
    current version: 23.1.0
    latest version: 23.7.4

    Please update conda by running

        $ conda update -n base -c defaults conda

    Or to minimize the number of packages updated during conda update use

        conda install conda=23.7.4



    ## Package Plan ##

    environment location: /Users/user/miniconda3/envs/conda-test

    added / updated specs:
        - python=3.11


    The following packages will be downloaded:

        package                    |            build
        ---------------------------|-----------------
        python-3.11.5              |       hb885b13_0        15.4 MB
        ------------------------------------------------------------
                                            Total:        15.4 MB

    The following NEW packages will be INSTALLED:

    bzip2              pkgs/main/osx-arm64::bzip2-1.0.8-h620ffc9_4
    ca-certificates    pkgs/main/osx-arm64::ca-certificates-2023.08.22-hca03da5_0
    libffi             pkgs/main/osx-arm64::libffi-3.4.4-hca03da5_0
    ncurses            pkgs/main/osx-arm64::ncurses-6.4-h313beb8_0
    openssl            pkgs/main/osx-arm64::openssl-3.0.10-h1a28f6b_2
    pip                pkgs/main/osx-arm64::pip-23.2.1-py311hca03da5_0
    python             pkgs/main/osx-arm64::python-3.11.5-hb885b13_0
    readline           pkgs/main/osx-arm64::readline-8.2-h1a28f6b_0
    setuptools         pkgs/main/osx-arm64::setuptools-68.0.0-py311hca03da5_0
    sqlite             pkgs/main/osx-arm64::sqlite-3.41.2-h80987f9_0
    tk                 pkgs/main/osx-arm64::tk-8.6.12-hb8d0fd4_0
    tzdata             pkgs/main/noarch::tzdata-2023c-h04d1e81_0
    wheel              pkgs/main/osx-arm64::wheel-0.38.4-py311hca03da5_0
    xz                 pkgs/main/osx-arm64::xz-5.4.2-h80987f9_0
    zlib               pkgs/main/osx-arm64::zlib-1.2.13-h5a0b063_0


    Proceed ([y]/n)? y

    Downloading and Extracting Packages

    Preparing transaction: done
    Verifying transaction: done
    Executing transaction: done
    #
    # To activate this environment, use
    #
    #     $ conda activate conda-test
    #
    # To deactivate an active environment, use
    #
    #     $ conda deactivate
    ```
## 4.2) 생성된 프로젝트 보기
- 기본 명령어
    ```bash
    (base) ➜  ~ conda env list
    ```
- 실행 결과
    ![miniconda-env-list](/posts/2023-09-18-miniconda-env-list.png)

## 4.3) 프로젝트 활성화(전환)
- 기본 명령어
    ```bash
    (base) ➜  ~ conda activate conda-test
    ```
- 실행 결과 (명령 프롬프트를 보면 `base`에서 `conda-test`로 변경된것을 확인할 수 있음)
    ```
    (conda-test) ➜  ~ python --version
    Python 3.11.5
    ```

## 4.4) 프로젝트 비활성화
- 기본 명령어 및 실행 결과
    ```bash
    (conda-test) ➜  ~ conda deactivate
    (base) ➜  ~ python --version
    Python 3.10.
    ```

## 4.5) 프로젝트 삭제
- 프로젝트명으로 삭제하기 (option: `-n`)
    ```bash
    (base) ➜  ~ conda-env remove -n conda-test

    Remove all packages in environment /Users/user/miniconda3/envs/conda-test:
    ```
- 프로젝트 경로로 삭제하기 (option: `-p`)
    ```bash
    (base) ➜  ~ conda env list # 삭제할 프로젝트가 저장된 path 찾기
    # conda environments:
    #
    conda-test               /Users/user/miniconda3/envs/conda-test
    (base) ➜  ~ conda-env remove -p /Users/user/miniconda3/envs/conda-test # path 삭제하기

    Remove all packages in environment /Users/user/miniconda3/envs/conda-test:
    ```
