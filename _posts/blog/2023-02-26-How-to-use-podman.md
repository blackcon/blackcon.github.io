---
title: Docker를 대신해줄 귀여운 명령어 Podman
categories: [podman, manual]
tags: [podman, docker, manual]
date: 2023-02-26 18:05:00 +0900
---
# Intro
![podman-logo](https://podman.io/images/podman.svg)<br>
최근들어 docker를 사용한 작업이 많아진 가운데 회사에서 licence의 문제로 docker desktop을 지우라는 명(?)을 전달받았습니다.
이에따라 docker를 삭제해야 했고, docker 관련 테스팅을 하기 위해선 개인 PC에서 깔짝깔짝 테스트를 해야하는 번거로움이 있었는데요.
그러던 중 [podman](https://podman.io/)이라는 tool을 알게되었고 관련하여 기본적인 셋팅 및 간단한 명령어를 소개하고자 합니다! 

# Install `podman`
해당 포스팅에서는 `MacOS`를 기반으로 하여 설명을 진행할것이고,,, 다른 platform을 사용한다면 아래 링크를 참고하시길 바랍니다 :)
- MacOS
  ```
  brew install podman
  ```
- Linux: [click](https://podman.io/getting-started/installation#installing-on-linux)
- Windows: [click](https://github.com/containers/podman/blob/main/docs/tutorials/podman-for-windows.md)

# Commands
podman의 전체 명령어를 알고싶다면 `podman help`를 입력하여 옵션 및 설명들을 확인할 수 있어요. (블로그 하단 참고)
이 포스팅에서 모든 명령어에 대해 다루기엔 너무 많으니, 초기에 기본적으로 사용해볼 명령어 몇 가지를 소개드려 보겠습니다.
- `podman machine init`
  - podman을 사용하기 위해 vm을 실행하여아 하는데 이 때의 초기설정을 해주는 명령어입니다.
  - 명령어
    ```bash
    ➜ podman machine init

    Downloading VM image: fedora-coreos-37.20230218.2.0-qemu.aarch64.qcow2.xz: done
    Extracting compressed file
    Image resized.
    Machine init complete
    To start your machine run:

      podman machine start
    ```
- `podman machine start`
  - `init`이 완료되었다면 vm을 시작해줘야 하는데요. 이 때 사용하는 명령어가 바로 `machine start`입니다.
  - 명령어
    ```bash
    ➜ podman machine start

    Starting machine "podman-machine-default"
    Waiting for VM ...
    Mounting volume... /Users:/Users
    Mounting volume... /private:/private
    Mounting volume... /var/folders:/var/folders

    This machine is currently configured in rootless mode. If your containers
    require root permissions (e.g. ports < 1024), or if you run into compatibility
    issues with non-podman clients, you can switch using the following command:

      podman machine set --rootful

    API forwarding listening on: /Users/user/.local/share/containers/podman/machine/podman-machine-default/podman.sock

    The system helper service is not installed; the default Docker API socket
    address can't be used by podman. If you would like to install it run the
    following commands:

      sudo /opt/homebrew/Cellar/podman/4.4.2/bin/podman-mac-helper install
      podman machine stop; podman machine start

    You can still connect Docker API clients by setting DOCKER_HOST using the
    following command in your terminal session:

      export DOCKER_HOST='unix:///Users/user/.local/share/containers/podman/machine/podman-machine-default/podman.sock'

    Machine "podman-machine-default" started successfully
    ```
- `podman system connection list`
  - 현재 podman vm에 connection된 목록들을 볼 수 있어요.
  - 명령어
    ```bash
    ➜ podman system connection list
    
    Name                         URI                                                         Identity                                 Default
    podman-machine-default       ssh://core@localhost:51678/run/user/501/podman/podman.sock  /Users/user/.ssh/podman-machine-default  true
    podman-machine-default-root  ssh://root@localhost:51678/run/podman/podman.sock           /Users/user/.ssh/podman-machine-default  false
    ```
- `podman pull`
  - docker의 pull과 동일하고 hub에 존재하는 image를 당겨오는 명령어입니다. (아래는 예시_)
  - 명령어
    ```bash
    ➜ podman pull mythril/myth
    
    Resolving "mythril/myth" using unqualified-search registries (/etc/containers/registries.conf.d/999-podman-machine.conf)
    Trying to pull docker.io/mythril/myth:latest...
    Getting image source signatures
    Copying blob sha256:a427715287193919b4f58dd6a5f2e54d88fca3623f105a731c4ee0074f9e6f43
    Copying blob sha256:b549f31133a955f68f9fa0d93f18436c4a180e12184b999a8ecf14f7eaa83309
    Copying blob sha256:d9d3d06107c9171d8ac032449557aa566a63dff6b63111875f3e650a2cf28343
    Copying blob sha256:ca63c4fb0e35dea511c90883b19ef8392e7eacaa609469ab23f88ea5874ca584
    Copying blob sha256:13b2a146d0c8b96d7982280f0efd09d971612bdc627a8eb18f04b8f88abc491f
    Copying blob sha256:1d6713d4655182bb807cdcf4ad12072b93a7969c542d9c52868db59d074ab3a3
    Copying blob sha256:ab0a4465f42f21b28c64b0279df70e2a012054aa8eae24b6c7e0dfb12d875191
    Copying blob sha256:250cacd08b0f4e79caf30961b002891b3d090b9a25ee3b3178910680b4a392f8
    Copying blob sha256:6c6259ac740b905a0d2abdd55d478f3907ff1de9717a1f1fd9adaf1f0d7f7914
    Copying config sha256:8f43d0a6470b8494bae11482419cf5831c25773573afb119885a2b5e90e901e1
    Writing manifest to image destination
    Storing signatures
    WARNING: image platform (linux/amd64) does not match the expected platform (linux/arm64)
    8f43d0a6470b8494bae11482419cf5831c25773573afb119885a2b5e90e901e1
    ```
- `podman run`
  - docker image를 실행해주는 명령어이다. 보통 `-it`옵션과 함께 사용되는 듯 합니다.
  - 명령어
    ```bash
    ➜ podman run -it myth
    
    WARNING: image platform (linux/amd64) does not match the expected platform (linux/arm64)
    usage: myth [-h] [-v LOG_LEVEL] {safe-functions,analyze,a,disassemble,d,list-detectors,read-storage,function-to-hash,hash-to-address,version,help} ...

    Security analysis of Ethereum smart contracts

    positional arguments:
      {safe-functions,analyze,a,disassemble,d,list-detectors,read-storage,function-to-hash,hash-to-address,version,help}
                            Commands
        safe-functions      Check functions which are completely safe using symbolic execution
        analyze (a)         Triggers the analysis of the smart contract
        disassemble (d)     Disassembles the smart contract
        list-detectors      Lists available detection modules
        read-storage        Retrieves storage slots from a given address through rpc
        function-to-hash    Returns the hash signature of the function
        hash-to-address     converts the hashes in the blockchain to ethereum address
        version             Outputs the version

    optional arguments:
      -h, --help            show this help message and exit
      -v LOG_LEVEL          log level (0-5)
    ```
- `podman ps`
  - 이 명령어도 docker와 같이 현재 실행중인 image들을 보여주는 명령어입니다.
  - 명령어
    ```bash
    ➜ podman ps
    
    CONTAINER ID  IMAGE       COMMAND     CREATED     STATUS      PORTS       NAMES
    ```
- `podman help`
  ```bash
  ➜ podman help
  
  Manage pods, containers and images

  Usage:
    podman [options] [command]

  Available Commands:
    attach      Attach to a running container
    build       Build an image using instructions from Containerfiles
    commit      Create new image based on the changed container
    container   Manage containers
    cp          Copy files/folders between a container and the local filesystem
    create      Create but do not start a container
    diff        Display the changes to the object's file system
    events      Show podman system events
    exec        Run a process in a running container
    export      Export container's filesystem contents as a tar archive
    generate    Generate structured data based on containers, pods or volumes
    healthcheck Manage health checks on containers
    help        Help about any command
    history     Show history of a specified image
    image       Manage images
    images      List images in local storage
    import      Import a tarball to create a filesystem image
    info        Display podman system information
    init        Initialize one or more containers
    inspect     Display the configuration of object denoted by ID
    kill        Kill one or more running containers with a specific signal
    kube        Play containers, pods or volumes from a structured file
    load        Load image(s) from a tar archive
    login       Login to a container registry
    logout      Logout of a container registry
    logs        Fetch the logs of one or more containers
    machine     Manage a virtual machine
    manifest    Manipulate manifest lists and image indexes
    network     Manage networks
    pause       Pause all the processes in one or more containers
    pod         Manage pods
    port        List port mappings or a specific mapping for the container
    ps          List containers
    pull        Pull an image from a registry
    push        Push an image to a specified destination
    rename      Rename an existing container
    restart     Restart one or more containers
    rm          Remove one or more containers
    rmi         Removes one or more images from local storage
    run         Run a command in a new container
    save        Save image(s) to an archive
    search      Search registry for image
    secret      Manage secrets
    start       Start one or more containers
    stats       Display a live stream of container resource usage statistics
    stop        Stop one or more containers
    system      Manage podman
    tag         Add an additional name to a local image
    top         Display the running processes of a container
    unpause     Unpause the processes in one or more containers
    untag       Remove a name from a local image
    update      update an existing container
    version     Display the Podman version information
    volume      Manage volumes
    wait        Block on one or more containers

  Options:
    -c, --connection string         Connection to use for remote Podman service (default "podman-machine-default")
        --help                      Help for podman
        --identity string           path to SSH identity file, (CONTAINER_SSHKEY) (default "/Users/user/.ssh/podman-machine-default")
        --log-level string          Log messages above specified level (trace, debug, info, warn, warning, error, fatal, panic) (default "warn")
        --noout                     do not output to stdout
        --ssh string                define the ssh mode (default "golang")
        --storage-opt stringArray   Used to pass an option to the storage driver
        --url string                URL to access Podman service (CONTAINER_HOST) (default "ssh://core@localhost:51678/run/user/501/podman/podman.sock")
    -v, --version                   version for podman
    ```

# EoD
요즘들어 새로운것을 많이 접하다보니 기억을 잘 못하는 저를 위해 글을 남기는.. :) <br>
