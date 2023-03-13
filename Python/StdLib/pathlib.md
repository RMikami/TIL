# フォルダ内のファイル名を取得
下記コードを実行すると、glob()メソッドで検索したファイル名を取得できる。

```python
>>> from pathlib import Path
>>> p = Path("./image")
# Windowsだと、pはWindowsPath型となる。
# これは文字列として扱われない。
# 文字列として扱いたい場合は、str(p)でstr型に変換する必要がある。
>>> for i in p.glob("*"):
...     print(i.name)
```

glob()メソッドはジェネレータオブジェクトを返す。

```python
>>> type(p.glob("*"))
<class 'generator'>
```

osモジュールでも下記のように同様な処理を行うことができるが、listdir()メソッドはリストオブジェクトを返す。リストオブジェクトは、ジェネレータオブジェクトよりもメモリ消費量が多いため、大量のデータ処理には向かない。

```python
>>> import os
>>> p = os.listdir("./image")
>>> type(p)
<class 'list'>
>>> for i in p:
>>>     print(i)
```

# パスの連結
Pathクラスに連結したいディレクトリの文字列を渡せば、'/'演算子を用いることで下記ようにパスを連結できる。

```python
image/aaa/bbb/ccc
>>> from pathlib import Path
>>> path = Path("./image")
>>> print(path1 / Path("aaa/bbb/ccc"))
image/aaa/bbb/ccc
```