# profanity3 for windows

Profanity is the fastest vanity generator for ethereum addresses and contract addresses.

"profanity3 is a fork of "profanity2" from 1inch Network <info@1inch.io>, which fixes the security problems of the original "profanity" from Johan Gustafsson <profanity@johgu.se>.

"profanity3" from Rodrigo Madera <madera@acm.org>, is the same as "profanity2" with just one special feature: it can reverse engineer "profanity1" keys.

"profanity3WINx64" is a fork of "profanity3" with some adjustments to compile for windows.

![Screenshot](/img/WS2022DC12.png?raw=true "Windows Server 2022 Datacenter")

# Important to know

A previous version of this project (hereby called "profanity1" for context) has a known critical issue due to a bad source of randomness. The issue enables attackers to recover the private key given a public key: https://blog.1inch.io/a-vulnerability-disclosed-in-profanity-an-ethereum-vanity-address-tool-68ed7455fc8c

The good guys at 1inch created a follow-up project called "profanity2" which was forked from the original "profanity1" project and modified to guarantee **safety by design**. They claim that "this means that the source code of this project does not require any audits, but still guarantee safe usage." Kind of a bold statement (if you ask me) although it's pretty much true.

The "profanity2" project is not generating keys anymore (as opposed to "profanity1"). Instead, it adjusts a user-provided public key until a desired vanity address is discovered. Users provide a seed public key in the form of a 128-symbol hex string with the `-z` parameter flag. The resulting private key should then be added to the seed private key to achieve a final private key with the desired vanity address (remember: private keys are just 256-bit numbers). Running "profanity2" can even be outsourced to someone completely unreliable - it is still safe by design.

## Getting public key for mandatory `-z` parameter

Generate private key and public key via openssl in MSYS2-terminal (remove prefix "04" from public key):
```bash
$ openssl ecparam -genkey -name secp256k1 -text -noout -outform DER | xxd -p -c 1000 | sed 's/41534e31204f49443a20736563703235366b310a30740201010420/Private Key: /' | sed 's/a00706052b8104000aa144034200/\'$'\nPublic Key: /'
```

## Merging private keys (never use online calculators!)

```PRIVATE_KEY_A = private key from "Generate private key"```
```PRIVATE_KEY_B = private key from "profanity"-search result```

### use Terminal

Use private keys as 64-symbol hexadecimal string WITHOUT `0x` prefix:
```bash
(echo 'ibase=16;obase=10' && (echo '(PRIVATE_KEY_A + PRIVATE_KEY_B) % FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F' | tr '[:lower:]' '[:upper:]')) | bc
```

### use Python

Use private keys as 64-symbol hexadecimal string WITH `0x` prefix:
```bash
$ python3
>>> hex((PRIVATE_KEY_A + PRIVATE_KEY_B) % 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F)
```

### handle "Leading Zero"-Bug (Example and Fix)
```bash
>>> (echo 'ibase=16;obase=10' && (echo '(0bc657b0af28b743c7f0d49c4de78efd47a5c8923dabfdef051fff5cdc7c30e7 + 0x0000f8ba428990fca1e618a252ac3614f5de19b20ff00c2ded57bfb6933830aa) % FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F' | tr '[:lower:]' '[:upper:]')) | bc
>>> BC7506AF1B2484069D6ED3EA093C5123D83E2444D9C0A1CF277BF226FB49AD0
>>> 0BC7506AF1B2484069D6ED3EA093C5123D83E2444D9C0A1CF277BF226FB49AD0 (Pad with 1 zero as length only 63)
```

Note that if **less than 64-symbol** is generated, simply **add leading zeroes** to the private key. This occurs due to the summation of `PRIVATE_KEY_A` & `PRIVATE_KEY_B` with leading 0s not being displayed in final hex generated. Example above has 1 overlapping leading 0 thus we add an additional zero.


# Usage
```
usage: ./profanity3 [OPTIONS]

  Mandatory args:
    -z                      Seed public key to start (without prefix 04)
                            (add it's private key to the "profanity3" resulting private key)
  Basic modes:
    --benchmark             Run without any scoring, a benchmark.
    --zeros                 Score on zeros anywhere in hash.
    --letters               Score on letters anywhere in hash.
    --numbers               Score on numbers anywhere in hash.
    --mirror                Score on mirroring from center.
    --leading-doubles       Score on hashes leading with hexadecimal pairs
    --crack                 Try to find the private key of a profanity1 public key

  Modes with arguments:
    --leading <single hex>  Score on hashes leading with given hex character.
    --matching <hex string> Score on hashes matching given hex string.

  Advanced modes:
    --contract              Instead of account address, score the contract
                            address created by the account's zeroth transaction.
    --leading-range         Scores on hashes leading with characters within
                            given range.
    --range                 Scores on hashes having characters within given
                            range anywhere.
  Range:
    -m, --min <0-15>        Set range minimum (inclusive), 0 is '0' 15 is 'f'.
    -M, --max <0-15>        Set range maximum (inclusive), 0 is '0' 15 is 'f'.

  Device control:
    -s, --skip <index>      Skip device given by index.
    -n, --no-cache          Don't load cached pre-compiled version of kernel.

  Tweaking:
    -w, --work <size>       Set OpenCL local work size. [default = 64]
    -W, --work-max <size>   Set OpenCL maximum work size. [default = -i * -I]
    -i, --inverse-size      Set size of modular inverses to calculate in one
                            work item. [default = 255]
    -I, --inverse-multiple  Set how many above work items will run in
                            parallell. [default = 16384]
  Examples:
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --leading f 
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --matching dead
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --matching badXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXbad
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --leading-range -m 0 -M 1
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --leading-range -m 10 -M 12
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --range -m 0 -M 1
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --contract --leading 0
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --crack

  About:
    profanity3 is a vanity address generator for Ethereum that utilizes
    computing power from GPUs using OpenCL.

  Forked "profanity3":
    Author: Rodrigo Madera <madera@acm.org>
    Disclaimer:
      This project "profanity3" was forked from the "profanity2" project and
      modified to allow you to assess the quality of your "profanity1" keys.
      No guarantees whatsoever are given, so use this at your own risk and
      don't bother me about it. Also, don't be evil. Use this to assess
      your own addresses and keep them safe. But better yet, if you have
      any wallets generated with profanity1, just throw them away.

  Forked "profanity2":
    Author: 1inch Network <info@1inch.io>
    Disclaimer:
      The project "profanity2" was forked from the original project and
      modified to guarantee "SAFETY BY DESIGN". This means source code of
      this project doesn't require any audits, but still guarantee safe usage.

  From original "profanity":
    Author: Johan Gustafsson <profanity@johgu.se>
    Beer donations: 0x000dead000ae1c8e8ac27103e4ff65f42a4e9203
    Disclaimer:
      Always verify that a private key generated by this program corresponds to
      the public key printed by importing it to a wallet of your choice. This
      program like any software might contain bugs and it does by design cut
      corners to improve overall performance.
```

## Example
```bash
STEP 1: create a random private and public key-pair
$ openssl ecparam -genkey -name secp256k1 -text -noout -outform DER | xxd -p -c 1000 | sed 's/41534e31204f49443a20736563703235366b310a30740201010420/Private Key: /' | sed 's/a00706052b8104000aa144034200/\'$'\nPublic Key: /'
> Private Key: 8825e602379969a2e97297601eccf47285f8dd4fedfae2d1684452415623dac3
> Public Key: 04e9507a57c01e9e18a929366813909bbc14b2d702a46c056df77465774d449e48b9f9c2279bb9a5996d2bd2c9f5c9470727f7f69c11f7eeee50efeaf97107a09c

remove prefix 04 from public-key 

STEP 2: search for privates keys
$ ./profanity3.exe -z e9507a57c01e9e18a929366813909bbc14b2d702a46c056df77465774d449e48b9f9c2279bb9a5996d2bd2c9f5c9470727f7f69c11f7eeee50efeaf97107a09c --matching 888888XXXXXXXXXXXXXXXXXXXXXXXXXXXX888888
> Time: 255s Score: 5 Private: 0x00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf Address: 0x8888c2664dcabec06ba8b89660b6f40fbf888888

STEP 3: merge private keys (without prefix 0x)
PRIVATE_KEY_A=8825e602379969a2e97297601eccf47285f8dd4fedfae2d1684452415623dac3
PRIVATE_KEY_B=00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf

$ (echo 'ibase=16;obase=10' && (echo '(8825e602379969a2e97297601eccf47285f8dd4fedfae2d1684452415623dac3 + 00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf) % FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F' | tr '[:lower:]' '[:upper:]')) | bc
> 882634F7873FFC8114FCA3CF01D8586BF2F194D5B81C86451C453C6C61538772

add prefix 0x

PRIVATE_KEY=0x882634F7873FFC8114FCA3CF01D8586BF2F194D5B81C86451C453C6C61538772
```

![Screenshot](/img/WIN11PRO.png?raw=true "Windows 11")

## Install openssl for Windows
- open PowerShell
- >choco install OpenSSL.Light
- restart PowerShell

## Install xxd in MSYS2-terminal
- pacman -S vim

## Install bc in MSYS2-terminal
- pacman -S bc

## Compile for Windows

- Install MSYS2
- Open MSYS2 (MINIGW64) shell (do not try other versions)
- ```pacman -S mingw-w64-x86_64-toolchain mingw-w64-x86_64-opencl-headers```
- ```pacman -S base-devel gcc vim cmake```
- ```pacman -S mingw-w64-x86_64-bc```
- cd /C/VANITY/profanity3WINx64
- make -f Makefile.WIN
- ./profanity3.exe

## Compile for Linux (in Github Codespaces)

- sudo apt-get update && sudo apt-get upgrade
- sudo apt-get install opencl-headers ocl-icd-opencl-dev intel-opencl-icd
- make -f Makefile.LINUX clean
- make -f Makefile.LINUX
- ./profanity3.x64

## Compile for Windows-Subsystem for Linux

- start windows powershell as administrator
- ```wsl --install Ubuntu-20.04```
- cd /mnt/c/VANITY/profanity3WINx64
- ```sudo apt-get update && sudo apt-get upgrade```
- ```sudo apt-get install make g++ dkms```
- ```sudo apt-get install opencl-headers ocl-icd-opencl-dev intel-opencl-icd```
- ```make -f Makefile.LINUX clean```
- ```make -f Makefile.LINUX```
- ./profanity3.x64
- install cuda for subsystem
- wget https://developer.download.nvidia.com/compute/cuda/11.8.0/local_installers/cuda_11.8.0_520.61.05_linux.run
- sudo sh cuda_11.8.0_520.61.05_linux.run

## Benchmarks - Current version
|Model|Clock Speed|Memory Speed|Modified straps|Speed|Time to match eight characters
|:-:|:-:|:-:|:-:|:-:|:-:|
|RTX 3070 OC|1850+191|6800+999|-I 64384 -w 64384 -i 512|501.0 MH/s|
|RTX 3070 OC|2010|7550|NO|470.0 MH/s|
|RTX 3070|1850|6800|NO|441.0 MH/s| ~10s
|GTX 1070 OC|1950|4450|NO|179.0 MH/s| ~24s
|GTX 1070|1750|4000|NO|163.0 MH/s| ~26s
|GTX 1060 3GB OC|2050|4212|NO|101.0 MH/s| 
|RX 480|1328|2000|YES|120.0 MH/s| ~36s
|Apple Silicon M1<br/>(8-core GPU)|-|-|-|45.0 MH/s| ~97s
|Apple Silicon M1 Max<br/>(32-core GPU)|-|-|-|172.0 MH/s| ~25s

## Tweaks
```bash
.\profanity3 -I 64384 -w 64384 -i 512 -z e9507a57c01e9e18a929366813909bbc14b2d702a46c056df77465774d449e48b9f9c2279bb9a5996d2bd2c9f5c9470727f7f69c11f7eeee50efeaf97107a09c --leading-doubles 

GPU0: NVIDIA GeForce RTX 3070, 8589279232 bytes available, 46 compute units (precompiled = no)
tweak raise RTX3700 from 441 MH/s to 462 MH/s

GPU0: NVIDIA GeForce RTX 3070 OC WC, Core-Clock: +170; Memory-Clock: +845
tweak raise RTX3700 OC from 470 MH/s to 497 MH/s

GPU0: NVIDIA GeForce RTX 3070 OC WC, Core-Clock: 1850+191; Memory-Clock: 6800+999
tweak raise RTX3700 OC from 470 MH/s to 497 MH/s
```

![Screenshot](/img/WS2022DC12OC501.png?raw=true "RTX3070OC")

### 50% probability to find a collision
```
  i.e.: rate = 440 MHashes / sec = 440'000'000 Hashes / sec
  permutations = 16 ^ (prefixlength + postfixlength)
  prob50% = log(0.5) / log(1 - 1 / permutations)
  timeTo50% = prob50% / rate
```

https://keisan.casio.com/calculator

```log(0.5)/log(1-1/16^12)/440000000/60/60/24 (days)```

```
GPU RTX 3070 -> Rate = 440 MH/s -> 50% probability:
 7 chars -> 0.5 sec
 8 chars -> 7 sec
 9 chars -> 108 sec
10 chars -> 28 min
11 chars -> 7.7 h
12 chars -> 5 days
13 chars -> 82 days
14 chars -> 3.6 years
15 chars -> 56 years
16 chars -> 921 years
```
