---
layout: post
title: Automatic builds and (unit) tests of Zephyr program via GitHub Actions and Twister
---
I had developed several Zephyr programs. I like Zephyr very much, see also my other blogposts about Zephyr, but Zephyr and the Nordic SDK are still in development.
This means things are changed, and when you do not regulary build you programs using the latest versions, you don't know whether you programs still work.

This means that I needed at least a mechanism to automatically build my Zephyr programs using the latest SDK and libraries.

Further contineuous integration is of cours also useful. So beside building program, I also wanted to be able to execute unit tests on my program, for instance after each change.

# How to do it?
In principle it is very easy:
* install the necessary SDK and tools
* define tests
* execute tests

Below I elaborate on each step.

## Install necessary SDK and tools
You can use an already developed docker image (just google), or you can install all items yourself. I ended up installing everything myself from the command line; it is not very complex, however executing on GitHub does cost some time.

In GitHub you have to define a so called workflow (for instance via the 'Action'  button on GitHub). See the instructions on GitHub for how to do this.

The GitHub workflow should have the following contents:
```
name: InstallZephyrSDKBuildTwister

on: 
  push:
  pull_request:
  workflow_dispatch:
  schedule:
  - cron: "0 19 * * 1-5"
jobs:
  installZephyrAndBuild:
    runs-on: ubuntu-latest
    steps:
      - name: InstallWestWorkspace
        run: |
          sudo apt update
          sudo apt upgrade
          wget https://apt.kitware.com/kitware-archive.sh
          sudo bash kitware-archive.sh
          sudo apt install --no-install-recommends git cmake ninja-build gperf   ccache dfu-util device-tree-compiler wget   python3-dev python3-pip python3-setuptools python3-tk python3-wheel xz-utils file   make gcc gcc-multilib g++-multilib libsdl2-dev
        
          pip3 install west
          
          cd ${{github.workspace}}
          cd ..
          #west init -m https://github.com/nrfconnect/sdk-nrf --mr v2.1.1 /home/runner/work
          west init -m https://github.com/nrfconnect/sdk-nrf /home/runner/work
          cd /home/runner/work
          west update
          pip3 install -r zephyr/scripts/requirements-base.txt
                    
          sudo apt install --no-install-recommends cmake ninja-build
          #source ./zephyr/zephyr-env.sh
          
          #install toolchains
          cd /usr/local/
          # get minimal version, without toolchains
          sudo wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.15.1/zephyr-sdk-0.15.1_linux-x86_64_minimal.tar.gz
          sudo tar -xvf zephyr-sdk-0.15.1_linux-x86_64_minimal.tar.gz
          
          # install only necessary toolchains
          cd zephyr-sdk-0.15.1
          sudo wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.15.1/toolchain_linux-x86_64_arm-zephyr-eabi.tar.gz
          sudo tar -xvf toolchain_linux-x86_64_arm-zephyr-eabi.tar.gz
          sudo wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.15.1/toolchain_linux-x86_64_x86_64-zephyr-elf.tar.gz
          sudo tar -xvf toolchain_linux-x86_64_x86_64-zephyr-elf.tar.gz

      - name: Checkout
        uses: actions/checkout@v3
        with:
          fetch-depth: 0
          path: repototest

      - name: BuildTargets
        working-directory: repototest
        run: |
          export ZEPHYR_TOOLCHAIN_VARIANT=zephyr
          export ZEPHYR_SDK_INSTALL_DIR=/opt/toolchains/zephyr-sdk-0.15.1/
          west build --pristine -b qemu_x86
          #next line gives error 'no module named 'cbor2''
          pip install cbor2
          west build --pristine -b thingy91_nrf9160_ns

          # used for debugging via SSH
      #- name: Setup upterm session
      #  uses: lhotari/action-upterm@v1

      - name: RunTwister
        working-directory: repototest
        run: |
          #pip install tabulate
          pip install ply
          #next line necessary to build for native_posix
          sudo apt install gcc-multilib
          # in sample.yaml is indicated that integration tests will use native_posix
          /home/runner/work/zephyr/scripts/twister -v --integration  -T ./

```
This installs the Zephyr SDK with the additions for nRF. After that it checks out you GitHub repo, and build it for two different targets (qemu_x86 and thingy91_nrf9160_ns). You can of course adapt these targets and add you own.
These builds are just to check that the repo can (still) be built successfully.

After that Twister is ran. Twister builds the repo again (now for native_posix) and runs the tests that should be defined in `sample.yaml`.

For the Hello World program the `sample.yaml` file has the following contents:
```
sample:
  description: Hello World sample, the simplest Zephyr
    application
  name: hello world
common:
    tags: introduction
# use option --integration on fi github actions
    integration_platforms:
      - native_posix
    harness: console
    harness_config:
      type: one_line
      regex:
        - "Hello World! (.*)"
tests:
  sample.basic.helloworld:
    tags: introduction

```

The `integration_plaforms`  indicates for which platforms the repo must be built if the `--integration` option is supplied to twister (what is done via the workflow file above)

