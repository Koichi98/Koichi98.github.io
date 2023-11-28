# Distributed Systems 
## Chapter7 Mutual Exclusion
### 1.Introduction
排他制御は並列プログラミングにおいて根本的な問題であり、様々なコンテキストで研究されてきた。
大きく分けて共有メモリに着目したものとmessage passingの2つである。
<br>
<br>
<br>

### 2.Message-Passing Systemsに対する解法
前提条件
- n プロセスによる分散システム
- それぞれのプロセスが他のすべてのプロセスに直接通信できる全結合グラフ
- 各プロセスはCritical Section (CS) に定期的に進入し、CSを実行し、CSを出て他の作業を行う

問題定義 - 以下の3つの条件を満たすようなプロトコルを考えたい
- ME1: [Mutual exclusion] 任意のタイミングで、最大一つのプロセスがCS内に存在し得る (safety)
- ME2: [Freedom from deadlock] あらゆる場面において(In every configuration)、最低でも一つのプロセスが動作してCS内に進入可能である。(safety)
- ME3: [Progress] CSに進入しようとしているすべてのプロセスが最終的にはそれに成功する (liveliness)

#### 2.1 Lamportの解法
一言で言うと「全員にリクエスト(タイムスタンプ付き)を送って、全員からackが返ってきてかつ自分のタイムスタンプが最小なら実行できる」

- 全結合グラフで動作
- 通信チャネルはFIFO
- 各プロセスはrequest-queue Qを保持
以下の5つのルールによって説明される
- LA1: CSへの侵入のリクエストをする際には、プロセスはタイムスタンプが付与されたリクエストを他のすべてのプロセスに送信し、自分のQにもそのリクエストを加える。
- LA2: プロセスがリクエストを受信した際には、それをQに加える。自身がCSにいない場合にはタイムスタンプが付与されたackを送信者に返し、CSにいる場合にはCSから退出してからそれを行う。
- LA3: プロセスは(1)自分の持つQにおいて自身のリクエストのタイムスタンプが他のすべてのリクエストのものよりも小さく、かつ(2)そのリクエストに対してのackを他のすべてのプロセスから受け取った際にCSに進入する。
- LA4: CSから退出するときには、(1)当該リクエストを自身のQから削除し、また(2)タイムスタンプが付与されたreleaseメッセージを他のすべてのプロセスに送信する。
- LA5: プロセスがreleaseメッセージを受け取った際には、対応するリクエストを自身のQから削除する。

Message Complexity - CSに進入して退出するのに必要なメッセージの数
各プロセスは(n-1)メッセージをリクエストに、(n-1)メッセージをackとして受け取り、さらに(n-1)メッセージをrelease messageとして退出時に送信するので3(n-1)。

ちなみに、同時にリクエストしたときに、遅いほうjが出したリクエストへの速い方iからのackが、iからのリクエストよりも早くjに届く可能性があるので通信チャネルはFIFOじゃないとだめ。
<br>
<br>

#### 2.2 Ricart-Agrawalaの解法
Lamportの解法を改善。(通信チャネルはFIFOでなくてもよい)<br>
一言で言うと「全員にリクエスト(タイムスタンプ付き)を送って、全員からackが返ってきたら実行できる」

- RA1:CSに進入しようとしている各プロセスはタイムスタンプが付与されたリクエストを他の全てのプロセスに送信する。
- RA2: リクエストを受信したプロセスは以下のどちらかを満たすときのみackを返す。(1)自身がCSに進入しようとしていない。(2)CSに進入しようとしているが、そのタイムスタンプが受信したリクエストに付与されているタイムスタンプよりも大きい。受信したプロセスがすでにCSに進入している場合、もしくは自身のタイムスタンプのほうが小さい場合には、自身がCSを退出するまでリクエストを**バッファする**
- RA3: プロセスは送信したリクエストに対してのackを他のすべてのプロセスから受け取った際にCSに進入する。
- RA4: CSから退出する際には、新たなリクエストを作ったり、他の作業を行ったりする前に、バッファしていたリクエスト全てに対してackを送信する必要がある。

Message Complexity
各プロセスは(n-1)メッセージをリクエストに、(n-1)メッセージをackとして受け取るので2(n-1)。

ME1の証明:<br>
2つのプロセスi,jがCSに同時に進入することができるのはそれらがともに(n-1)個のackを受信した場合だが、RA2によりi,jが互いにackを送り合うことはできないのでこれはありえない。<br>
<br>
ME2とME3の証明：<br>
プロセスをノードとする有向グラフを考える。jがすでにCS内にいる、もしくはjのリクエストのタイムスタンプがiのよりも小さい場合に、iからjへの有向辺を加える。このとき、Gは自明にacyclic。<br>
CSに進入しようとするプロセスiは、少なくとも一つi->jの辺があるプロセスjが存在するときに待ち続け、jはCSを完了するまでiにackを送らない。RA3より、out-degreeが0となるノードのプロセスはすべてのackを受け取り、CSに進入する。CSから退出する際には、ackを送信し、すなわちすべての待っているノードのout-degreeがデクリメントされる(RA4)。<br>
よって、out-degreeの少ない順にCSに進入する。Gがacyclicであるためにデッドロックは起きず、またすべての待っているプロセスは有限回でCSに進入する。
<br>
<br>

#### 2.3 Maekawaの解法
同じpermission basedのアルゴリズムでも、LamportやRicart-Agrawalaの解法ではすべてのプロセスから許可を必要としていたが、実際は全てから許可をもらう必要はないのでは？というのがベースのアイデア。
各プロセスは個々に定義されるsubset $S_i$からの許可を取得することで、CSへの進入を許可される<br>
$S_i$は以下を満たす
- $\forall i,j \leq i \leq n-1:: S_i \cap S_j \neq \varnothing$
- プロセスiとjがともにCSに入りたいときには、 $S_i \cap S_j$ が調停者としてどちらか一方だけを選んでackを送り、もう一方を延期させる
- $i \in S_i$. 自分自身からは当然CSに侵入するにあたってackを受け取ることとする。メッセージコストはかからない
- 各プロセスは同じ数だけの部分集合に含まれる。各プロセスが同じプロセス数分の調停者となっていることはシステムの対称性を形成する。

以下の5つのルールによって説明される
- MA1: CSに侵入する際に、プロセスiはタイムスタンプが付与されたリクエストを $S_i$ のすべてのプロセスに送る
- MA2: 自身がCSに入っておらず、リクエスト受け取ったプロセスはタイムスタンプの値が最も小さいものにackを送る。その上で、自分自身をそのプロセスに対してlockし、他のすべてのリクエストをqueueに入れる。もし、CSに入っていた場合にはこの操作を自身がCSを出るまで延期する。
- MA3: プロセスiはすべての $S_i$ からのackを受け取るとCSに入る
- MA4: CSを退出する際にはプロセスiはreleaseメッセージを $S_i$ のすべてのプロセスに送る
- MA5: releaseメッセージをプロセスiから受け取るとプロセスはlockを解除し、そのリクエストの削除を行った上で、また最小のタイムスタンプの値のものにackを送る。

しかし、このアルゴリズムはデッドロックする可能性がある。例えば以下のような場合。
- $S_0 = {0,1,2}$のうち、プロセス0と2がackを0に送るが、プロセス1は1に送る
- $S_1 = {1,3,5}$のうち、プロセス1と3がackを1に送るが、プロセス5は2に送る
- $S_2 = {2,4,5}$のうち、プロセス4と5がackを2に送るが、プロセス2は0に送る
このとき、0は1を、1は2を、2は0を待つことになり、デッドロック。

この問題の本質はプロセスがリクエストを受け取ったときに、他によりタイムスタンプが小さいものがこの後来るかが分からないという点にある。
それゆえ、実際に $S_i$ で発生しているリクエストに対して、本当にタイムスタンプが小さいものに対してackをするとは限らず、デッドロックが起きてしまう。
2つの回避策がある。
##### Global FIFO
各プロセスが受け取るリクエストは必ずタイムスタンプ順であるという仮定をおく。(通信チャネルがFIFOになっている)。
これがME2やME3を満たすようになるのだが、実世界では実現が難しい。

##### Maekawa2
request, ack, release, failed, inquire, relinquishというsignalを用いて実現。<br>
//TODO
<br>
<br>
<br>

### 3 Token-Passing Algorithms
トークンと呼ばれるものをCSへの進入の許可証として利用し、プロセス間で渡し合うことという方法。

#### 3.1 Suzuki-Kasami Algorithm
- 全結合グラフに対して定義されるアルゴリズム
- トークンを持たないプロセスiはCSに進入したい際に、リクエスト(i,num)を送信する。(numはsequence number)
- 各プロセスは整数配列req[0.. n-1]を持ち、req[j]はプロセスjから受信した直近のリクエストのsequence number
- トークンが持つデータ構造は以下の2つ。
  - (1)プロセスjによって直近で実行された際のsequence numberをlast[j]が表す、整数配列last[0.. n-1]
  - (2)トークンを待っているプロセスidをためておくためのqueue Q
     
[図解pg4](https://docs.google.com/presentation/d/19KHvjLuHe5_7m8bEg8hPUKBa8Wne0vkR_EqGoyr8v4I/edit?usp=sharing)


アルゴリズム
- 進入時
  - プロセスiがCSに進入したいがトークンを持たない際に、プロセスiは自身のreq_i[i]のsequence numberをインクリメントし、トークンを要求するためにリクエスト(i,sn)を他のすべてのプロセスに送信する。
  - プロセスjがリクエスト(i,sn)をプロセスiから受け取ると、req_j[i]=max(req_j[i],sn)を行ってreq_j[i]を更新する。
- CS実行時
  - プロセスiはトークンを取得したらCSを実行
- 退出時
  - last[i] = req_i[i]を行ってreq_i[i]が実行されたことを示す
  - 各プロセスj(0..i-1,i+1..n-1)に対して、もしQにjが入っておらず、かつreq_i[j]=last[j]+1であれば、jをQに追加してプロセスjがリクエストしていることを反映させる
  - 上記の更新を行った上で、Qが空でなければQからidを一つ取り出してそのプロセスにトークンを送る
  - Qが空の場合にはそのままトークンを保持する

Message Complexity
各プロセスは(n-1)メッセージをリクエストに、1メッセージをtokenとして受け取るのでn。
<br>
<br>

#### 3.2 Raymond's Algorithm
- トークンを持つプロセスをrootとし、辺がrootの方向を指すような有向木としてプロセス群を構成する
- 各プロセスはqueue Qを持つ
  - このqueueは隣接するノード(プロセス)からきたリクエストでまだトークンを送られていないプロセスのものを保持する
- 各プロセスはholder変数をもつ
  - i -> jの有向辺がある場合、jはiのholderである

[図解pg13](https://docs.google.com/presentation/d/19KHvjLuHe5_7m8bEg8hPUKBa8Wne0vkR_EqGoyr8v4I/edit?usp=sharing)

アルゴリズム
- 進入時
  - プロセスiがトークンを持っておらず、かつ**Qが空**であり、CSに進入したい際にはリクエストをholderのプロセスに送る。リクエストを送ったらそのリクエストを自身のQに追加する。
  - rootへのパス上のプロセスjがプロセスiのリクエストを受け取ると、そのリクエストを自身のQに追加する。また、以前に受け取った任意のリクエストをholderに送っていなければ、自身のholderに送る
  - rootプロセスrがリクエストを受け取ると、リクエスト元のプロセスにトークンを送り、holderをそのプロセスにセットする
  - トークンを受け取ると、プロセスjはQの先頭を削除し、そのエントリに示されているプロセスにトークンを送る。また、holderをその送り先に更新する。
  - Qを削除した後もまだQがからでなければリクエストをholderに向かって送ることでトークンを送り返してもらうようにする
- CS実行時
  - プロセスはトークンを受け取り、自分のリクエストが自身のQの先頭にある場合にはCSを実行する
- 退出時
  - Qが空でなければ、Qから先頭を削除してそのエントリに示されているプロセスにトークンを送り、holderをそのプロセスに更新する
  - 上記を行った上でまだQが空でなければ、リクエストをholderに送ることでトークンを送り返してもらうようにする
 (結局holderを更新する作業は(有向辺を逆さにする操作)atomicにしないとバグるはず？)

Message Complexity
木の中の任意の2つのノードの平均距離はO(logn)なのでo(logn)
<br>
<br>
<br>


### 4 Solutions on the Shared Memory Model
共有メモリに対するatomic read and write operationsが前提 <br>
**The requirements of a correct solution on the shared-memory model are similar to those in the message passing model, except that fairness is specified as freedom from livelock or freedom from starvation: no process can be indefinitely prevented from entering its CS by other processes in the system.**

#### 4.1 Peterson's Algorithm
2プロセスにおける解法
- プロセス0とプロセス1
- 各プロセスiにはBooleanのflag[i]があり、これはどのプロセスからもread可能だがwriteはiにのみ可能

```C 
define flag[0],flag[1]: shared Boolean;
initially both are false;

# process0 
do{
  flag[0] = true;
  while (flag[1]);

    critical section;

  flag[0] = false;
    
  } while(true);


# process1
do{
  flag[1] = true;
  while (flag[0]);

    critical section;

  flag[1] = false;
    
  } while(true);

```
上の解法はsafetyを保証するが、deadlock-freeではない。2つのプロセスのflagが同時にtrueになってしまうとお互いがお互いを待つことになる。
そこで整数型のintを共有変数として導入することでこれを解決する。turnを更新したプロセス自身はCSへの進入を待つという原理。
```C 
define flag[0],flag[1]: shared Boolean;
turn: shared integer
initially flag[0],flag[1] are false; turn = 0 or 1

# process0 
do{
  flag[0] = true;
  turn = 0
  while (flag[1] && turn == 0);

    critical section;

  flag[0] = false;
    
  } while(true);


# process1
do{
  flag[1] = true;
  turn = 1
  while (flag[0] && turn == 1);

    critical section;

  flag[1] = false;
    
  } while(true);

```
ちなみに、nプロセスへの拡張は以下の通り
```C
define flag,turn: array[0..n-1] of shared Boolean;
initially \forall k:flag[k] = 0, and turn = 0

process i
do{
  j = 1;
  do{
    flag[i] = j;
    turn[j] = i;
    while(\exist \neq i:flag[k] /geq j \land turn[j] = i);
    j = j + 1;
  }while(j != n-1)

    critical section;

    flag[i] = false;
    
  } while(true);

```















