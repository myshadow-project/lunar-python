# lunar-python `LunarYear.fromYear` 缓存 Patch 说明

## 背景

在 `playground_da_yun_liu_nian_generation.py` 的大运流年流月三层嵌套循环中，每次调用
`LiuNian.getGanZhi()` / `LiuYue.getGanZhi()` 都会触发 `Solar.getLunar()`，
进而调用 `LunarYear.fromYear(year)`。

### 调用链（来自 cProfile 实测）

```
getLunar()                          ← 483次调用，累计 2.45s（占总耗时 97%）
  └── Lunar.fromSolar()
        └── LunarYear.fromYear()
              └── LunarYear.compute()   ← 967次，每次 ~2.5ms
                    └── ShouXingUtil.qiAccurate / calcShuo
                          └── ShouXingUtil.eLon()   ← 87,512次，累计 1.25s
                                └── math.cos()      ← 13,895,854次，累计 0.59s
```

### 根本原因

`LunarYear.fromYear` 现有实现只有**单元素缓存**（`__CACHE_YEAR`），
仅缓存最后一次查询的年份。三层循环中年份不断切换（大运年 → 流年 → 流月），
导致缓存命中率接近 0，每次都重新执行完整的天文历法计算。

```python
# 现有实现 - 单元素缓存，频繁失效
@staticmethod
def fromYear(lunar_year):
    LunarYear.__lock.acquire()
    if LunarYear.__CACHE_YEAR is None or LunarYear.__CACHE_YEAR.getYear() != lunar_year:
        y = LunarYear(lunar_year)
        LunarYear.__CACHE_YEAR = y   # 只保存最后一个
    else:
        y = LunarYear.__CACHE_YEAR
    LunarYear.__lock.release()
    return y
```

---

## Patch 内容

**文件：** `lunar_python/LunarYear.py`

### 修改 1：将单元素缓存替换为字典缓存

**位置：** 类变量声明区（约第 38-40 行）

```python
# 删除
__CACHE_YEAR = None

__lock = threading.Lock()
```

```python
# 替换为
__CACHE = 

__lock = threading.Lock()
```

---

### 修改 2：更新 `fromYear` 方法

**位置：** `fromYear` 静态方法（约第 55-63 行）

```python
# 删除
@staticmethod
def fromYear(lunar_year):
    LunarYear.__lock.acquire()
    if LunarYear.__CACHE_YEAR is None or LunarYear.__CACHE_YEAR.getYear() != lunar_year:
        y = LunarYear(lunar_year)
        LunarYear.__CACHE_YEAR = y
    else:
        y = LunarYear.__CACHE_YEAR
    LunarYear.__lock.release()
    return y
```

```python
# 替换为
@staticmethod
def fromYear(lunar_year):
    LunarYear.__lock.acquire()
    y = LunarYear.__CACHE.get(lunar_year)
    if y is None:
        y = LunarYear(lunar_year)
        LunarYear.__CACHE[lunar_year] = y
    LunarYear.__lock.release()
    return y
```

---

## 完整 diff

```diff
--- a/lunar_python/LunarYear.py
+++ b/lunar_python/LunarYear.py
@@ -38,7 +38,7 @@
-    __CACHE_YEAR = None
+    __CACHE = {}
 
     __lock = threading.Lock()
 
@@ -55,11 +55,9 @@
     @staticmethod
     def fromYear(lunar_year):
         LunarYear.__lock.acquire()
-        if LunarYear.__CACHE_YEAR is None or LunarYear.__CACHE_YEAR.getYear() != lunar_year:
+        y = LunarYear.__CACHE.get(lunar_year)
+        if y is None:
             y = LunarYear(lunar_year)
-            LunarYear.__CACHE_YEAR = y
-        else:
-            y = LunarYear.__CACHE_YEAR
+            LunarYear.__CACHE[lunar_year] = y
         LunarYear.__lock.release()
         return y
```

---

## 预期效果

| 指标 | Patch 前 | Patch 后（预估） |
|------|---------|----------------|
| `getLunar` 调用次数 | 483 | 483（不变） |
| `LunarYear.compute` 执行次数 | 967 | ~20（唯一年份数） |
| `math.cos` 调用次数 | 13,895,854 | ~285,000 |
| 大运流年流月生成耗时 | ~2.45s | ~0.05s |
| 总体耗时 | ~2.53s | ~0.3s |

加速比约 **8-10x**。

---

## 注意事项

1. **内存**：字典缓存会持续增长。八字场景中年份范围通常在 1900-2100 之间（约 200 个条目），
   每个 `LunarYear` 对象约 5-10KB，总计 < 2MB，可接受。
   如需限制内存，可改用 `functools.lru_cache(maxsize=256)`，但需将方法改为模块级函数。

2. **线程安全**：patch 保留了原有的 `threading.Lock()`，线程安全性不变。

3. **正确性**：`LunarYear` 对象是纯计算结果，输入相同（年份）则输出完全相同，
   缓存不会引入状态问题。

---

## 测试验证建议

Patch 后运行以下验证：

```python
from lunar_python import Solar

test_cases = [
    (1985, 6, 15, 14),
    (1970, 1, 1, 0),
    (2000, 12, 31, 23),
]

for year, month, day, hour in test_cases:
    solar = Solar.fromYmdHms(year, month, day, hour, 0, 0)
    lunar = solar.getLunar()
    print(f"{year}/{month}/{day} -> {lunar.getYearInGanZhi()}年 {lunar.getMonthInGanZhi()}月 {lunar.getDayInGanZhi()}日")
```

结果应与 patch 前完全一致。

---

## 库信息

- 库名：`lunar-python`
- PyPI：https://pypi.org/project/lunar-python/
- GitHub：https://github.com/6tail/lunar-python
- 修改文件：`lunar_python/LunarYear.py`
- 修改行数：2处，共删除 5 行，新增 4 行
