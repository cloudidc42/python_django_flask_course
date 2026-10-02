# Part 023 - Math and Random (คณิตศาสตร์และการสุ่ม)

## เป้าหมาย
- ใช้ `math` module สำหรับการคำนวณ
- ใช้ `statistics` module
- ใช้ `random` module: random, randint, choice, shuffle, sample
- แนะนำ `numpy` สำหรับ numerical computing
- ตัวอย่าง: simulations, statistics, Monte Carlo

---

## 1. math Module

```python
import math

# Constants
print(math.pi)    # 3.141592653589793
print(math.e)     # 2.718281828459045
print(math.tau)   # 6.283185307179586 (2*pi)
print(math.inf)   # inf
print(math.nan)   # nan
print(-math.inf)  # -inf

# Basic operations
print(math.sqrt(16))    # 4.0
print(math.pow(2, 10))  # 1024.0 (คืน float)
print(2 ** 10)          # 1024 (คืน int)

# Rounding
print(math.floor(3.7))  # 3 (ปัดลง)
print(math.ceil(3.2))   # 4 (ปัดขึ้น)
print(math.trunc(3.9))  # 3 (ตัดทศนิยม)
print(round(3.5))       # 4 (Python round)

# Absolute value
print(math.fabs(-5.5))  # 5.5 (คืน float เสมอ)
print(abs(-5.5))        # 5.5

# Log functions
print(math.log(100))         # 4.605... (natural log, base e)
print(math.log(100, 10))     # 2.0 (log base 10)
print(math.log10(1000))      # 3.0
print(math.log2(1024))       # 10.0
print(math.exp(1))           # e^1 = 2.718...

# Trigonometry (radians)
angle_deg = 45
angle_rad = math.radians(angle_deg)  # แปลง degrees -> radians
print(math.sin(angle_rad))  # 0.7071...
print(math.cos(angle_rad))  # 0.7071...
print(math.tan(angle_rad))  # 1.0

# Inverse trig
print(math.degrees(math.asin(0.5)))  # 30.0
print(math.degrees(math.acos(0.5)))  # 60.0
print(math.degrees(math.atan(1.0)))  # 45.0

# atan2 - คืน angle ใน 4 quadrants
print(math.degrees(math.atan2(1, 1)))   # 45.0
print(math.degrees(math.atan2(-1, 1)))  # -45.0

# Factorial, combinatorics
print(math.factorial(10))  # 3628800
print(math.comb(10, 3))    # 120 (10 choose 3)
print(math.perm(10, 3))    # 720 (10 permute 3)
print(math.gcd(12, 18))    # 6
print(math.lcm(4, 6))      # 12

# Hyperbolic functions
print(math.sinh(1))  # 1.1752...
print(math.cosh(1))  # 1.5430...
print(math.tanh(1))  # 0.7615...

# Special functions
print(math.isfinite(1.0))     # True
print(math.isinf(math.inf))   # True
print(math.isnan(math.nan))   # True
print(math.isclose(0.1 + 0.2, 0.3, rel_tol=1e-9))  # True

# fsum - accurate floating point sum
print(sum([0.1, 0.1, 0.1, 0.1, 0.1, 0.1, 0.1, 0.1, 0.1, 0.1]))  # 0.9999...
print(math.fsum([0.1] * 10))  # 1.0 (แม่นยำกว่า)

# prod - product
print(math.prod([1, 2, 3, 4, 5]))  # 120


# ตัวอย่าง: คำนวณทางคณิตศาสตร์
def distance_3d(p1: tuple, p2: tuple) -> float:
    """ระยะห่างระหว่างสองจุดใน 3D"""
    return math.sqrt(sum((a - b) ** 2 for a, b in zip(p1, p2)))

def circle_area(radius: float) -> float:
    return math.pi * radius ** 2

def sphere_volume(radius: float) -> float:
    return (4/3) * math.pi * radius ** 3

print(f"Distance: {distance_3d((0,0,0), (1,2,3)):.4f}")  # 3.7417
print(f"Circle area (r=5): {circle_area(5):.2f}")          # 78.54
print(f"Sphere volume (r=3): {sphere_volume(3):.2f}")      # 113.10
```

---

## 2. statistics Module

```python
import statistics
from typing import List

# ตัวอย่างข้อมูล
scores = [72, 85, 90, 68, 76, 92, 88, 79, 83, 91, 65, 78, 86, 74, 95]

# Measures of Central Tendency
print("=== Central Tendency ===")
print(f"Mean (ค่าเฉลี่ย):   {statistics.mean(scores):.2f}")
print(f"Median (มัธยฐาน):  {statistics.median(scores)}")
print(f"Mode (ฐานนิยม):    {statistics.mode(scores)}")

# จัดการ multi-mode
data_multi = [1, 2, 2, 3, 3, 4]
print(f"Multimode: {statistics.multimode(data_multi)}")  # [2, 3]

# Weighted mean
weights = [1, 2, 3, 2, 1, 3, 2, 1, 2, 3, 1, 2, 3, 2, 1]
weighted = statistics.fmean(scores)  # fast mean (float)
print(f"Fast mean: {weighted:.2f}")

# Measures of Spread
print("\n=== Spread ===")
print(f"Variance (population):  {statistics.pvariance(scores):.2f}")
print(f"Variance (sample):      {statistics.variance(scores):.2f}")
print(f"StdDev (population):    {statistics.pstdev(scores):.2f}")
print(f"StdDev (sample):        {statistics.stdev(scores):.2f}")

# Quantiles
quantiles = statistics.quantiles(scores, n=4)  # quartiles
print(f"\nQ1: {quantiles[0]}")   # 25th percentile
print(f"Q2: {quantiles[1]}")    # 50th percentile (median)
print(f"Q3: {quantiles[2]}")    # 75th percentile
iqr = quantiles[2] - quantiles[0]
print(f"IQR: {iqr}")

# Find outliers using IQR method
lower_fence = quantiles[0] - 1.5 * iqr
upper_fence = quantiles[2] + 1.5 * iqr
outliers = [x for x in scores if x < lower_fence or x > upper_fence]
print(f"Outliers: {outliers}")

# Correlation
x = [1, 2, 3, 4, 5]
y = [2, 4, 5, 4, 5]
corr = statistics.correlation(x, y)
print(f"\nCorrelation: {corr:.4f}")

# Linear regression
slope, intercept = statistics.linear_regression(x, y)
print(f"Linear regression: y = {slope:.4f}x + {intercept:.4f}")


# Custom statistics functions
def describe(data: List[float]) -> dict:
    """Describe statistics ของข้อมูล"""
    n = len(data)
    sorted_data = sorted(data)
    
    return {
        "count": n,
        "mean": statistics.mean(data),
        "median": statistics.median(data),
        "mode": statistics.multimode(data),
        "std": statistics.stdev(data) if n > 1 else 0,
        "variance": statistics.variance(data) if n > 1 else 0,
        "min": min(data),
        "max": max(data),
        "range": max(data) - min(data),
        "q1": statistics.quantiles(data, n=4)[0],
        "q3": statistics.quantiles(data, n=4)[2],
    }

print("\n=== Description ===")
for key, value in describe(scores).items():
    print(f"  {key}: {value}")
```

---

## 3. random Module

```python
import random

# Seed สำหรับ reproducibility
random.seed(42)

# === Basic Random Numbers ===
print("=== Basic ===")
print(random.random())         # [0.0, 1.0)
print(random.uniform(1, 10))   # float ระหว่าง 1-10
print(random.randint(1, 10))   # int ระหว่าง 1-10 (inclusive)
print(random.randrange(0, 100, 5))  # 0, 5, 10, ..., 95

# Gaussian distribution
print(random.gauss(mu=0, sigma=1))    # Normal distribution
print(random.normalvariate(0, 1))     # Same, but thread-safe

# === Sequences ===
print("\n=== Sequences ===")
items = ['apple', 'banana', 'cherry', 'date', 'elderberry']

# Choice - สุ่มหนึ่งรายการ
print(random.choice(items))

# Choices - สุ่มหลายรายการ (with replacement)
print(random.choices(items, k=3))

# Choices with weights
print(random.choices(items, weights=[5, 2, 2, 1, 1], k=5))

# Sample - สุ่มไม่ซ้ำ (without replacement)
print(random.sample(items, k=3))

# Shuffle - สุ่มลำดับ (in-place)
deck = list(range(1, 11))
random.shuffle(deck)
print(f"Shuffled deck: {deck}")

# สร้าง shuffled copy (ไม่แก้ไขต้นฉบับ)
original = list(range(1, 6))
shuffled = original.copy()
random.shuffle(shuffled)
print(f"Original: {original}, Shuffled: {shuffled}")

# === Cryptographically Secure Random ===
import secrets

# ใช้ secrets สำหรับ security-sensitive operations
token = secrets.token_hex(16)        # 16 bytes = 32 hex chars
url_token = secrets.token_urlsafe(32) # URL-safe base64

print(f"\nSecure token: {token}")
print(f"URL token: {url_token}")

# สุ่ม password
import string
alphabet = string.ascii_letters + string.digits + string.punctuation
password = ''.join(secrets.choice(alphabet) for _ in range(16))
print(f"Secure password: {password}")
```

---

## 4. Simulations และ Monte Carlo

### Monte Carlo Pi Estimation

```python
import random
import math

def estimate_pi(n_samples: int = 1_000_000) -> float:
    """
    ประมาณค่า Pi ด้วย Monte Carlo simulation
    
    หลักการ: สุ่มจุดใน square [-1,1] x [-1,1]
    ถ้าจุดอยู่ใน unit circle (r <= 1) นับเป็น "inside"
    Pi ≈ 4 * (จำนวนจุดใน circle) / (จำนวนจุดทั้งหมด)
    """
    inside = 0
    
    for _ in range(n_samples):
        x = random.uniform(-1, 1)
        y = random.uniform(-1, 1)
        
        if x**2 + y**2 <= 1:
            inside += 1
    
    return 4 * inside / n_samples


# ทดสอบ
for n in [1000, 10000, 100000, 1000000]:
    pi_est = estimate_pi(n)
    error = abs(pi_est - math.pi) / math.pi * 100
    print(f"n={n:>8,}: pi ≈ {pi_est:.6f}, error = {error:.2f}%")


# Monte Carlo สำหรับ financial simulation
def simulate_stock_price(
    initial_price: float,
    days: int,
    annual_return: float = 0.10,
    annual_volatility: float = 0.20,
    n_simulations: int = 1000
) -> dict:
    """
    Simulate stock price ด้วย Geometric Brownian Motion
    
    Args:
        initial_price: ราคาเริ่มต้น
        days: จำนวนวัน
        annual_return: ผลตอบแทนต่อปี (เช่น 0.10 = 10%)
        annual_volatility: ความผันผวนต่อปี
        n_simulations: จำนวนการ simulate
    """
    import statistics
    
    dt = 1/252  # 252 trading days per year
    daily_return = annual_return * dt
    daily_volatility = annual_volatility * math.sqrt(dt)
    
    final_prices = []
    
    for _ in range(n_simulations):
        price = initial_price
        for _ in range(days):
            # Geometric Brownian Motion
            random_return = random.gauss(0, 1)
            price *= math.exp(
                (daily_return - 0.5 * daily_volatility**2) + 
                daily_volatility * random_return
            )
        final_prices.append(price)
    
    final_prices.sort()
    
    return {
        "initial_price": initial_price,
        "days": days,
        "mean_final_price": statistics.mean(final_prices),
        "median_final_price": statistics.median(final_prices),
        "std_final_price": statistics.stdev(final_prices),
        "var_5pct": final_prices[int(n_simulations * 0.05)],  # 5% VaR
        "probability_profit": sum(1 for p in final_prices if p > initial_price) / n_simulations,
        "simulations": n_simulations
    }


result = simulate_stock_price(
    initial_price=1000,
    days=252,       # 1 ปี
    annual_return=0.10,
    annual_volatility=0.20,
    n_simulations=10000
)

print("\n=== Stock Price Simulation (1 Year) ===")
print(f"Initial Price:    {result['initial_price']:,.2f}")
print(f"Mean Final Price: {result['mean_final_price']:,.2f}")
print(f"5% VaR:           {result['var_5pct']:,.2f}")
print(f"P(Profit):        {result['probability_profit']:.1%}")
```

### Random Data Generation

```python
import random
import string
from datetime import datetime, timedelta, date
from typing import List, Dict

class DataGenerator:
    """สร้างข้อมูลสำหรับทดสอบ"""
    
    FIRST_NAMES = ["Alice", "Bob", "Charlie", "Diana", "Eve", 
                   "Frank", "Grace", "Henry", "Iris", "Jack",
                   "สมชาย", "สมหญิง", "มานี", "วิชัย", "สุดา"]
    
    LAST_NAMES = ["Smith", "Johnson", "Williams", "Brown", "Jones",
                  "ใจดี", "บุญมี", "แก้วมณี", "สุขสันต์", "ทองดี"]
    
    DEPARTMENTS = ["Engineering", "Marketing", "HR", "Finance", "Operations"]
    POSITIONS = ["Junior", "Mid", "Senior", "Lead", "Manager"]
    
    def __init__(self, seed: int = None):
        if seed is not None:
            random.seed(seed)
    
    def name(self) -> str:
        """สร้างชื่อสุ่ม"""
        return f"{random.choice(self.FIRST_NAMES)} {random.choice(self.LAST_NAMES)}"
    
    def email(self, name: str = None) -> str:
        """สร้าง email สุ่ม"""
        if not name:
            name = self.name()
        
        username = name.lower().replace(' ', '.')
        domain = random.choice(['gmail.com', 'yahoo.com', 'company.th', 'test.org'])
        return f"{username}@{domain}"
    
    def phone(self) -> str:
        """สร้างเบอร์โทรไทยสุ่ม"""
        prefix = random.choice(['081', '082', '083', '084', '085',
                               '086', '087', '088', '089', '091',
                               '092', '093', '094', '095', '096',
                               '097', '098', '099', '061', '062',
                               '063', '064', '065', '066'])
        number = ''.join([str(random.randint(0, 9)) for _ in range(7)])
        return f"{prefix}-{number[:3]}-{number[3:]}"
    
    def salary(self, position: str) -> float:
        """สร้างเงินเดือนตาม position"""
        ranges = {
            "Junior": (25000, 40000),
            "Mid": (40000, 65000),
            "Senior": (65000, 100000),
            "Lead": (90000, 150000),
            "Manager": (120000, 250000),
        }
        low, high = ranges.get(position, (30000, 50000))
        return round(random.uniform(low, high), 2)
    
    def date_in_range(self, start: date, end: date) -> date:
        """สร้างวันที่สุ่มในช่วง"""
        delta = (end - start).days
        return start + timedelta(days=random.randint(0, delta))
    
    def employee(self) -> Dict:
        """สร้างข้อมูลพนักงานสุ่ม"""
        name = self.name()
        position = random.choice(self.POSITIONS)
        
        return {
            "id": f"EMP{random.randint(1000, 9999)}",
            "name": name,
            "email": self.email(name),
            "phone": self.phone(),
            "department": random.choice(self.DEPARTMENTS),
            "position": position,
            "salary": self.salary(position),
            "hire_date": self.date_in_range(
                date(2020, 1, 1), 
                date.today()
            ).isoformat(),
            "is_active": random.choices([True, False], weights=[9, 1])[0]
        }
    
    def employees(self, n: int) -> List[Dict]:
        """สร้างพนักงานหลายคน"""
        return [self.employee() for _ in range(n)]


# ทดสอบ
gen = DataGenerator(seed=42)

employees = gen.employees(5)
print("Generated Employees:")
for emp in employees:
    print(f"  {emp['name']:25} | {emp['department']:12} | {emp['position']:7} | ฿{emp['salary']:,.0f}")
```

---

## 5. numpy Intro

```python
try:
    import numpy as np
    HAS_NUMPY = True
except ImportError:
    print("numpy not installed. Install with: pip install numpy")
    HAS_NUMPY = False

if HAS_NUMPY:
    # === Array Creation ===
    print("=== Array Creation ===")
    
    # จาก list
    a = np.array([1, 2, 3, 4, 5])
    print(f"1D array: {a}")
    print(f"dtype: {a.dtype}")
    
    # 2D array
    b = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
    print(f"\n2D array:\n{b}")
    print(f"shape: {b.shape}")
    print(f"ndim: {b.ndim}")
    
    # Special arrays
    zeros = np.zeros((3, 4))
    ones = np.ones((2, 3))
    identity = np.eye(3)
    empty = np.empty((2, 2))
    
    # Range arrays
    arr = np.arange(0, 10, 2)      # [0, 2, 4, 6, 8]
    linspace = np.linspace(0, 1, 5) # [0, 0.25, 0.5, 0.75, 1.0]
    
    print(f"\narange: {arr}")
    print(f"linspace: {linspace}")
    
    # Random arrays
    np.random.seed(42)
    rand_arr = np.random.rand(3, 3)      # uniform [0, 1)
    randn_arr = np.random.randn(3, 3)    # standard normal
    randint_arr = np.random.randint(0, 10, (3, 3))  # random integers
    
    # === Array Operations ===
    print("\n=== Operations ===")
    
    a = np.array([1, 2, 3, 4, 5])
    b = np.array([10, 20, 30, 40, 50])
    
    # Element-wise operations
    print(a + b)     # [11, 22, 33, 44, 55]
    print(a * b)     # [10, 40, 90, 160, 250]
    print(a ** 2)    # [1, 4, 9, 16, 25]
    print(np.sqrt(a))  # [1.0, 1.414, 1.732, 2.0, 2.236]
    
    # Broadcasting
    a = np.array([[1, 2, 3], [4, 5, 6]])
    b = np.array([10, 20, 30])
    print(a + b)  # แต่ละ row + b
    
    # === Aggregations ===
    print("\n=== Aggregations ===")
    data = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
    
    print(f"Sum: {data.sum()}")             # 45
    print(f"Mean: {data.mean():.2f}")        # 5.0
    print(f"Std: {data.std():.4f}")          # 2.5819...
    print(f"Min: {data.min()}")             # 1
    print(f"Max: {data.max()}")             # 9
    
    # Axis-wise
    print(f"Sum by row: {data.sum(axis=1)}")    # [6, 15, 24]
    print(f"Sum by col: {data.sum(axis=0)}")    # [12, 15, 18]
    print(f"Max by row: {data.max(axis=1)}")    # [3, 6, 9]
    
    # === Slicing and Indexing ===
    print("\n=== Slicing ===")
    a = np.arange(12).reshape(3, 4)
    print(a)
    print(a[0])          # first row
    print(a[:, 1])       # second column
    print(a[1:3, 1:3])   # sub-matrix
    
    # Boolean indexing
    mask = a > 5
    print(a[mask])       # values > 5
    
    # === Linear Algebra ===
    print("\n=== Linear Algebra ===")
    A = np.array([[1, 2], [3, 4]])
    B = np.array([[5, 6], [7, 8]])
    
    print("Matrix multiplication:")
    print(A @ B)          # matrix multiply
    # หรือ np.dot(A, B)
    
    print(f"\nDeterminant: {np.linalg.det(A):.2f}")
    print(f"Inverse:\n{np.linalg.inv(A)}")
    
    eigenvalues, eigenvectors = np.linalg.eig(A)
    print(f"\nEigenvalues: {eigenvalues}")
    
    # Solve linear system: Ax = b
    b = np.array([5, 11])
    x = np.linalg.solve(A, b)
    print(f"\nSolution to Ax=b: {x}")
    
    # === Statistics with numpy ===
    print("\n=== Statistics ===")
    data = np.random.normal(loc=100, scale=15, size=10000)
    
    print(f"Mean: {np.mean(data):.2f}")
    print(f"Std: {np.std(data):.2f}")
    print(f"Percentiles:")
    for p in [25, 50, 75, 90, 95, 99]:
        print(f"  P{p}: {np.percentile(data, p):.2f}")
    
    # Correlation matrix
    x = np.random.randn(100)
    y = x * 0.5 + np.random.randn(100) * 0.5
    z = np.random.randn(100)
    
    data_2d = np.column_stack([x, y, z])
    corr_matrix = np.corrcoef(data_2d.T)
    print(f"\nCorrelation matrix:\n{np.round(corr_matrix, 3)}")
```

---

## 6. Statistical Analysis Examples

```python
import statistics
import math
import random

def analyze_grades(grades: list) -> dict:
    """วิเคราะห์คะแนนนักเรียน"""
    n = len(grades)
    
    # Basic stats
    mean = statistics.mean(grades)
    median = statistics.median(grades)
    stdev = statistics.stdev(grades) if n > 1 else 0
    
    # Grade distribution
    grade_dist = {}
    for g in grades:
        if g >= 80:
            letter = "A"
        elif g >= 70:
            letter = "B"
        elif g >= 60:
            letter = "C"
        elif g >= 50:
            letter = "D"
        else:
            letter = "F"
        grade_dist[letter] = grade_dist.get(letter, 0) + 1
    
    # Pass/Fail
    pass_count = sum(1 for g in grades if g >= 50)
    
    # z-scores
    z_scores = [(g - mean) / stdev if stdev > 0 else 0 for g in grades]
    
    return {
        "n": n,
        "mean": round(mean, 2),
        "median": median,
        "stdev": round(stdev, 2),
        "min": min(grades),
        "max": max(grades),
        "pass_rate": round(pass_count / n * 100, 1),
        "grade_distribution": grade_dist,
        "highest_z_score": round(max(z_scores), 2),
        "lowest_z_score": round(min(z_scores), 2),
    }


# Histogram ด้วย text
def text_histogram(data: list, bins: int = 10, width: int = 40) -> str:
    """แสดง histogram แบบ text"""
    min_val = min(data)
    max_val = max(data)
    bin_size = (max_val - min_val) / bins
    
    # นับ frequency
    counts = [0] * bins
    for x in data:
        bin_idx = min(int((x - min_val) / bin_size), bins - 1)
        counts[bin_idx] += 1
    
    max_count = max(counts)
    
    lines = []
    for i, count in enumerate(counts):
        low = min_val + i * bin_size
        high = low + bin_size
        bar_len = int(count / max_count * width)
        bar = "█" * bar_len
        lines.append(f"{low:6.1f}-{high:6.1f} | {bar} {count}")
    
    return "\n".join(lines)


# สร้างคะแนนสุ่ม
random.seed(42)
# Normal distribution ของคะแนน
grades = [
    max(0, min(100, random.gauss(72, 12)))
    for _ in range(200)
]
grades_int = [round(g) for g in grades]

print("=== Grade Analysis ===")
analysis = analyze_grades(grades_int)
for key, value in analysis.items():
    print(f"  {key}: {value}")

print("\n=== Distribution ===")
print(text_histogram(grades_int))


# Simulation: Dice Statistics
def simulate_dice(n_dice: int = 2, n_rolls: int = 10000) -> dict:
    """Simulate การทอยลูกเต๋า"""
    results = []
    for _ in range(n_rolls):
        roll = sum(random.randint(1, 6) for _ in range(n_dice))
        results.append(roll)
    
    min_val = n_dice
    max_val = n_dice * 6
    
    freq = {}
    for r in results:
        freq[r] = freq.get(r, 0) + 1
    
    return {
        "dice": n_dice,
        "rolls": n_rolls,
        "mean": statistics.mean(results),
        "expected_mean": n_dice * 3.5,  # ค่าที่ควรจะได้
        "frequency": {k: v / n_rolls for k, v in sorted(freq.items())}
    }


dice_sim = simulate_dice(2, 100000)
print("\n=== 2 Dice Simulation ===")
print(f"Mean: {dice_sim['mean']:.2f} (Expected: {dice_sim['expected_mean']})")
print("Distribution:")
for value, freq in dice_sim["frequency"].items():
    bar = "█" * int(freq * 200)
    print(f"  {value:2d}: {bar} {freq:.1%}")
```

---

## Exercises

### Exercise 1: Statistics Calculator
สร้าง `StatCalculator` class ที่:
- รับ data แบบ streaming (ไม่โหลดทั้งหมด)
- คำนวณ mean, variance, std แบบ online (Welford's algorithm)
- รองรับ sliding window

### Exercise 2: Random Name Generator
สร้าง name generator สำหรับ RPG:
- สุ่มชื่อตาม race (Human, Elf, Dwarf)
- แต่ละ race มี pattern ของชื่อที่ต่างกัน
- ใช้ Markov chain เพื่อสร้างชื่อที่ฟังดูสมจริง

### Exercise 3: Monte Carlo Portfolio
สร้าง portfolio simulation:
- รับ assets หลายตัวกับ expected return และ volatility
- Simulate portfolio performance หลาย scenario
- คำนวณ VaR (Value at Risk) ที่ confidence level ต่างๆ

---

## สรุป

| Module/Function | คำอธิบาย | ตัวอย่าง |
|----------------|---------|---------|
| `math.sqrt()` | รากที่สอง | `math.sqrt(16)` → 4.0 |
| `math.pi` | ค่า pi | 3.14159... |
| `math.log()` | logarithm | `math.log(e)` → 1.0 |
| `statistics.mean()` | ค่าเฉลี่ย | `mean([1,2,3])` → 2 |
| `statistics.stdev()` | ส่วนเบี่ยงเบน | `stdev([1,2,3])` |
| `random.random()` | [0, 1) | `0.37...` |
| `random.choice()` | สุ่มหนึ่งตัว | `choice([1,2,3])` |
| `random.sample()` | สุ่มไม่ซ้ำ | `sample(lst, 3)` |
| `random.shuffle()` | สุ่มลำดับ | in-place |
| `secrets.token_hex()` | secure random | สำหรับ security |

---

## ต่อไป

[Part 024 - JSON and CSV](part-024.md) - json module, csv module, pandas intro
