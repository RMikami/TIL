concurrent.futuresモジュールは、並行処理を行うための標準ライブラリである。並行処理で実行したい処理を渡すと、その処理を後述するfutureオブジェクトにカプセル化し、非同期処理として実行してくれる。

# ThreadPoolExecutorクラス
マルチスレッドで非同期処理を行う場合、concurrent.futures.ThreadPoolExecutorクラスを利用する。<br>
I/Oを伴う処理では、その処理にかかる時間はハードウェアやネットワークなどの外部に依存する。そのため、プログラムを書き換えても個々の処理の高速化は期待できない。しかし、複数の処理がある場合は非同期実行でマルチスレッド化すると、通信中の待ち時間を有効活用できて合計時間を短縮できる。<br>
マルチプロセス処理でも合計時間を短縮できるが、スレッドにはプロセスよりもオーバーヘッドが小さいというメリットがある。<br>
例として、複数のサイトのトップページをダウンロードする処理を考える。ダウンロード処理は待ち時間が発生するI/O処理の典型である。比較のため、まずは逐次処理で実装し、その後マルチスレッド処理に変更する。

```python
from hashlib import md5
from pathlib import Path
from urllib import request

urls = [
    'https://twitter.com',
    'https://facebook.com',
    'https://instagram.com'
]

def download(url):
    req = request.Request(url)
    name = md5(url.encode('utf-8')).hexdigest()
    file_path = './' + name
    with request.urlopen(req) as res:
        Path(file_path).write_bytes(res.read())
        return url, file_path
```

## 逐次処理で実装
上記を逐次処理で行う場合の時間を計測する。

```python
from functools import wraps
import time

def elapsed_time(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        v = f(*args, **kwargs)
        end = time.perf_counter()
        print(end - start)
        return v
    return wrapper

@elapsed_time
def get_sequential():
    for url in urls:
        print(download(url))

get_sequential() # 2.0219604919998346秒
```

## マルチスレッドで実装
上記処理をマルチスレッドで平行化する。マルチスレッド化でリクエストのレスポンスを待つことなく、次々とリクエストを投げているため実行時間は短縮されている。

```python
from concurrent.futures import (ThreadPoolExecutor, as_completed)

@elapsed_time
def get_multi_thread():
    with ThreadPoolExecutor(max_workers=3) as executor:
        futures = [executor.submit(download, url) for url in urls]
        for future in as_completed(futures):
            print(future.result())

get_multi_thread() # 0.9271178809999583秒
```

ThreadPoolExecutorクラスのインスタンスはコンテキストマネージャーになっているので、with文を利用できる。また、インスタンス化の際には引数max_workersで最大スレッド数を指定できる。<br>
非同期実行する処理はメソッドexecutor.submit()で登録する。もし呼び出し時に渡したいパラメータがある場合は、第二引数以降で指定する。<br>
非同期処理の実行結果は、executor.submit()の戻り値のメソッドresult()を呼び出すと取得できる。ただし、まだ処理が完了していない場合には結果がNoneとなるため、as_completed()関数を利用することで処理が完了したものから順に返すようにしている。

### マルチスレッドの注意点
逐次処理では問題なく動くコードであっても、マルチスレッドにすると期待通りに動かなくなる場合がある。<br>
マルチスレッドでの動作に問題がある実装をする。下記のコードは、2つのスレッドがそれぞれ1,000,000回ずつカウンタをインクリメントするものである。

```python
from concurrent.futures import (ThreadPoolExecutor, wait)

class Counter:
    def __init__(self):
        self.count = 0
    def increment(self):
        self.count = self.count + 1

def count_up(counter):
    for _ in range(1000000):
        counter.increment()

counter = Counter()
threads = 2
with ThreadPoolExecutor() as e:
    futures = [e.submit(count_up, counter) for _ in range(threads)]
    done, not_done = wait(futures)

print(f'{counter.count=:,}') # 1,602,716
```
上記を実行すると、処理完了後のカウンタの値が2,000,000になっていないことがわかる。この原因となっているコードは、インクリメントを行っている"self.count = self.cont + 1"の1行である。コードは1行であるが、その裏では「現在の値を読み取る」「1を足す」「結果を代入する」の一連の処理が行われている。この一連の処理の途中でスレッドが切り替わってしまうと、インスタンス変数self.countに2つのスレッドが同時にアクセスする状況が発生し、不整合な状態が生じる。

### スレッドセーフな実装
上記問題を解決するために、スレッドセーフなカウンタを実装する。ここでは、threading.Lockオブジェクトを使い、ロックによる排他制御を取り入れる。ロックによる排他制御の仕組みはシンプルで、ロックを獲得したスレッドのみが処理を実行できる。あるスレッドがロックを獲得している場合、そのロックが解放されるまでは、他のスレッドによるロックの獲得はブロックされる。したがって、排他制御を行いたい箇所でロックを獲得し、処理が終わったら速やかにロックを解放する。ロックの解放漏れを防ぐためにも、Lockオブジェクトはwith文とともに使うのが良い。

```python
from concurrent.futures import (ThreadPoolExecutor, wait)
import threading

class ThreadSafeCounter:
    lock = threading.Lock()
    def __init__(self):
        self.count = 0
    def increment(self):
        with self.lock:
            # 制御したい一連の処理をこのブロック内に書く
            self.count = self.count + 1

def count_up(counter):
    for _ in range(1000000):
        counter.increment()

counter = ThreadSafeCounter()
threads = 2
with ThreadPoolExecutor() as e:
    futures = [e.submit(count_up, counter) for _ in range(threads)]
    done, not_done = wait(futures)

print(f'{counter.count=:,}') # 2,000,000
```

上記コードは、「現在の値を読み取る」「1を足す」「結果を代入する」の一連の処理に対して排他制御を導入したものである。したがって、このコードはスレッドセーフな実装になっており、カウンタの値は期待通りの2,000,000になる。

# ProcessPoolExecutorクラス
マルチプロセスで非同期処理を行う場合、concurrent.futures.ProcessPoolExecutorクラスを利用する。<br>
マルチプロセス処理は暗号化(復号)や次元数の大きい行列演算などのCPUバウンドな処理に有効である。例として、フィボナッチ数列を計算する関数を実装する。

```python
import sys

def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, b + a
    else:
        return a

def main():
    n = int(sys.argv[1])
    print(fibonacci(n))

if __name__ == "__main__":
    main()
```

fibonacci()関数は、引数nを受け取り、n + 1番目のフィボナッチ数列の値を返す関数である。ここでは、高負荷をかけるためにn = 1000000を指定する。

## 逐次処理で実装
上記を逐次処理で行う場合の時間を計測する。計算する回数はCPUのコア数に応じて調整する。get_sequential()関数を追加し、逐次処理でfibonacci()関数を呼び出す。

```python
import sys
import os
import time
from functools import wraps

def elapsed_time(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        v = f(*args, **kwargs)
        end = time.perf_counter()
        print(end - start)
        return v
    return wrapper

def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, b + a
    else:
        return a

@elapsed_time
def get_sequential(nums):
    for num in nums:
        print(fibonacci(num))

def main():
    n = int(sys.argv[1])
    nums = [n] * os.cpu_count()
    get_sequential(nums)

if __name__ == "__main__":
    main()
```

私の環境では関数os.cpu_count()が8を返すため8回計算が行われ、合計で約71秒かかった。

## マルチプロセスで実装
上記処理をマルチプロセスで並列化する。ProcessPoolExecutorクラスの使い方はThreadPoolExecutorクラスと同じである。

```python
import sys
import os
import time
from functools import wraps
from concurrent.futures import (ProcessPoolExecutor, as_completed)

def elapsed_time(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        v = f(*args, **kwargs)
        end = time.perf_counter()
        print(end - start)
        return v
    return wrapper

def fibonacci(n):
    a, b = 0, 1
    for _ in range(n):
        a, b = b, b + a
    else:
        return a

@elapsed_time
def get_multi_process(nums):
    with ProcessPoolExecutor() as e:
        futures = [e.submit(fibonacci, num) for num in nums]
        for future in as_completed(futures):
            print(future.result())

def main():
    n = int(sys.argv[1])
    nums = [n] * os.cpu_count()
    get_multi_process(nums)

if __name__ == "__main__":
    main()
```

pythonコマンドに渡すスクリプト、つまりメインモジュールでマルチプロセス処理を行う場合は、マルチプロセスの開始処理をif \_\_name__ == "\_\_main__":ブロックで保護する。これは、新たに起動されるPythonインタプリタがメインモジュールを安全にインポートできるようにするためである。<br>
上記を実行すると計測時間は約20秒となり、逐次処理と比べて所要時間が短くなった。

### マルチプロセスの注意点
マルチプロセスでは、マルチスレッドとは異なる注意点がある。これらは主にプロセスの仕組みやプロセス間通信に起因する。<br>
ProcessPoolExecutorクラスは、キューと呼ばれるデータ構造を利用してプロセス間でオブジェクトの受け渡しを行っている。ProcessPoolExecutorクラスが使うキューはmultiprocessing.Queueクラスで実現されており、このキューに追加されるオブジェクトはpickleと呼ばれる形式でシリアライズされている。つまり、ProcessPoolExecutorクラスを使ったマルチプロセス処理では、pickle化できるオブジェクトしか実行したり、返したりすることができない。pickle化できないオブジェクトの一例に、lambda式で定義したオブジェクトがある。
また、マルチプロセスでは、乱数を扱う場合も注意が必要である。下記は、numpy.random.random()関数を使ってそれぞれのプロセス内で乱数を生成するコードである。

```python
from concurrent.futures import (ProcessPoolExecutor, as_completed)
import numpy as np

def use_numpy_random():
    # 乱数生成器を初期化する場合はこの行を実行する
    # np.random.seed()
    return np.random.random()

def main():
    with ProcessPoolExecutor() as e:
        futures = [e.submit(use_numpy_random) for _ in range(3)]
        for future in as_completed(futures):
            print(future.result())

if __name__ == "__main__":
    main()
```

実行すると、生成された3つの乱数は全て同じ値になってしまった。これはプロセスの開始方式がfork(親プロセスを複製して子プロセスを生成する方式)となっている場合に生じる現象で、Unix環境ではforkがデフォルト値となっている。この場合は各プロセスの中でnumpy.random.seed()関数を用いて、乱数生成器の初期化を行うと解決する。なお、標準ライブラリrandomモジュールのrandom.random()関数を使った場合は、プロセスのフォークの際に自動で乱数生成器が初期化されるのでこの心配はいらない。

## マルチスレッドで実装
マルチスレッドにした場合の所要時間も確認してみる。ProcessPoolExecutorクラスをThreadPoolExecutorクラスに変更するだけで、マルチプロセスからマルチスレッドに変更できる。<br>
実行すると所要時間は約138秒となった。これはGILの制約により、CPUバウンドな処理に対してはマルチスレッドによる高速処理ができないためである。逐次処理より遅くなったのは、スレッド切り替えによるオーバーヘッドのためである。