---
layout: post
title: Draft
---
*(This is translate post from Japanese [original post](http://yohhoy.hatenablog.jp/entry/2014/09/23/193617) to English, under [CC-BY 4.0](https://creativecommons.org/licenses/by/4.0/deed).)*

<!-- 多くのプログラミング言語では、マルチスレッド処理向け同期プリミティブとして「ミューテックス(mutex)」と「**条件変数(condition variable)**」を提供しています。
ミューテックスは排他制御機構として有名ですし、その動作もロック(lock)／アンロック(unlock)と比較的理解しやすいため、不適切に利用されるケースはあまり無いと思います。
一方、条件変数の動作仕様はしばしば誤解され、不適切な利用による並行性バグに悩まされるケースが多いようです。 -->

A lot of programming languages provides "mutex" and "**condition variable**" for synchronization primitives in multithreaded programing.
Mutex is known as a mutual excection mechanism, which its lock and unlock behaviors are not hard to undertand, there might be a rare case of inappropriate usage.
On the other hand, some programmers have a misunderstanding about the specification of condition variable, he/she often suffers from concurrency bug because of inappropriate usage.

<!-- 本記事では、スレッドセーフなFIFO(First-In-First-Out)キューを段階的に実装していく事例を通して、条件変数の適切な使い方について説明していきます。
例示コードではC++11標準ライブラリを用いますが、Pthreads(POSIX Threads)やC11標準ライブラリへは単純に読み替え可能です。
また、C++11標準ライブラリからBoost.Threadライブラリへの対応関係は自明なはずです。 -->

In this post, I'll explain the proper use of condition variable, through step-by-step implementation of thread-safety FIFO(First-In-First-Out) queue.
We'll use C++11 Standard Library in example code, you could rewrite easily with Pthreads(POSIX Threads) or C11 Standard Library.
Also you can rewrite trivially with Boost.Thread library from C++11 Standard Library.

<!-- 本記事中のソースコードはBoost Software License 1.0で公開しています。 -->

All source code in this post are distributed under Boost Software License 1.0.

<!-- Pthreadsに馴染みがある人向けの補足：`pthread_cond_signal`→`notify_one`、`pthread_cond_broadcast`→`notify_all`へと読み替えてください。 -->

For familiar with Pthreads: Replace `pthread_cond_signal` to `notify_one`, `pthread_cond_broadcast` to`notify_all`.


# Let's start off

<!-- 次の仕様を満たす、スレッドセーフ・FIFOキュー`mt_queue`を実装していきます。 -->

We start to implement thread-safe FIFO queue `mt_queue` which meets the following requirement.

<!-- クラス仕様: int型要素を最大10個まで格納可能なFIFO操作コンテナ。各メンバ関数を異なるスレッドから同時に呼び出してもよい（スレッドセーフ）。
push操作: キューに新しい要素を1つ追加する。キューに空き容量が無ければ、空きができるまで呼び出し元スレッドをブロッキングする。
pop操作: キューから最も古い要素を1つ取り出す。キュー内に要素が存在しなれば、要素が追加されるまで呼び出し元スレッドをブロッキングする。 -->

<dl>
<dt>Specification:</dt><dd>FIFO-queue container which contains int value up to 10 elements. User can invoke each its member function from different threads simultaneously. (thrad-safe)</dd>
<dt>Push operation:</dt><dd>Add one new element to queue. If the queue is full, caller thread is blocked until there are some rooms.</dd>
<dt>Pop operation:</dt><dd>Get oldest element from queue. If the queue is empty, caller thread is blocked until there are some elements.</dd>
</dl>

<!-- とりあえずロジックだけ実装すると、下記のような感じでしょうか。 -->

Let's just implement the behavior of FIFO-queue, here is initial version:

```cpp
#include <queue>

class mt_queue {
  static const int capacity = 10;
  std::queue<int> q_;
public:
  void push(int data) {
    while (q_.size() == capacity)
      ;
    q_.push(data);
  }
  int pop() {
    while (q_.empty())
      ;
    int data = q_.front();
    q_.pop();
    return data;
  }
};
```

<!-- このコードはマルチスレッド動作について一切考慮していないため、異なるスレッドから`push()`/`pop()`メンバ関数が同時に呼び出されると、データ競合(data race)により正常に動作しません。
偶然に正常動作する可能性もゼロではありませんが、本質的に“壊れた”コードであると認識すべきです。
マルチスレッド処理プログラミングでは並行処理設計が何よりも重要です。
1万回に1回といった低頻度でのプログラムクラッシュ、システム高負荷状態でのみ発生するデッドロック、デバッグビルドは問題無いのにリリースビルドだと処理ストールといった、ステキなデバッグ体験をしたいのでない限り、“とりあえず動けばOK”という方針はお勧めできません。 -->

In above code, we did not take account of multithreding at all.
When difference threads call `push()`/`pop()` member function at the same time, the code doesn't run correctly because it has data race.
It might run rarely as we expected, but we should acknowledge that it is essentially "broken" code.
The concurrency architechture is of utmost importance in the design of multithreaded program.
It cann't be recommended "Works as expected, it's OK" style, unless you would like wonderful experience to debug crash occurs with terribly low frequency, or deadlock under heavily-loaded system status, or stall process in Release Build not in Debug Build.


# Step0: Mutex + busy-loop wait

<!-- マルチスレッド処理プログラミングでは、ミューテックスによる排他制御が全ての基本となります。
複数スレッドから同時更新される可能性のある変数アクセスは、ミューテックスによるロックで全て保護する必要があります。
まずは、最も単純なビジーループ(busy-loop; busy-wait)方式で仕様通り実装してみましょう。 -->

The *controlling mutual execution with mutex object* is the basis of multithreaded program for everything.
We should procect by mutex locking all variable accesses which multiple threads would modify it simultaneously.
First, let's implement requirements with simplest busy-loop (busy-wait) mechanism.
&#91;[mt_queue0.cpp](https://gist.github.com/yohhoy/d305a6c5249c55ed89a3#file-mt_queue0-cpp)&#93;

```cpp
// snippet of mt_queue0.cpp
#include <mutex>

class mt_queue {
  //...
  std::mutex mtx_;  // <-- introduce mutex
public:
  void push(int data) {
    std::unique_lock<std::mutex> lk(mtx_);  // acquire lock
    while (q_.size() == capacity) {      lk.unlock();  // release lock temporary
      std::this_thread::yield();
      lk.lock();    // re-acquire lock
    }
    q_.push(data);
  }  // release lock
  int pop() {
    std::unique_lock<std::mutex> lk(mtx_);  // acquire lock
    while (q_.empty()) {
      lk.unlock();  // release lock temporary
      std::this_thread::yield();
      lk.lock();    // re-acquire lock
    }
    int data = q_.front();
    q_.pop();
    return data;
  }  // release lock
};
```

メンバ関数実装のwhileループ中で、ロック一時解放`lk.unlock()`→ロック再獲得`lk.lock()`を忘れずに記述しましたか？
このロック一時解放を行わないと、`push()`/`pop()`の同時呼び出しでデッドロックが発生します。
例えば、1)キューが空っぽのとき`pop()`呼び出し→2)ロックを保持したまま`while (q_.empty())`ループ→3)別スレッドの`push()`呼び出し先頭で永久にロック獲得待ちという状況です。
もう1パターン自明なデッドロックシナリオがありますが、こちらは練習問題として考えてみて下さい。ヒント：push/popの状況を逆転

また、ロック一時解放領域で`yield()`を呼び出していますが、この処理を省略しても`mt_queue`クラスは仕様通り動きます。
処理系によっては`yield()`の有無で動作性能が改善するでしょうが、ビジーループがCPUリソースを浪費する非効率な方式であることに変わりはありません。
（ビジーループが常に不適切ということではなく、OSカーネルやハードウェア制御ドライバなどの低レイヤ・プログラミングでは有用なケースもあります。
ただし、通常のアプリケーションプログラミングでは基本的に避けるべきでしょう。）

# Step1: Waiting and notification with condition variable

それでは、さっそく条件変数を使っていきましょう。
排他制御用のミューテックスはそのままに、新しく条件変数を追加することに注意してください。
ここはよく誤解されるポイントなのですが、セマフォのようにそれ単体で機能する同期プリミティブと異なり、条件変数は必ずミューテックスと紐付けて利用しなければ意味をなしません。
[mt_queue1.cpp](https://gist.github.com/yohhoy/d305a6c5249c55ed89a3#file-mt_queue1-cpp)

```cpp
// snippet of mt_queue1.cpp
#include <mutex>
#include <condition_variable>
 
class mt_queue {
  //...
  std::mutex mtx_;
  std::condition_variable cv_;  // <-- add conditional variable
public:
  void push(int data) {
    std::unique_lock<std::mutex> lk(mtx_);
    while (q_.size() == capacity) {
      cv_.wait(lk);    // wait until change of element count
    }
    q_.push(data);
    cv_.notify_all();  // notify change of element count
  }
  int pop() {
    std::unique_lock<std::mutex> lk(mtx_);
    while (q_.empty()) {
      cv_.wait(lk);    // wait until change of element count
    }
    int data = q_.front();
    q_.pop();
    cv_.notify_all();  // notify change of element count
    return data;
  }
};
```

このコードでは条件変数オブジェクト`cv_`を、「キュー内の要素数に何らかの変更があった」イベントの待機／通知処理に利用しています。
このときCPUリソース利用の観点からは、条件変数を用いた待機ループ`while (condition) { cv_.wait(lk); }`は、ビジーループの効率的な実装と解釈できます。
つまりクラス利用者の視点からは、ビジーループ実装と条件変数実装とでFIFOキューの動作仕様に違いはなく、プログラム実行時の振る舞いだけが異なるものとなります。

```cpp
// Step0(mt_queue0.cpp)
while (condition) {
  lk.unlock();
  std::this_thread::yield();
  lk.lock();
}
```
```cpp
// Step1(mt_queue1.cpp)
while (condition) {
  cv_.wait(lk);
}
```

条件変数に対する待機`cv_.wait(lk)`は、内部的には“ロック一時解除`lk.unlock()`＋条件変数へ通知があるまで自スレッドを休止＋ロック再取得`lk.lock()`”処理を行います。
ビジーループ版では他スレッドに実行機会のヒントを与える(`yield()`)だけの処理が、条件変数版では論理的に意味のあるイベントが起きるまで、つまり明示的に条件変数への通知が行われるまで自スレッドを休止状態とし、CPUリソースを他スレッドへ明け渡す処理となります。

# Step2: Improve runtime efficiency

Step1では待機処理に条件変数を利用する事で、CPUリソースの浪費を抑えることができました。
Step2では、より効率的な条件変数の利用について検討しましょう。
ここでは下記4つを順番に適用していきます。

1. イベント種別に応じた条件変数の分離
2. 冗長な条件変数通知の削減
3. 条件変数通知先スレッドの限定
4. 条件変数待機を述語(Predicate)版に書き換え

## Step2-1: Separate condtion variable object

Step1では条件変数オブジェクト`cv_`を、「キュー内の要素数に何らかの変化があった」イベントに対応付けていました。
この設計では、あらゆるキュー操作によって条件変数への通知が発生するため、必要のない冗長な通知処理が行われています。
push操作は「キューが満杯でない」状態を／pop操作は「キューが空でない」状態を待機すれば良いので、両イベントに直接対応する条件変数に分離しましょう。
新しい設計では1つのミューテックス`mtx_`に対して、2つの条件変数オブジェクト`cv_nofull_`, `cv_noempty_`を紐付けます。

```cpp
// Step1(mt_queue1.cpp)
std::mutex mtx_;
std::condition_variable cv_;
```
```cpp
// Step2-1
std::mutex mtx_;
std::condition_variable cv_nofull_;   // not fullness
std::condition_variable cv_noempty_;  // not empty queue
```

条件変数の設計では、待機側でどんなデータ状態を満たすべきかに基づいて抽出することをお勧めします。

## Step2-2: Reduce redundant notification

`push()`実装の通知処理に着目すると、Step1では条件変数への通知を無条件に行っていました。
本質的には、push操作では「キューが空でなくなった＝少なくとも1個要素が存在する」イベントのみをpop操作へ通知すれば十分なため、通知の発火条件を限定することで無駄な処理を減らし、より効率的な実装とできます。
同様にpop操作では「キューが満杯でなくなった＝少なくとも1要素を追加できる」イベントのみを通知すれば十分です。

```cpp
// Step1: push() member function
q_.push(data);
cv_.notify_all();  // notify change of element count
```
```cpp
// Step2-2: push() member function
bool do_signal = q_.empty();  // iff queue is empty before operation,
q_.push(data);
if (do_signal)
  cv_noempty_.notify_all();   // notify "not empty queue".
```

条件変数への通知処理は、待機側の条件式を満足するよう変更を行ったタイミングを抽出すると、実行時オーバーヘッドを最小化することができます。

## Step2-3: Restrict notify thread

Step1では条件変数への通知処理に`notify_all()`(broadcast)を用いていました。
空キューへの1回のpush操作によって、pop操作を待機中の全スレッドが起こされますが、実際に要素の取り出しに成功するのは1スレッドだけで、それ以外のpop操作要求スレッドは全て待機処理に再突入します。
つまり、push操作が通知すべきpop操作待機スレッドは1つで十分であり、それ以上のスレッドへ通知が行われても冗長ということです。
このようなケースでは`notify_one()`(signal)を利用することで、通知先スレッドを限定して実行効率の改善をはかることができます。

```cpp
// Step1: push() member function
cv_.notify_all();  // notify all waiting threads
```
```cpp
// Step2-3: push() member function
cv_noempty_.notify_one();  // notify one of threads waiting on pop()
```

このような`notify_all()`から`notify_one()`への変更は、i)通知先が最大で1スレッド かつ ii)通知を受信するスレッドの待機条件を常に満たす場合 にのみ安全に行えます。
本記事で設計したFIFOキューの場合、push操作からの通知を受信した任意のpop操作スレッドは、必ず待機ループを抜けてpop操作を完了すると保証できるため、`notify_one()`へと安全に変更できます。
特に条件ii)を正しく考慮していないと、デッドロックといった並行性バグの原因となるため十分注意してください。

追記：`notify_one()`により生じるデッドロックについては、おまけ記事「条件変数とデッドロック・パズル（出題編）（解答編）」も参考にしてください。

## Step2-4: Rewrite waiting on condtion variable

最後に、条件変数の待機処理`wait()`を述語(Predicate)をとる2引数オーバーロードに書き換えておきます。
この変更は効率化とは無関係ですが、条件変数オブジェクトの名前(`cv_nofull_`, `cv_noempty_`)と、待機条件式(`q_.size() < capacity`, `!q_.empty()`)の意味を一致させることができます。
また後述Step3でFIFOキュー機能拡張を行うときに、待機条件の論理式を拡張しやすいというメリットもあります。

```cpp
// Step1: push() member function
while (q_.size() == capacity) {
  cv_.wait(lk);
}
// Step1: pop() member function
while (q_.empty()) {
  cv_.wait(lk);
}
```
```cpp
// Step2-4: push() member function
cv_nofull_.wait(lk, [&]{
  return (q_.size() < capacity);
});
// Step2-4: pop() member function
cv_noempty_.wait(lk, [&]{
  return !q_.empty();
});
```

## Summary of Step2

Step2での最終的なコードは次の通りです。
[mt_queue2.cpp](https://gist.github.com/yohhoy/d305a6c5249c55ed89a3#file-mt_queue2-cpp)

```cpp
// snippet of mt_queue2.cpp
class mt_queue {
  //...
  std::mutex mtx_;
  std::condition_variable cv_nofull_;   // condition for not fullness
  std::condition_variable cv_noempty_;  // condition for not empty queue
public:
  void push(int data) {
    std::unique_lock<std::mutex> lk(mtx_);
    cv_nofull_.wait(lk, [&]{     // wait until "not fullness"
      return (q_.size() < capacity);
    });
    bool do_signal = q_.empty();
    q_.push(data);
    if (do_signal)
      cv_noempty_.notify_one();  // notify being "not empty queue"
  }
  int pop() {
    std::unique_lock<std::mutex> lk(mtx_);
    cv_noempty_.wait(lk, [&]{   // wait until "not empty queue"
      return !q_.empty();
    });
    bool do_signal = (q_.size() == capacity);
    int data = q_.front();
    q_.pop();
    if (do_signal)
      cv_nofull_.notify_one();  // notify being "not fullness"
    return data;
  }
};
```

さらなる改善策として、通知処理をミューテックスのロック解放後まで遅延させるという手法もあります。
ただし、この変更によりコードが壊れるコーナーケースが存在すること、また処理系の実装品質が十分ならばあまり効果を期待できないため、本記事では適用しないこととします。
詳細はこちらの記事を参考にしてください。

# Step3: Review mt_queue functionality

Step2までで、性能面について十分考慮されたスレッドセーフ・FIFOキューを実装できました。
Step3では機能面について改めて考えてみましょう。ここでは下記2点を検討します。

1. データ列の終端処理
2. 強制的な処理中断

## Step3-1: End of data stream

最初に設定したFIFOキュー仕様では、pushデータ列の終端処理について特別の考慮をしませんでした。
いくぶん不恰好ですが、例えば`-1`を特殊な“データ終端マーカ”値とみなすことも出来るでしょう。

```cpp
// mt_queue class user code
void producer_thread(mt_queue& mq) {
  for (int i = 1; i <= N; ++i)
    mq.push(i);
  mq.push(-1);  // notify end of data stream
}
void consumer_thread(mt_queue& mq) {
  int v;
  while ((v = mq.pop()) > 0)
    process_data(v);
}
```

生産者スレッド(`producer_thread`)と消費者スレッド(`consumer_thread`)が1つづつのSingle-Producer/Single-Consumerモデルならこの設計でも十分ですが、Multi-Producer/Multi-Consumerモデルでは面倒な問題が浮上します。
例えば2生産者スレッド＋3消費者スレッドで動作させた場合、FIFOキューには終端マーカが2個しかpushされないため、3番目の消費者スレッドはデータ終端到達を知ることが出来ません。
生産者／消費者スレッド数が動的に変化する場合はもはやお手上げです。

結局、“終端マーカ”方式のようなクラス利用側コード設計に頼るのではなく、FIFOキュー自体に終端到達通知`close()`メンバ関数を追加するのが良いでしょう。
これに伴って、`push()`は終端到達検知時に例外`closed_queue`を送出し、また`pop()`は終端到達状態を返すよう関数シグニチャを変更します。

```cpp
class mt_queue {
  //...
  bool closed_ = false;  // hold end-of-stream
public:
  void close() {  // notify end-of-stream
    std::lock_guard<std::mutex> lk(mtx_);
    closed_ = true;
    cv_nofull_.notify_all();
    cv_noempty_.notify_all();
  }
  void push(int data) {
    std::unique_lock<std::mutex> lk(mtx_);
    cv_nofull_.wait(lk, [&]{
      return (q_.size() < capacity) || closed_;  // add condition
    });
    if (closed_)
      throw closed_queue();  // throw exception
    //...
  }
  bool pop(int& data) {      // change function signature
    std::unique_lock<std::mutex> lk(mtx_);
    cv_noempty_.wait(lk, [&]{
      return !q_.empty() || (q_.empty() && closed_);  // add condition
    });
    if (q_.empty() && closed_)
      return false;          // end-of-stream reached
    data = q_.front();
    //...
    return true;             // midstream (not end-of-stream)
  }
};
```

`pop()`実装の待機条件では`... || (q_.empty() && closed_)`、つまり“キューが空でかつ終端通知がされているとき”に始めてデータ終端と判定すべきです。
また`close()`実装では、全ての条件変数に対して`notify_all()`通知が必要になります。

## Step3-2: Cancellation of process

終端到達通知`close()`によって通常のデータ終端処理を表現できるようになりましたが、ブロッキング中のpush/pop操作に対する処理中断要求`abort()`があるとより実用的になります。
`push()`/`pop()`では中断要求検知時に例外`abort_exception`を送出するよう拡張します。
なお`close()`はブロッキング操作ではないため、中断すべき処理をもちません。

```cpp
class mt_queue {
  //...
  bool aborted_ = false;  // hold cancellation request
public:
  void abort() {  // cancellation request
    std::lock_guard<std::mutex> lk(mtx_);
    aborted_ = true;
    cv_nofull_.notify_all();
    cv_noempty_.notify_all();
  }
  void push(int data) {
    std::unique_lock<std::mutex> lk(mtx_);
    cv_nofull_.wait(lk, [&]{
      return (q_.size() < capacity) || closed_ || aborted_;  // add condition
    });
    if (closed_)
      throw closed_queue();
    if (aborted_)
      throw abort_exception();  // throw exception
    q_.push(data);
    //...
  }
  //...
};
```

`push()`実装の待機条件式`(q_.size() < capacity) || closed_ || aborted_`と、後続の条件分岐式とを比べてみてください。
下記の対応関係を読み取れたでしょうか？
1つの条件変数オブジェクトに複合的な条件を設定する場合、この構造が基本形となるはずです。

```cpp
<ConditionVariable>.wait(lk, [&]{
  return ConditionA || ConditionB || ConditionC;
}):
if (ConditionA) { ProcessA; }
if (ConditionB) { ProcessB; }
if (ConditionC) { ProcessC; }
```

# Goal!

本記事で実装したスレッドセーフ・FIFOキューの最終コードは次の通りです。
[mt_queue3.cpp](https://gist.github.com/yohhoy/d305a6c5249c55ed89a3#file-mt_queue3-cpp)

```cpp
// snippet of mt_queue3.cpp
struct closed_queue : std::exception {};
struct abort_exception : std::exception {};

class mt_queue {
  static const int capacity = 10;
  std::queue<int> q_;
  std::mutex mtx_;
  std::condition_variable cv_nofull_;
  std::condition_variable cv_noempty_;
  bool closed_ = false;
  bool aborted_ = false;
public:
  void push(int data)
  {
    std::unique_lock<std::mutex> lk(mtx_);
    cv_nofull_.wait(lk, [&]{
      return (q_.size() < capacity) || closed_ || aborted_;
    });
    if (closed_)
      throw closed_queue();
    if (aborted_)
      throw abort_exception();
    bool do_signal = q_.empty();
    q_.push(data);
    if (do_signal)
      cv_noempty_.notify_one();
  }
  bool pop(int& data)
  {
    std::unique_lock<std::mutex> lk(mtx_);
    cv_noempty_.wait(lk, [&]{
      return !q_.empty() || (q_.empty() && closed_) || aborted_;
    });
    if (q_.empty() && closed_)
      return false;
    if (aborted_)
      throw abort_exception();
    bool do_signal = (q_.size() == capacity);
    data = q_.front();
    q_.pop();
    if (do_signal)
      cv_nofull_.notify_one();
    return true;
  }
  void close()
  {
    std::lock_guard<std::mutex> lk(mtx_);
    closed_ = true;
    cv_nofull_.notify_all();
    cv_noempty_.notify_all();
  }
  void abort()
  {
    std::lock_guard<std::mutex> lk(mtx_);
    aborted_ = true;
    cv_nofull_.notify_all();
    cv_noempty_.notify_all();
  }
};
```

これでもう条件変数は怖くありません…よね？
