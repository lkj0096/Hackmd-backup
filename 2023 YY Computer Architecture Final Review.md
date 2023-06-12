---
title: 2023 YY Computer Architecture Final Review
---



<p style="font-size:32px; font-weight:600">祭祖期末複習</p>

[TOC]

> 這一條方程式有三個未知數，你想要解？你是怎麼考進台科的？[name=YY Liu]

# 請先進祭祖土地公廟捻香後再進來複習。
## Single Cycle Processor
### Instructions
```
sw $rt, 0($rs)
add $rd $rs $rt
```

![](https://hackmd.io/_uploads/Bkdux_Vvn.png)

### Datapath
- ![](https://hackmd.io/_uploads/SkeqYHED2.png)
- MUX point:
     - RegDst: 控制write register讀rt / rd (Mem & Addi / Rtype)
     - ALUSrc: 控制ALU第二個input讀rt / imm16 (Rtype / Mem & Addi)
     - PCSrc: 控制pc+4 / pc+4+(signExt(imm16)x4) (not branch / branch)
     - MemtoReg: 控制register write讀ALU result / mem access result
- ALU Control:
     - ![](https://hackmd.io/_uploads/H1HnFHNv2.png)

## Pipeline
### Pipeline Hazards

![](https://hackmd.io/_uploads/SyYRpG4w2.png)


* Structural hazards
    attempt to use the same resource in two different ways at the same time.
    - 記憶體在同一個cycle同時被寫入和讀出

* Data hazards
    attempt to use item before ready.
    1. RAW
        在寫入記憶體前就讀資料
    2. WAW
        新指令的資料被舊指令的資料覆蓋
    3. WAR
        新指令的資料覆蓋掉舊指令希望讀取到的資料

* Control hazards
    attempt to make decision before condition is evaluated.
    - 發生於Branch，在更改program counter前，會把branch後的指令抓出來執行

<hr>

### Design with pipeline support(?)

![](https://hackmd.io/_uploads/SJweVwNPh.png)

- 快遞公司： ( forwarding unit )
  $\text{ID}$ 's `$rs` \|| `$rt` == $\text{MEM}$ 's `$rd $rt`
  或
  $\text{ID}$ 's `$rs` \|| `$rt` == $\text{WB}$ 's `$rd $rt`
  就控制ALU輸入的MUX，把$EX$ 或$\text{WB}$ 的 `$rd $rt` 抓回ALU
  `$rd` for R-format, `$rt` for I-format

  INPUT: 
      1. $\text{MEM}$ `$rd $rt`
      2. $\text{WB}$ `$rd $rt`
      3. if $\text{MEM}$, $\text{WB}$ is using `$rd $rt`
      4. $\text{EX}$ `$rt` `$rs`
  
  OUTPUT:
      1. ALU's MUX selecting signal

- 塞一顆泡泡： ( Hazard Dection Unit )
  lw因為要等到記憶體讀完才能使用
  所以要在EX塞一個泡泡進去然後把$\text{IF}$ & $\text{ID}$ 鎖住

  INPUT:
      1. if opcode == `lw`
  
  OUTPUT:
      1. $\text{IF}$ lock singal
      2. `$PC` lock singal
      3. MUX selecting signal to push 0 in $\text{EX/MEM/WB}$ latch

- 想跳走，我就把後面擋住：( Hazard Dection Unit )
    branch在普通flow下，從開始執行到真正修改`$PC`要等到 $\text{EX}$ 結束，所以又會多fetch跟decode兩個指令
    1. 將branch的判斷改在 $\text{ID}$ 狀態後馬上直接進行。
    2. 在$\text{ID}$ 狀態直接把branch addr馬上算好到給`$PC`
    3. 為了避免這個問題，改善了Hazard Dection Unit東西。
    
    INPUT:
        1. if opcode == `branch`
        2. if need to branch
    
    OUTPUT:
        1. IF flush
        2. MUX selecting signal to choose branch address

<hr>

## Memory Hierarchy
### Intro

*  Random Access Memory
  
|           | SRAM    | DRAM       |
| --------- | ------- | ---------- |
| power     | High    | Low        |
| cost      | High    | Low        |
| density   | Low     | High       |
| refresh   | No need | Need       |
| component | D-FF    | Transistor |


* Locality
    * Temporal Locality	
      用到這筆資料，以後應該也會常常用到
    * Spatial Locality
      用到這筆資料，把附近的資料也抓出來

* Hierarchy
    
    ![](https://hackmd.io/_uploads/ryGR5wbD3.png)

* 4個問題：
    1.	block placement			放哪裡
    2.	block identification	怎麼找
    3.	block replacement		替死鬼
    4.	write strategy		    寫入資料

<hr>

### Cache

- cache中最小的資料單位為一個block
- direct-mapped cache
    - 低位元的address 直接對應cache index (index)
    - 高位元的address 放到cache data中 (tag)

    ![](https://hackmd.io/_uploads/S1906wbPh.png)
    ![](https://hackmd.io/_uploads/ryd7xObwh.png)

- How to write data?

    - Write Through
        cache和memory都更新
        但memory很慢，所以CPI會增加
        - what if miss?
        1. write allocate: 把cache抓進來，mem也一起更新
        2. write around: 不抓進cache，只更新memory
    
    - Write Buffer
        ![](https://hackmd.io/_uploads/SJZzzuZv2.png)
        寫入的資料先放到write buffer中
        write buffer滿了? -> CPU stalled
    
    - Write Back
        在cache中做記號(dirty bit)
        cache garbage collection時 dirty bit == 1 就寫回memory
        - what if miss?
        1. 就抓進來，只更新cache
            

- Interleaving

    ![](https://hackmd.io/_uploads/HJuPw_-v2.png)

    |           | 左     | 中     | 右     |
    | --------- | ------ | ------ | ------ |
    | Bus width | 一單位  | 四單位 | 一單位 |
    | Cost      | 小     | 大     | 中     |
    | time      | 慢     | 快     | 中     |
    
    - 把多個request使用相同一組bus拿到不同的memory bank處理 (credit: 木白)
    
    ![](https://hackmd.io/_uploads/ryBl8gfPh.png)
    
- Measuring Cache Performance
    - $\begin{split}
       Memory\ Stall\ Cycles 
      \end{split}$
      
      $\begin{split}
       = \frac {Memory\ accesses}{Program} \cdot 
       {Miss\ rate} \cdot {Miss\ penalty}
      \end{split}$
      
      $\begin{split}
       = \frac {Instructions}{Program} \cdot \frac{Misses}{Instruction} \cdot {Miss\ penalty}
      \end{split}$
      
      $\frac {Memory\ accesses} {Program}$ : 該程式有使用過多少次memory
      
      ${Miss\ rate}$ : 存取cache時miss的機率
      
      ${Miss\ penalty}$ : miss需要等待的時間
      
      $\frac {Instructions}{Program}$ : 程式有多少instructions
      
      $\frac{Misses}{Instruction}$ : instructions中有多少會發生miss

    - $AMAT$ : Average Memory Access Time
     
      $={Hit\ time} + {Miss\ rate} \cdot {Miss\ penalty}$
    

- Associative
    
    1. Direct mapping (miss rate最高)
       一個蘿蔔一個坑 (1個block一個index)
    
    2. Set (N-way) Associative
       N個蘿蔔一個坑 (N個block一個index)
       
    3. Full Associative (miss rate最低)
       全部都可以放 

- multi-level
    
    - L1找不到就去L2找，還找不到就只好去memory
    - L1 -> 最快，但很小，Temporal Locality
    - L2 -> 比較慢一點，但大很多，spatial Locality

- cache miss 原因
 
    1. Compulsory 
       電腦剛開，就沒辦法
    2. Conflict 
       跟別人的address撞到
    3. Capacity 
       容量太小
    4. Invalidation 
       其他core或thread改到記憶體，讓cache變invalid

<hr>

### Virtual Memory

* name translation table:
    | Physical Memory | Virtual Memory |
    | --------------- | -------------- |
    | block           | page           |
    | miss            | page fault     |


<hr>

## 考前猜題
1. explain 3 types of hazard.
    -  structural hazard
        -  有兩個(以上)個指令會用到同一個conponents
    -  control hazard
        -  還未判斷好branch condition，就執行後續的指令
    -  data hazard
        -  RAW, WAR, WAW
        -  e.g.讀到還未沒寫完的資料(舊的資料)
2. directed map 操作 or N - way associative or fully associative
    - e.g. 高位元當作tag, 低位元當作index 
    - 以index決定要放在cache中的哪一格，再看tag對不對
    - tag對了 $\rightarrow$ hit
    - tag不對 $\rightarrow$ miss
3. pipeline datapath (forwarding unit & hazard detection unit)
	![](https://hackmd.io/_uploads/B1828wED3.png)
    - 畫forwarding unit, hazard detection的圖給你，讓你把datapath畫上去
    - forwarding unit : 
        - input : 
            - EX/MEM rd
            - EX/MEM WB
            - MEM/WB rd
            - MEM/WB WB
            - ID/EX rs
            - ID/EX rt
        - output : 
            - ALU的MUX(兩條)
    - hazard detection unit : 
        - input : 
            - IF/ID rs
            - IF/IF rt
            - ID/EX rt
            - ID/EX lw
        - output : 
            - PC
            - IF/ID 
            - ID/EX MUX
            
4. 給一個data path & 一些指令(可能會造成hazard)，問要幾個cycles才能結束
    - 若n個指令(沒有hazard) $\rightarrow$則為(n+5-1)cycles *(秉諺說這不會考)*
    - 若n個指令(有hazard)$\rightarrow$則你自己想辦法QQ
5. 計算題組(CPI類)
7. SRAM與DRAM的比較 *(contributed by 鈺晨)*
	- SRAM: 密度低，高耗電，很貴，很快
	- DRAM: 密度高，低耗電，很便宜，很慢



<br><br><br><br><hr>

# *** 祭祖期末土地公廟 ***

:::info
CS2006301 計算機組織 Computer Organization 期末土地公廟
***歡迎諸位信眾參拜，自由樂捐***
:::

![](https://hackmd.io/_uploads/ryyc3PEP3.png =350x)![](https://media.discordapp.net/attachments/1097881043245223977/1117762684142751774/E699AEE6B58EE7A685E5AFBA_E9A699E6B2B9E7AEB12.png?width=899&height=675 =350x)

> \\ | / 記得燒香拜拜 \\ | /
> 下面開放上香

>供品桌
>![](https://hackmd.io/_uploads/B1tRh_NP3.png =50x)![](https://hackmd.io/_uploads/B1tRh_NP3.png =50x)![](https://hackmd.io/_uploads/B1tRh_NP3.png =50x)
>![](https://hackmd.io/_uploads/B1tRh_NP3.png =50x)![](https://hackmd.io/_uploads/B1tRh_NP3.png =50x)![](https://hackmd.io/_uploads/B1tRh_NP3.png =50x)
>
>![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)
>![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)![](https://hackmd.io/_uploads/BJsvJYEw2.jpg =50x)
>[name=信眾 -林小弟 贈\|/]
>![](https://hackmd.io/_uploads/SyFsGKVPn.png =50x)
---某信眾上香完忘記帶走的假髮



>0ω0
>030
>QuQ
>OvO
>o.O
>=.=

:::warning
宮廟聖地，請保持肅靜
保持社交距離一行縮排
上香麻煩依照排隊順序
:::

>![](https://cdn.discordapp.com/emojis/1014054379810213929.gif?size=128&quality=lossless =80x)
>  [name=Yu-chen Kuo]

>![](https://cdn.discordapp.com/emojis/1014054379810213929.gif?size=128&quality=lossless =80x)
>[name=MIGO]

>![](https://cdn.discordapp.com/emojis/1014054379810213929.gif?size=128&quality=lossless =80x)
>[name=Ping Yen Wu]

>![](https://cdn.discordapp.com/emojis/1014054379810213929.gif?size=128&quality=lossless =80x)
>[name=cyberloser]

```
 \\     ||     //
  \\    ||    //
   \\   ||   //
    \\  ||  //
     \\ || //
      \\||//  
``` 
>[name=lkj]

>![](https://cdn.discordapp.com/attachments/936967715141320725/1117760201379020962/image.png =200x)
>[name=XDD]

>![](https://cdn.discordapp.com/emojis/1014054379810213929.gif?size=2048&quality=lossless =80x)
>[name=ke] 

>![](https://cdn.discordapp.com/attachments/415857001718087710/1117763502212403200/1b21c5ce00225a2d9645fa28cd84db8a.png =200x)
>[name=ArDai]

>![image alt](https://cdn.discordapp.com/attachments/911889529705742365/1117395788771889162/F.gif =200x)
>[name=賴教授前來上香辛酸畫面流出]

>![](https://cdn.discordapp.com/emojis/1014054379810213929.gif?size=2048&quality=lossless =80x)
>[name=ice] 


Source:
![](https://cdn.discordapp.com/emojis/1014054379810213929.gif?size=2048&quality=lossless =80x)



