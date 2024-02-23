# profanity3 for windows 中文說明 & 踩地雷排解
###### 以下翻譯內容來自作者xdeltax的profanity3WINx64專案的README.md，有加入本人的一些不專業見解、安裝說明以及踩雷debug之處，非全文照翻


## 目錄
  - [簡介](#簡介)
  - [事前了解的知識](#事前了解的知識)
  - [使用需知](#使用需知)
          - [在了解這是什麼之前，請不要使用它。](#在了解這是什麼之前請不要使用它)
  - [使用教學 (Windows Only)](#使用教學-windows-only)
    - [0.建立環境 (Windows \>= 7)](#0建立環境-windows--7)
      - [(1) 安裝choco (參考來源)](#1-安裝choco-參考來源)
      - [(2) 安裝MSYS2 (MSYS2官網)](#2-安裝msys2-msys2官網)
      - [(3) 安裝OpenSSL](#3-安裝openssl)
      - [(4) 安裝xxd](#4-安裝xxd)
      - [(5) 安裝bc](#5-安裝bc)
    - [1.在windows上編譯Profanity3](#1在windows上編譯profanity3)
    - [2.生成一串公鑰A 以及 私鑰A (絕對只能在本地端執行)](#2生成一串公鑰a-以及-私鑰a-絕對只能在本地端執行)
    - [3.利用公鑰A 碰撞計算私鑰B (可以外包)](#3利用公鑰a-碰撞計算私鑰b-可以外包)
    - [4.合併私鑰(絕對只能在本地端執行)](#4合併私鑰絕對只能在本地端執行)
      - [方法一：從MSYS2 終端機](#方法一從msys2-終端機)
      - [方法二：從Python shell(非python程式)](#方法二從python-shell非python程式)
  - [profanity3 help file](#profanity3-help-file)
  - [效能](#效能)
    - [修正參數](#修正參數)
    - [碰撞時間](#碰撞時間)
  - [debug Q\&A](#debug-qa)

## 簡介

Profanity 可以用來生成EVM虛榮地址/虛名地址
(我比較想把它取名叫做自訂地址 台灣人不管怎麼翻都很奇怪 有查到香港可以叫做靚號地址)

為什麼需要生成這些地址? 請google「漢明權重」。簡單的來說，在以太坊內，公鑰地址內越多成雙成對的「0」，就越可以降低所需的gas fee，對於一些耗gas的智能合約來說能省gas就省gas([參考來源](https://www.odaily.news/post/5183914))。
同時它也可以生成像是0x5269...、0x8888...、0x6666...、0xdead...等開頭，或是0xdead...dead等的部分字元自訂的公鑰，在只有0~F的字海中發現這個酷酷ㄉ地址，還可以順便跟幣友炫耀(x。

- Profanity原本是2017年由Johan Gustafsson 推出的專案，後來在2022年被1inch 揭露其隨機生成碼可以被逆向工程導致私鑰外洩，而該專案也被封存
- Profanity2是後來1inch修改Profanity 的種子碼生成方式，現在可正常使用
- Profanity3是後來Rodrigo Madera 的加強版，多了可以逆向生成原本Profanity 生成公鑰回推私鑰的功能
  
以上都是Linux user friendly，Windows user not friendly 的版本

- profanity3WINx64 是來自xdeltax 的加強版，增加了對於windows 環境的支援



## 事前了解的知識

- 知道什麼是以太坊地址
- 知道什麼是公鑰什麼是私鑰
- 知道簡單的機率問題
- 知道公鑰和私鑰是如何產生的
- 略懂英文 (至少幣圈術語要看的懂吧)


## 使用需知

###### 在了解這是什麼之前，請不要使用它。

> DYOR (Do Your Own Research.)

該專案的先前版本 (也就是Profanity) 存在隨機源 (種子) 不佳的問題，可參考下方連結，該問題使攻擊者可以在給定公鑰的情況下恢復私鑰，也導致了2022年造市商Wintermute被盜了約50億台幣，相關訊息：

[1inch漏洞消息](https://blog.1inch.io/a-vulnerability-disclosed-in-profanity-an-ethereum-vanity-address-tool/) 
[鉅亨消息/金色財經快訊](https://news.cnyes.com/news/id/4955767) 

此專案大部分的代碼都基於1inch團隊後來修改的Profanity2程式。他們說：

> 該專案的源代碼不需要任何審核，但仍然保證安全使用。

"profanity2" 專案不生成金鑰 (恰好與 "profanity" 相反)。它調整用戶提供公鑰的偏移值，直到發現所需的虛榮地址。使用`-z`參數，後面以128 字符十六進制字符串的形式提供種子公鑰，然後將結果的私鑰添加到種子私鑰中，以獲得具有所需虛榮地址的最終私鑰 (請記住：私鑰只是256位數)，這甚至可以將中間碰撞的過程外包給完全不可靠的人，而它仍然是安全的 (理論上)。

## 使用教學 (Windows Only)

### 0.建立環境 (Windows >= 7)

####  (1) 安裝choco ([參考來源](https://www.nvda.org.tw/refined/ui=2004100000tm=1989344034))
- Windows10：win + X 游標上下選擇到 windows powershell (工作管理員) 進入

- Windows11：win + X 游標上下選擇到 終端機 (系統管理員) 進入

- 輸入```Get-ExecutionPolicy```

- 如果顯示 ```Ristricted```，則再執行```Set-ExecutionPolicy AllSigned```，並輸入```y```繼續；如果顯示 ```RemoteSigned```，則不須執行。


- 安裝 chocolatey

- 輸入```Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))```

- 重新開啟Powershell視窗

- 輸入```choco```，確認是否有chocolatey版本訊息正常顯示

#### (2) 安裝MSYS2 ([MSYS2官網](https://www.msys2.org/))
- 上方超連結點進去，跟著Installation，我這邊下載的是```msys2-x86_64-20240113.exe```
- 打開安裝檔，選擇安裝路徑，完成安裝後會看到start menu多了好幾個MSYS2的shell，在這個專案內我們只會用到MSYS2 MINIGW64。


#### (3) 安裝OpenSSL
- 打開PowerShell
  
- 輸入```choco install OpenSSL.Light```

- 遇到```Do you want to run the script?([Y]es/[A]ll - yes to all/[N]o/[P]rint): ```，輸入```A```同意所有程式碼繼續執行

- 重開機

#### (4) 安裝xxd
- 打開MSYS2
- 輸入```pacman -S vim```

#### (5) 安裝bc
- 打開MSYS2
- 輸入```pacman -S bc```


### 1.在windows上編譯Profanity3
###### 如果需要Linux的編譯方式，可以移駕到Profanity3原文，這裡因為篇幅關係只放Windows的

###### MSYS2的```Ctrl+V```並非貼上，請使用```Shift+Insert```，或是滑鼠右鍵```Paste```

- 打開MSYS2 MINIGW64 (一定只能開這個版本，其他版本會編譯失敗)
  
<img src="https://github.com/brianoy/profanity3/assets/24865458/96be05a9-2425-4a1b-9a40-ce1b1a3d7c98" width="50%" >


- 輸入```pacman -S mingw-w64-x86_64-toolchain mingw-w64-x86_64-opencl-headers```
- 輸入```pacman -S base-devel gcc vim cmake```
- 輸入```pacman -S mingw-w64-x86_64-bc```
  
  
  
- 請將位址改成你放profanity3WINx64的實際資料夾位置，輸入```cd /C/somewhere/in/your/computer/profanity3WINx64```，須注意路徑引用問題，若使用```\```反斜線的路徑，請確保路徑被雙引號包住```""```
- Windows系統請輸入```make -f Makefile.WIN```
- 編譯完成
- 輸入```./profanity3.exe```

### 2.生成一串公鑰A 以及 私鑰A (絕對只能在本地端執行)

透過 openssl 在 MSYS2 終端生成私鑰和公鑰（從公鑰中刪除前綴 "04"）：
```bash
$ openssl ecparam -genkey -name secp256k1 -text -noout -outform DER | xxd -p -c 1000 | sed 's/41534e31204f49443a20736563703235366b310a30740201010420/Private Key: /' | sed 's/a00706052b8104000aa144034200/\'$'\nPublic Key: /'
```

得到：

私鑰A = ```Private Key: 8825e602379969a2e97297601eccf47285f8dd4fedfae2d1684452415623dac3```

公鑰A = ```Public Key: 04e9507a57c01e9e18a929366813909bbc14b2d702a46c056df77465774d449e48b9f9c2279bb9a5996d2bd2c9f5c9470727f7f69c11f7eeee50efeaf97107a09c```

我們需要將公鑰A的```04```前綴刪除，實際得到：```e9507a57c01e9e18a929366813909bbc14b2d702a46c056df77465774d449e48b9f9c2279bb9a5996d2bd2c9f5c9470727f7f69c11f7eeee50efeaf97107a09c```這串128碼16進位數的公鑰A。

### 3.利用公鑰A 碰撞計算私鑰B (可以外包)
###### 此範例是指定待擬合的公鑰，並尋找(碰撞)其私鑰

- 格式：
```bash
./profanity3.exe -z 公鑰A --matching 888888XXXXXXXXXXXXXXXXXXXXXXXXXXXX888888
```

- 範例：
```bash
./profanity3.exe -z e9507a57c01e9e18a929366813909bbc14b2d702a46c056df77465774d449e48b9f9c2279bb9a5996d2bd2c9f5c9470727f7f69c11f7eeee50efeaf97107a09c --matching 888888XXXXXXXXXXXXXXXXXXXXXXXXXXXX888888
```

- 得到：
```bash
> Time: 255s Score: 5 Private: 0x00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf Address: 0x8888c2664dcabec06ba8b89660b6f40fbf888888
```

私鑰B = ```Private: 0x00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf```

公鑰B = ```Address: 0x8888c2664dcabec06ba8b89660b6f40fbf888888```


### 4.合併私鑰(絕對只能在本地端執行)
###### 可以使用MSYS2 終端機或Python shell來執行

- 格式為：
```bash
私鑰A = 初始私鑰
私鑰B = 碰撞後產生的私鑰
```

私鑰A = ```8825e602379969a2e97297601eccf47285f8dd4fedfae2d1684452415623dac3```

私鑰B = ```00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf```


### 方法一：從MSYS2 終端機

- 請確保計算時兩個私鑰都是```XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX```，不須加上雙引號```""```、不須加上單引號```''```、不須加上前綴```0x```，為64碼16進位數。

- 終端機輸入格式為：
```bash
> (echo 'ibase=16;obase=10' && (echo '(私鑰A + 私鑰B) % FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F' | tr '[:lower:]' '[:upper:]')) | bc
```

- 在此範例，也就是：

(1)開啟MSYS2 終端機
  
(2)輸入：
```bash
$ (echo 'ibase=16;obase=10' && (echo '(8825e602379969a2e97297601eccf47285f8dd4fedfae2d1684452415623dac3 + 00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf) % FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F' | tr '[:lower:]' '[:upper:]')) | bc
```

(3)得到：
```bash
882634F7873FFC8114FCA3CF01D8586BF2F194D5B81C86451C453C6C61538772
```

- 自行加上```0x```前綴，可得實際私鑰C：```0x882634F7873FFC8114FCA3CF01D8586BF2F194D5B81C86451C453C6C61538772```

### 方法二：從Python shell(非python程式)

請確保計算時兩個私鑰都是```0xXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX```，不須加上雙引號```""```、不須加上單引號```''```，需加上前綴```0x```，```0x```後為64碼16進位數。

- 終端機輸入格式為：
```bash
> hex((私鑰A + 私鑰B) % 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F)
```

在此範例，也就是：

(1)開啟powershell終端機或cmd終端機

(2)輸入```python```，進入```python shell```

(3)輸入：
```bash
>>> hex((0x8825e602379969a2e97297601eccf47285f8dd4fedfae2d1684452415623dac3 + 0x00004ef54fa692de2b8a0c6ee30b63f96cf8b785ca21a373b400ea2b0b2facaf) % 0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F)
```

(4)得到：
```bash
'0x882634f7873ffc8114fca3cf01d8586bf2f194d5b81c86451c453c6c61538772'
```
- 實際私鑰C：```0x882634F7873FFC8114FCA3CF01D8586BF2F194D5B81C86451C453C6C61538772```

## profanity3 help file
```

  強制參數：
    -z                      以種子公鑰開始（不包含前綴04）
                            （將其私鑰添加到“profanity3”生成的私鑰中）
  基本模式：
    --benchmark             不計分數運行，進行基準測試。
    --zeros                 在哈希的任何位置打分。
    --letters               在哈希的任何位置打分。
    --numbers               在哈希的任何位置打分。
    --mirror                從中心進行鏡像打分。
    --leading-doubles       在以十六進制對開頭的哈希上打分。
    --crack                 嘗試找到profanity1公鑰的私鑰。

  帶參數的模式：
    --leading <single hex>  在以給定十六進制字符開頭的哈希上打分。
    --matching <hex string> 在與給定十六進制字符串匹配的哈希上打分。

  高級模式：
    --contract              不是帳戶地址，而是對帳戶的第零筆交易創建的合約
                            地址進行打分。
    --leading-range         在給定範圍內以字符開頭的哈希上打分。
    --range                 在給定範圍內具有字符的哈希上打分。
  範圍：
    -m, --min <0-15>        設置範圍最小值（包括），0是“0”，15是“f”。
    -M, --max <0-15>        設置範圍最大值（包括），0是“0”，15是“f”。

  設備控制：
    -s, --skip <index>      跳過由索引指定的設備。
    -n, --no-cache          不加載內核的緩存的預編譯版本。

  調整：
    -w, --work <size>       設置OpenCL本地工作大小。[默認值= 64]
    -W, --work-max <size>   設置OpenCL最大工作大小。[默認值= -i * -I]
    -i, --inverse-size      設置要在一個工作項中計算的模反數的大小。[默認值= 255]
    -I, --inverse-multiple  設置將在其中運行多少個上述工作項
                            的並行運行。[默認值= 16384]
  示例：
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --leading f 
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --matching dead
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --matching badXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXbad
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --leading-range -m 0 -M 1
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --leading-range -m 10 -M 12
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --range -m 0 -M 1
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --contract --leading 0
    ./profanity3 -z HEX_PUBLIC_KEY_128_CHARS_LONG --crack

  關於：
    profanity3 是一個使用OpenCL的GPU的計算能力的以太坊虛擬機(EVM)地址生成器。

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





## 效能

**各顯卡算力**
|型號|核心頻率|記憶體頻率|帶入的修正參數|算力(百萬哈希/秒)|擬合8位數所需時間
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

### 修正參數
使用RTX 3070, 8G RAM為例，輸入修正參數，可以有效的加強性能：

```bash
.\profanity3 -I 64384 -w 64384 -i 512 -z e9507a57c01e9e18a929366813909bbc14b2d702a46c056df77465774d449e48b9f9c2279bb9a5996d2bd2c9f5c9470727f7f69c11f7eeee50efeaf97107a09c --leading-doubles 
```
**RTX3070超頻**
|型號|Δ核心頻率|Δ記憶體頻率|算力(百萬哈希/秒)
|:-:|:-:|:-:|:-:|
|RTX 3070|+0|+0|462 MH/s|
|RTX 3070|+170|+845|497 MH/s|
|RTX 3070|+191|+999|501 MH/s|


### 碰撞時間

**碰撞到50%哈希值的時間計算公式：**
$$\frac{\log{(0.5)}}{\log{(1-\frac{1}{{16}^n})}}\div H (H/sec)=t (sec)$$
> n為擬合公鑰字元數，為介在1~40的正整數
> 
> H為GPU算力，單位為哈希值/秒
> 
> t為擬合時間，單位為秒

多數計算機會因為精度問題導致計算錯誤，請使用[線上高精度計算機](https://keisan.casio.jp/calculator)計算，進入網頁代入```log(0.5)/log(1-1/16^n)/440000000```，n為擬合字元數，單位為(秒)，可以調整網頁的「桁数」獲取更高精度的log值。

**以3070為例，算力為440MH/s，並且50%的碰撞機率，擬合12碼：**
$$\frac{\log{(0.5)}}{\log{(1-\frac{1}{{16}^{12}})}}\div\frac{440000000\ (H/sec)}{60\times60\times24\ (sec)}=5.13\ (days)$$

**440MH/s算力花費時間**
|字元擬合數量|碰撞50%花費時間
|:-:|:-:|
|7碼|0.5秒|
|8碼|7秒|
|9碼|108秒|
|10碼|28分鐘|
|11碼|7.7小時|
|12碼|5天|
|13碼|82天|
|14碼|3.6年|
|15碼|56年|
|16碼|921年|
|17碼|14743年|
|⋮|⋮|
|40碼|$7.3\times{10}^{31}$年|


## debug Q&A
### 1.python shell叫不出來

> 請確認python是否有在環境變數裡面


### 2.處理前導0問題

```bash
>>> (echo 'ibase=16;obase=10' && (echo '(0bc657b0af28b743c7f0d49c4de78efd47a5c8923dabfdef051fff5cdc7c30e7 + 0x0000f8ba428990fca1e618a252ac3614f5de19b20ff00c2ded57bfb6933830aa) % FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEFFFFFC2F' | tr '[:lower:]' '[:upper:]')) | bc
>>> BC7506AF1B2484069D6ED3EA093C5123D83E2444D9C0A1CF277BF226FB49AD0
```
此範例中會看到生成的私鑰```BC7506AF1B2484069D6ED3EA093C5123D83E2444D9C0A1CF277BF226FB49AD0```為63碼，只須向前面補0直到64碼即可```0BC7506AF1B2484069D6ED3EA093C5123D83E2444D9C0A1CF277BF226FB49AD0```，所以實際私鑰為：```0x0BC7506AF1B2484069D6ED3EA093C5123D83E2444D9C0A1CF277BF226FB49AD0```

因為私鑰A和私鑰B的求和過程中未在生成的十六進制中顯示前導0。


### 3.編譯失敗
遇到：
```bash
profanity.cpp:16:10: fatal error: CL/cl.h: No such file or directory
16 | #include <CL/cl.h>
| ^~~~~~~~~
compilation terminated.
```
> 請確認開啟的視窗是否為MSYS2 MINGW64，而非MSYS2 MSYS

### 4.編譯失敗
遇到：
```bash
Mode: matching
Target: Address
Devices:
  GPU0: NVIDIA GeForce GTX 1650, 4294639616 bytes available, 14 compute units (precompiled = yes)
  GPU1: gfx90c, 9765650432 bytes available, 7 compute units (precompiled = no)

Initializing OpenCL...
  Creating context...Segmentation fault
```
> 在原始的profanity和profanity2都有被提及此問題，疑似是多顆GPU的問題，目前無解
