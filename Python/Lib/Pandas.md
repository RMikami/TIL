# 数値データを指定した範囲で区分けする
## pandas.cut()
第一引数に元データとなる一次元配列(リスト、numpy.ndarray、pandas.Series)を指定する。<br>
第二引数に分割数を指定すると、元データの最大値と最小値の間を等間隔で分割する。<br>
precisionに数値を指定すると、区切る境界の小数点以下の桁数を設定できる。

```python
>>> import pandas as pd
>>> x = pd.Series([14, 36, 45, 53, 66, 74, 98])
>>> pd.cut(x, 5, precision = 1)
0    (13.9, 30.8]
1    (30.8, 47.6]
2    (30.8, 47.6]
3    (47.6, 64.4]
4    (64.4, 81.2]
5    (64.4, 81.2]
6    (81.2, 98.0]
dtype: category
Categories (5, interval[float64, right]): [(13.9, 30.8] < (30.8, 47.6] < (47.6, 64.4] <
                                           (64.4, 81.2] < (81.2, 98.0]]
```

# 各データの頻度(出現回数)を取得する
## pandas.Series.value_counts()
pandas.Seriesにおいてvalue_counts関数を使用することで各データの頻度を取得することができる。<br>
デフォルトでは出現回数が多いものから順にソートされる(降順)が、引数sort=Falseとするとソートされない。引数ascending=Trueとすると昇順にソートされる。

```python
>>> import pandas as pd
>>> x = pd.Series([14, 36, 45, 53, 66, 74, 98])
>>> x_cut = pd.cut(x, 5, precision = 1)
>>> x_cut.value_counts(sort = False)
(13.9, 30.8]    1
(30.8, 47.6]    2
(47.6, 64.4]    1
(64.4, 81.2]    2
(81.2, 98.0]    1
dtype: int64
```

# グラフを作成する
## 棒グラフ
垂直棒グラフはpandas.plot.bar()、水平棒グラフはpandas.plot.barh()で作成できる。