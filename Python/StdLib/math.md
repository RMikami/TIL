# math.floor() , math.ceil()
小数点以下の切り捨て、切り上げができる。<br>
math.floor()を実行すると切り捨て、math.ceil()を実行すると切り上げとなる。

### math.floor()とint()の違い
小数点以下の切り捨てにはint()を使うこともできる。<br>
正の値に対してはmath.floor()とint()は同じ結果を返すが、負の値に対してはmath.floor()とint()で異なる結果を返す。
```python
>>> import math
>>> print(math.floor(-2.5))
-3
>>> print(int(-2.5))
-2
```
これはmath.floor()が常に負の方向へ丸められるのに対し、int()は常に0の方向へ丸められるからである。