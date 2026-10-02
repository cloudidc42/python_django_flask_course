# Part 010: Sets
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

หลังจากเรียนจบ Part นี้ คุณจะสามารถ:
- สร้างและใช้งาน Set ได้
- ใช้ Set Operations ได้ (union, intersection, difference, symmetric_difference)
- เข้าใจ Frozen Sets และใช้งานได้
- เขียน Set Comprehension ได้
- เลือกใช้ Set อย่างเหมาะสมใน use cases ต่างๆ

---

## 1. Set พื้นฐาน

```python
# สร้าง Set - ไม่มีซ้ำ ไม่มีลำดับ
empty_set = set()                          # set ว่าง (ต้องใช้ set() ไม่ใช่ {})
numbers = {1, 2, 3, 4, 5}                 # set ตัวเลข
fruits = {"apple", "banana", "cherry"}    # set string
mixed = {1, "hello", 3.14, True}          # ผสม type (ต้องเป็น hashable)

# ⚠️ {} คือ dict ว่าง ไม่ใช่ set ว่าง
empty_dict = {}
empty_set = set()
print(type(empty_dict))  # <class 'dict'>
print(type(empty_set))   # <class 'set'>

# สร้างจาก iterable (ซ้ำถูกลบออก)
from_list = set([1, 2, 2, 3, 3, 3, 4])
print(from_list)    # {1, 2, 3, 4}

from_string = set("mississippi")
print(from_string)  # {'m', 'i', 's', 'p'}  ← unique chars

# Deduplication
duplicates = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
unique = set(duplicates)
unique_list = list(unique)
print(unique)       # {1, 2, 3, 4, 5, 6, 9}

# len()
print(len(fruits))  # 3

# Membership test - O(1) เร็วกว่า list O(n)
print("apple" in fruits)     # True
print("mango" in fruits)     # False
print("mango" not in fruits) # True

# Iteration (ไม่รับประกันลำดับ)
for fruit in fruits:
    print(fruit, end=" ")
print()

# Set ไม่รองรับ indexing
try:
    print(fruits[0])
except TypeError as e:
    print(f"Error: {e}")  # 'set' object is not subscriptable
```

---

## 2. Set Methods - เพิ่ม/ลบ

```python
fruits = {"apple", "banana", "cherry"}

# add() - เพิ่มสมาชิก 1 ตัว
fruits.add("date")
fruits.add("apple")  # ซ้ำ - ไม่เปลี่ยนอะไร
print(fruits)        # {'apple', 'banana', 'cherry', 'date'}

# update() - เพิ่มหลายตัว (รับ iterable)
fruits.update(["elderberry", "fig"])
fruits.update({"grape", "honeydew"})
print(fruits)

# remove() - ลบ (KeyError ถ้าไม่พบ)
fruits.remove("fig")
try:
    fruits.remove("mango")
except KeyError as e:
    print(f"KeyError: {e}")

# discard() - ลบ (ไม่ error ถ้าไม่พบ)
fruits.discard("mango")     # ไม่ error
fruits.discard("grape")     # ลบ grape
print(fruits)

# pop() - ลบและคืนค่าแบบ random (ไม่รับประกันว่าตัวไหน)
removed = fruits.pop()
print(f"ลบ (random): {removed}")

# clear() - ลบทั้งหมด
temp = {1, 2, 3}
temp.clear()
print(temp)  # set()

# copy() - shallow copy
original = {"a", "b", "c"}
copied = original.copy()
copied.add("d")
print(original)  # {'a', 'b', 'c'}
print(copied)    # {'a', 'b', 'c', 'd'}
```

---

## 3. Set Operations

### 3.1 Union (รวม)

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# | operator
union = a | b
print(union)  # {1, 2, 3, 4, 5, 6, 7, 8}

# union() method
union2 = a.union(b)
print(union2)  # เหมือนกัน

# union หลาย set
c = {8, 9, 10}
print(a | b | c)       # operator
print(a.union(b, c))   # method (รับ multiple args)

# |= in-place
a_copy = a.copy()
a_copy |= b
print(a_copy)  # {1, 2, 3, 4, 5, 6, 7, 8}
print(a)       # {1, 2, 3, 4, 5}  ← ไม่เปลี่ยน

# ตัวอย่างจริง: รวม tags
post1_tags = {"python", "programming", "tutorial"}
post2_tags = {"python", "django", "web", "tutorial"}
all_tags = post1_tags | post2_tags
print(f"Tags ทั้งหมด: {all_tags}")
```

### 3.2 Intersection (ร่วม)

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# & operator
intersection = a & b
print(intersection)  # {4, 5}

# intersection() method
inter2 = a.intersection(b)
print(inter2)  # {4, 5}

# intersection หลาย set
c = {5, 6, 7}
print(a & b & c)              # {5}
print(a.intersection(b, c))   # {5}

# &= in-place
a_copy = a.copy()
a_copy &= b
print(a_copy)  # {4, 5}

# intersection_update() - in-place
a_copy = a.copy()
a_copy.intersection_update(b)
print(a_copy)  # {4, 5}

# ตัวอย่างจริง: ผู้ที่สนใจทั้งสองสิ่ง
python_fans = {"Alice", "Bob", "Charlie", "Diana", "Evan"}
django_fans = {"Charlie", "Diana", "Frank", "Grace"}
both = python_fans & django_fans
print(f"สนใจทั้ง Python และ Django: {both}")
# {'Charlie', 'Diana'}

# ตรวจสอบว่า set มี element ร่วมกันหรือไม่
set1 = {1, 2, 3}
set2 = {4, 5, 6}
print(set1.isdisjoint(set2))  # True (ไม่มี element ร่วม)
print(set1.isdisjoint({3, 4}))  # False (มี 3 ร่วม)
```

### 3.3 Difference (ผลต่าง)

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# - operator (a - b = ใน a แต่ไม่ใน b)
diff = a - b
print(diff)  # {1, 2, 3}

# difference() method
diff2 = a.difference(b)
print(diff2)  # {1, 2, 3}

# b - a
print(b - a)  # {6, 7, 8}  ← อีกทาง

# difference หลาย set
c = {2, 3}
print(a - b - c)          # {1}
print(a.difference(b, c)) # {1}

# -= in-place
a_copy = a.copy()
a_copy -= b
print(a_copy)  # {1, 2, 3}

# ตัวอย่างจริง: หา items ที่ delete ไป
old_items = {"item1", "item2", "item3", "item4"}
new_items = {"item2", "item3", "item5"}

deleted = old_items - new_items
added = new_items - old_items
kept = old_items & new_items

print(f"ลบออก: {deleted}")    # {'item1', 'item4'}
print(f"เพิ่มใหม่: {added}")  # {'item5'}
print(f"คงอยู่: {kept}")      # {'item2', 'item3'}
```

### 3.4 Symmetric Difference (ผลต่างสมมาตร)

```python
a = {1, 2, 3, 4, 5}
b = {4, 5, 6, 7, 8}

# ^ operator (อยู่ใน a หรือ b แต่ไม่ใน a&b)
sym_diff = a ^ b
print(sym_diff)  # {1, 2, 3, 6, 7, 8}

# symmetric_difference() method
sym_diff2 = a.symmetric_difference(b)
print(sym_diff2)

# ^= in-place
a_copy = a.copy()
a_copy ^= b
print(a_copy)  # {1, 2, 3, 6, 7, 8}

# ตัวอย่างจริง: หาความแตกต่างระหว่าง 2 versions
v1_features = {"login", "signup", "profile", "feed", "search"}
v2_features = {"login", "signup", "profile", "chat", "notifications"}

changes = v1_features ^ v2_features
removed = v1_features - v2_features
added = v2_features - v1_features

print(f"เปลี่ยนแปลง: {changes}")   # {'feed', 'search', 'chat', 'notifications'}
print(f"ลบออก: {removed}")          # {'feed', 'search'}
print(f"เพิ่มใหม่: {added}")        # {'chat', 'notifications'}
```

### 3.5 Subset และ Superset

```python
a = {1, 2, 3}
b = {1, 2, 3, 4, 5}
c = {4, 5, 6}

# issubset() - a เป็น subset ของ b หรือไม่
print(a.issubset(b))    # True  (ทุก element ของ a อยู่ใน b)
print(a <= b)           # True  (operator)
print(a < b)            # True  (proper subset - a != b)
print(b < b)            # False (ต้องเล็กกว่า - ไม่เท่ากัน)

print(c.issubset(b))    # False (4,5,6 ไม่ใช่ subset ของ b ที่มี 4,5 แต่ไม่มี 6)

# issuperset() - b เป็น superset ของ a หรือไม่
print(b.issuperset(a))  # True  (b มีทุก element ของ a)
print(b >= a)           # True  (operator)
print(b > a)            # True  (proper superset)

# ตัวอย่างจริง: ตรวจสอบ permissions
ADMIN_PERMISSIONS = {"read", "write", "delete", "create", "admin"}
EDITOR_PERMISSIONS = {"read", "write", "create"}
VIEWER_PERMISSIONS = {"read"}

user_permissions = {"read", "write"}

print(user_permissions.issubset(EDITOR_PERMISSIONS))  # True (editor permissions เพียงพอ)
print(user_permissions.issubset(VIEWER_PERMISSIONS))  # False (มากกว่า viewer)

def has_permission(user_perms, required_perms):
    return required_perms.issubset(user_perms)

print(has_permission(user_permissions, {"read"}))         # True
print(has_permission(user_permissions, {"read", "write"}))  # True
print(has_permission(user_permissions, {"delete"}))        # False
```

---

## 4. Frozen Sets

```python
# frozenset - immutable set
fs = frozenset([1, 2, 3, 4, 5])
print(fs)           # frozenset({1, 2, 3, 4, 5})
print(type(fs))     # <class 'frozenset'>

# frozenset รองรับ set operations แต่ไม่มี in-place methods
a = frozenset([1, 2, 3])
b = frozenset([3, 4, 5])

print(a | b)   # frozenset({1, 2, 3, 4, 5})
print(a & b)   # frozenset({3})
print(a - b)   # frozenset({1, 2})
print(a ^ b)   # frozenset({1, 2, 4, 5})

# ไม่มี add, remove, update
try:
    fs.add(6)
except AttributeError as e:
    print(f"Error: {e}")

# frozenset เป็น hashable (ใช้เป็น dict key ได้)
graph = {
    frozenset({"A", "B"}): 5.0,   # edge A-B น้ำหนัก 5
    frozenset({"B", "C"}): 3.0,   # edge B-C น้ำหนัก 3
    frozenset({"A", "C"}): 8.0,   # edge A-C น้ำหนัก 8
}

for edge, weight in graph.items():
    nodes = list(edge)
    print(f"{nodes[0]} - {nodes[1]}: {weight}")

# frozenset ใน set
set_of_sets = {frozenset([1, 2]), frozenset([3, 4]), frozenset([1, 2])}
print(set_of_sets)  # {frozenset({1, 2}), frozenset({3, 4})}  ← ซ้ำถูกลบ

# ตัวอย่างจริง: Permission groups
PERMISSIONS = {
    "admin": frozenset(["read", "write", "delete", "create", "admin"]),
    "editor": frozenset(["read", "write", "create"]),
    "viewer": frozenset(["read"]),
}

def get_role(permissions: set) -> str:
    """หา role จาก permissions"""
    perm_fs = frozenset(permissions)
    for role, role_perms in PERMISSIONS.items():
        if perm_fs == role_perms:
            return role
    return "custom"

print(get_role({"read"}))                              # viewer
print(get_role({"read", "write", "create"}))           # editor
print(get_role({"read", "write"}))                     # custom
```

---

## 5. Set Comprehension

```python
# Set comprehension - คล้าย list comprehension แต่ใช้ {}
squares = {x**2 for x in range(1, 11)}
print(squares)  # {1, 4, 9, 16, 25, 36, 49, 64, 81, 100}

# กับเงื่อนไข
even_squares = {x**2 for x in range(1, 11) if x % 2 == 0}
print(even_squares)  # {4, 16, 36, 64, 100}

# จาก string
words = ["hello", "WORLD", "Python", "PYTHON", "hello"]
unique_lower = {w.lower() for w in words}
print(unique_lower)  # {'hello', 'world', 'python'}

# ตัวอักษรใน string
vowels = {c for c in "Hello World" if c.lower() in "aeiou"}
print(vowels)  # {'e', 'o'}

# จาก list ของ dict
students = [
    {"name": "Alice", "dept": "CS"},
    {"name": "Bob", "dept": "Math"},
    {"name": "Charlie", "dept": "CS"},
    {"name": "Diana", "dept": "Physics"},
]

departments = {s["dept"] for s in students}
print(departments)  # {'CS', 'Math', 'Physics'}

# Nested comprehension
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat_set = {x for row in matrix for x in row if x % 2 == 0}
print(flat_set)  # {2, 4, 6, 8}
```

---

## 6. Use Cases จริง

### 6.1 Deduplication

```python
# ลบข้อมูลซ้ำ
emails = [
    "alice@example.com",
    "bob@example.com",
    "alice@example.com",  # ซ้ำ
    "charlie@example.com",
    "bob@example.com",    # ซ้ำ
]

unique_emails = list(set(emails))
print(f"จาก {len(emails)} → {len(unique_emails)} emails")

# ลบซ้ำแต่รักษาลำดับ (Python 3.7+)
def deduplicate_ordered(items):
    """ลบซ้ำโดยรักษาลำดับ"""
    seen = set()
    result = []
    for item in items:
        if item not in seen:
            seen.add(item)
            result.append(item)
    return result

items = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]
print(deduplicate_ordered(items))  # [3, 1, 4, 5, 9, 2, 6]

# ลบซ้ำใน list ของ dict
records = [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"},
    {"id": 1, "name": "Alice"},  # ซ้ำ
    {"id": 3, "name": "Charlie"},
]

seen_ids = set()
unique_records = []
for r in records:
    if r["id"] not in seen_ids:
        seen_ids.add(r["id"])
        unique_records.append(r)

print(unique_records)
```

### 6.2 Membership Testing

```python
# Set membership O(1) เร็วกว่า list O(n)
import time

# สร้างข้อมูลทดสอบ
big_list = list(range(1000000))
big_set = set(range(1000000))
target = 999999

# ทดสอบ list
start = time.time()
for _ in range(1000):
    _ = target in big_list
list_time = time.time() - start

# ทดสอบ set
start = time.time()
for _ in range(1000):
    _ = target in big_set
set_time = time.time() - start

print(f"List: {list_time:.4f}s")
print(f"Set:  {set_time:.4f}s")
print(f"Set เร็วกว่า {list_time/set_time:.0f}x")

# ตัวอย่าง: Spam filter
SPAM_WORDS = frozenset([
    "free", "money", "prize", "winner", "click", "buy", "offer",
    "discount", "limited", "urgent", "guarantee", "cash", "earn",
])

def is_spam(message: str) -> bool:
    words = set(message.lower().split())
    spam_matches = words & SPAM_WORDS
    if len(spam_matches) >= 2:
        return True, spam_matches
    return False, spam_matches

messages = [
    "Hello, how are you?",
    "FREE money! Click here to earn CASH now!",
    "Meeting tomorrow at 3pm",
    "WINNER! Claim your FREE prize with DISCOUNT offer!",
]

for msg in messages:
    spam, words = is_spam(msg)
    status = "SPAM" if spam else "OK"
    print(f"[{status}] {msg[:50]}")
    if spam:
        print(f"       Spam words: {words}")
```

### 6.3 Graph Operations

```python
# Graph ด้วย adjacency set
class Graph:
    def __init__(self):
        self.vertices = set()
        self.edges = {}  # vertex -> set of neighbors
    
    def add_vertex(self, v):
        self.vertices.add(v)
        self.edges.setdefault(v, set())
    
    def add_edge(self, u, v, directed=False):
        self.add_vertex(u)
        self.add_vertex(v)
        self.edges[u].add(v)
        if not directed:
            self.edges[v].add(u)
    
    def neighbors(self, v):
        return self.edges.get(v, set())
    
    def is_connected(self, u, v):
        return v in self.edges.get(u, set())
    
    def bfs(self, start):
        """Breadth-First Search"""
        visited = set()
        queue = [start]
        order = []
        
        while queue:
            vertex = queue.pop(0)
            if vertex not in visited:
                visited.add(vertex)
                order.append(vertex)
                queue.extend(self.edges[vertex] - visited)
        
        return order
    
    def common_neighbors(self, u, v):
        """เพื่อนร่วมกัน"""
        return self.edges.get(u, set()) & self.edges.get(v, set())

# สร้าง social network
g = Graph()
connections = [
    ("Alice", "Bob"), ("Alice", "Charlie"), ("Alice", "Diana"),
    ("Bob", "Charlie"), ("Bob", "Evan"),
    ("Charlie", "Frank"), ("Diana", "Evan"),
    ("Evan", "Frank"),
]

for u, v in connections:
    g.add_edge(u, v)

print("=== Social Network ===")
for person in sorted(g.vertices):
    friends = sorted(g.neighbors(person))
    print(f"  {person}: {friends}")

print("\nเพื่อนร่วมของ Alice และ Bob:")
print(f"  {sorted(g.common_neighbors('Alice', 'Bob'))}")

print("\nBFS จาก Alice:")
print(f"  {g.bfs('Alice')}")
```

---

## 7. ตัวอย่างโปรแกรมจริง: Tag System

```python
"""
ระบบ Tag สำหรับ Content Management
"""
from collections import defaultdict
from typing import Set, Dict, List, Optional

class TagSystem:
    """ระบบจัดการ tags สำหรับ content"""
    
    def __init__(self):
        self.content_tags: Dict[str, Set[str]] = {}    # content_id -> tags
        self.tag_contents: Dict[str, Set[str]] = defaultdict(set)  # tag -> content_ids
    
    def tag_content(self, content_id: str, tags: Set[str]) -> None:
        """เพิ่ม tags ให้กับ content"""
        if content_id in self.content_tags:
            # ลบ mapping เก่า
            for old_tag in self.content_tags[content_id]:
                self.tag_contents[old_tag].discard(content_id)
        
        self.content_tags[content_id] = set(tags)
        for tag in tags:
            self.tag_contents[tag].add(content_id)
    
    def add_tags(self, content_id: str, new_tags: Set[str]) -> None:
        """เพิ่ม tags ให้กับ content ที่มีอยู่แล้ว"""
        current = self.content_tags.get(content_id, set())
        added = new_tags - current
        for tag in added:
            self.tag_contents[tag].add(content_id)
        self.content_tags[content_id] = current | new_tags
    
    def remove_tags(self, content_id: str, tags_to_remove: Set[str]) -> None:
        """ลบ tags ออกจาก content"""
        if content_id not in self.content_tags:
            return
        removed = self.content_tags[content_id] & tags_to_remove
        for tag in removed:
            self.tag_contents[tag].discard(content_id)
        self.content_tags[content_id] -= tags_to_remove
    
    def get_tags(self, content_id: str) -> Set[str]:
        """คืน tags ของ content"""
        return self.content_tags.get(content_id, set()).copy()
    
    def find_by_tags(self, tags: Set[str], mode: str = "any") -> Set[str]:
        """ค้นหา content ตาม tags
        mode='any': content ที่มี tag อย่างน้อย 1 ตัว
        mode='all': content ที่มีทุก tag
        """
        if not tags:
            return set()
        
        content_sets = [self.tag_contents[tag] for tag in tags]
        
        if mode == "all":
            result = content_sets[0].copy()
            for s in content_sets[1:]:
                result &= s
        else:  # any
            result = set()
            for s in content_sets:
                result |= s
        
        return result
    
    def similar_content(self, content_id: str, n: int = 5) -> List[tuple]:
        """หา content ที่มี tags คล้ายกัน"""
        my_tags = self.content_tags.get(content_id, set())
        if not my_tags:
            return []
        
        scores = {}
        for tag in my_tags:
            for other_id in self.tag_contents[tag]:
                if other_id != content_id:
                    other_tags = self.content_tags[other_id]
                    # Jaccard similarity
                    common = len(my_tags & other_tags)
                    total = len(my_tags | other_tags)
                    scores[other_id] = common / total if total > 0 else 0
        
        return sorted(scores.items(), key=lambda x: -x[1])[:n]
    
    def popular_tags(self, n: int = 10) -> List[tuple]:
        """คืน tags ที่ใช้บ่อยที่สุด"""
        tag_counts = {tag: len(contents) for tag, contents in self.tag_contents.items()}
        return sorted(tag_counts.items(), key=lambda x: -x[1])[:n]
    
    def tag_cloud(self) -> Dict[str, int]:
        """คืน dict ของ tag -> count"""
        return {tag: len(contents) for tag, contents in self.tag_contents.items()}

# ทดสอบ
ts = TagSystem()

# เพิ่ม content พร้อม tags
ts.tag_content("post_001", {"python", "programming", "tutorial", "beginner"})
ts.tag_content("post_002", {"python", "django", "web", "backend"})
ts.tag_content("post_003", {"javascript", "react", "web", "frontend"})
ts.tag_content("post_004", {"python", "data-science", "pandas", "tutorial"})
ts.tag_content("post_005", {"machine-learning", "python", "sklearn", "tutorial"})
ts.tag_content("post_006", {"django", "rest-api", "backend", "python"})

print("=== ค้นหา Posts ===")
python_posts = ts.find_by_tags({"python"})
print(f"Python posts: {sorted(python_posts)}")

tutorial_posts = ts.find_by_tags({"python", "tutorial"}, mode="all")
print(f"Python + Tutorial: {sorted(tutorial_posts)}")

web_or_backend = ts.find_by_tags({"web", "backend"}, mode="any")
print(f"Web or Backend: {sorted(web_or_backend)}")

print("\n=== Similar Content ===")
for similar_id, score in ts.similar_content("post_001"):
    tags = ts.get_tags(similar_id)
    print(f"  {similar_id} (similarity: {score:.2f}): {tags}")

print("\n=== Popular Tags ===")
for tag, count in ts.popular_tags(6):
    bar = "█" * count
    print(f"  {tag:20}: {count} {bar}")
```

---

## 8. Exercises

### Exercise 1: Enrollment System

```python
"""
ระบบลงทะเบียนวิชาเรียน:
1. นักเรียนลงทะเบียนวิชาได้
2. หาวิชาที่นักเรียนทุกคนลงทะเบียน (intersection)
3. หาวิชาทั้งหมดที่มีคนลงทะเบียน (union)
4. แนะนำวิชาที่ควรลง (เพื่อนลงแต่ตัวเองยังไม่ลง)
"""

students_courses = {
    "Alice": {"Math", "Physics", "Chemistry", "Python", "English"},
    "Bob": {"Math", "Biology", "Python", "English", "History"},
    "Charlie": {"Physics", "Chemistry", "Python", "Art", "Music"},
    "Diana": {"Math", "Physics", "Python", "English", "Drama"},
    "Evan": {"Biology", "Chemistry", "History", "Art", "Python"},
}

# วิชาที่ทุกคนลงทะเบียน
all_sets = list(students_courses.values())
everyone_courses = all_sets[0].copy()
for s in all_sets[1:]:
    everyone_courses &= s
print(f"วิชาที่ทุกคนลง: {everyone_courses}")

# วิชาทั้งหมด
all_courses = set()
for courses in students_courses.values():
    all_courses |= courses
print(f"วิชาทั้งหมด ({len(all_courses)}): {sorted(all_courses)}")

# แนะนำวิชา (เพื่อนส่วนใหญ่ลง แต่ตัวเองยังไม่ลง)
def recommend_courses(student, students_data, min_friends=2):
    my_courses = students_data[student]
    others = {name: courses for name, courses in students_data.items() if name != student}
    
    course_count = {}
    for courses in others.values():
        for course in courses - my_courses:  # วิชาที่ฉันยังไม่ลง
            course_count[course] = course_count.get(course, 0) + 1
    
    return {c for c, cnt in course_count.items() if cnt >= min_friends}

for student in sorted(students_courses.keys()):
    rec = recommend_courses(student, students_courses)
    if rec:
        print(f"  แนะนำให้ {student}: {sorted(rec)}")
```

### Exercise 2: Spell Checker

```python
"""
Simple Spell Checker โดยใช้ Set:
1. โหลด dictionary ของคำที่ถูกต้อง
2. ตรวจสอบคำสะกด
3. แนะนำคำที่ถูกต้องใกล้เคียง
"""

# Dictionary ขนาดเล็กสำหรับทดสอบ
DICTIONARY = frozenset([
    "hello", "world", "python", "programming", "language", "computer",
    "science", "algorithm", "data", "structure", "function", "class",
    "object", "variable", "integer", "string", "boolean", "list",
    "dictionary", "tuple", "set", "module", "import", "return",
    "define", "print", "input", "output", "error", "exception",
])

def edits1(word):
    """หาคำที่ต่างกัน 1 character (insertion/deletion/substitution/transposition)"""
    letters = "abcdefghijklmnopqrstuvwxyz"
    splits = [(word[:i], word[i:]) for i in range(len(word) + 1)]
    
    deletes     = {L + R[1:] for L, R in splits if R}
    transposes  = {L + R[1] + R[0] + R[2:] for L, R in splits if len(R) > 1}
    replaces    = {L + c + R[1:] for L, R in splits if R for c in letters}
    inserts     = {L + c + R for L, R in splits for c in letters}
    
    return deletes | transposes | replaces | inserts

def check_word(word, dictionary=DICTIONARY):
    """ตรวจสอบคำสะกด"""
    word = word.lower()
    if word in dictionary:
        return True, []
    
    # หาคำแนะนำ
    candidates = edits1(word) & dictionary
    if not candidates:
        # ลอง edits2 (ช้ากว่า)
        edits2 = {e2 for e1 in edits1(word) for e2 in edits1(e1)}
        candidates = edits2 & dictionary
    
    return False, sorted(candidates)[:5]

# ทดสอบ
test_words = ["python", "pyhton", "programing", "langage", "varaible", "tpule", "dictonary"]

print("=== Spell Checker ===")
for word in test_words:
    correct, suggestions = check_word(word)
    if correct:
        print(f"  ✓ {word}")
    else:
        sugg = ", ".join(suggestions) if suggestions else "ไม่พบคำแนะนำ"
        print(f"  ✗ {word} → แนะนำ: {sugg}")

# ตรวจประโยค
def check_sentence(text, dictionary=DICTIONARY):
    words = text.lower().split()
    errors = {}
    for word in words:
        # ลบ punctuation
        clean = word.strip(".,!?;:")
        if clean and clean not in dictionary:
            _, suggestions = check_word(clean)
            errors[clean] = suggestions
    return errors

sentence = "I love progrmming in phyton language"
errors = check_sentence(sentence)
print(f"\nประโยค: '{sentence}'")
print("ข้อผิดพลาด:")
for word, suggestions in errors.items():
    print(f"  '{word}' → {suggestions}")
```

### Exercise 3: Access Control

```python
"""
ระบบ Access Control ด้วย Set:
1. กำหนด permissions ให้ roles
2. users มีหลาย roles
3. ตรวจสอบว่า user มี permission ที่ต้องการหรือไม่
4. หา minimum role ที่ต้องการ
"""

class AccessControl:
    # Permissions
    PERMISSIONS = frozenset([
        "read", "write", "delete", "create",
        "manage_users", "view_reports", "export_data",
        "system_config", "backup", "audit_log",
    ])
    
    # Role definitions
    ROLES = {
        "viewer": frozenset(["read"]),
        "editor": frozenset(["read", "write", "create"]),
        "manager": frozenset(["read", "write", "create", "delete", "view_reports"]),
        "admin": frozenset(["read", "write", "create", "delete", "manage_users",
                           "view_reports", "export_data", "backup", "audit_log"]),
        "superadmin": frozenset(PERMISSIONS),  # ทุก permission
    }
    
    def __init__(self):
        self.users = {}  # user_id -> set of roles
    
    def assign_role(self, user_id: str, role: str) -> None:
        if role not in self.ROLES:
            raise ValueError(f"ไม่มี role '{role}'")
        self.users.setdefault(user_id, set()).add(role)
    
    def revoke_role(self, user_id: str, role: str) -> None:
        if user_id in self.users:
            self.users[user_id].discard(role)
    
    def get_permissions(self, user_id: str) -> frozenset:
        """คืน permissions ทั้งหมดของ user (union ของทุก role)"""
        roles = self.users.get(user_id, set())
        all_perms = set()
        for role in roles:
            all_perms |= self.ROLES[role]
        return frozenset(all_perms)
    
    def has_permission(self, user_id: str, permission: str) -> bool:
        return permission in self.get_permissions(user_id)
    
    def has_all_permissions(self, user_id: str, permissions: Set[str]) -> bool:
        return permissions.issubset(self.get_permissions(user_id))
    
    def check_access(self, user_id: str, required_perms: Set[str]) -> dict:
        user_perms = self.get_permissions(user_id)
        granted = required_perms & user_perms
        denied = required_perms - user_perms
        return {
            "allowed": len(denied) == 0,
            "granted": granted,
            "denied": denied,
        }

# ทดสอบ
ac = AccessControl()
ac.assign_role("alice", "admin")
ac.assign_role("bob", "editor")
ac.assign_role("charlie", "viewer")
ac.assign_role("diana", "editor")
ac.assign_role("diana", "manager")  # หลาย role

print("=== Access Check ===")
users = ["alice", "bob", "charlie", "diana"]
for user in users:
    perms = ac.get_permissions(user)
    roles = ac.users.get(user, set())
    print(f"  {user} ({','.join(sorted(roles))}): {len(perms)} permissions")

print("\n=== Permission Check: delete ===")
for user in users:
    can_delete = ac.has_permission(user, "delete")
    status = "✓" if can_delete else "✗"
    print(f"  {status} {user}")

print("\n=== Detailed Access ===")
required = {"write", "delete", "view_reports"}
for user in users:
    result = ac.check_access(user, required)
    status = "GRANTED" if result["allowed"] else "DENIED"
    print(f"  [{status}] {user}")
    if not result["allowed"]:
        print(f"    ขาด: {result['denied']}")
```

---

## 9. สรุป Part 010

### สิ่งที่เรียนรู้:

✅ **Set** - collection ที่ไม่มีซ้ำ ไม่มีลำดับ  
✅ **add() / update()** - เพิ่มสมาชิก  
✅ **remove() / discard()** - ลบสมาชิก  
✅ **union (|)** - รวม 2 set  
✅ **intersection (&)** - ร่วม 2 set  
✅ **difference (-)** - ผลต่าง  
✅ **symmetric_difference (^)** - ผลต่างสมมาตร  
✅ **issubset (<=)** - ตรวจ subset  
✅ **issuperset (>=)** - ตรวจ superset  
✅ **isdisjoint()** - ไม่มี element ร่วม  
✅ **frozenset** - immutable set (hashable)  
✅ **Set Comprehension** - `{x for x in ...}`  
✅ **Deduplication** - ลบข้อมูลซ้ำ  
✅ **Membership Testing** - O(1) เร็วกว่า list  

### Quick Reference:

```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

# Operations
a | b    # union: {1,2,3,4,5,6}
a & b    # intersection: {3,4}
a - b    # difference: {1,2}
b - a    # difference: {5,6}
a ^ b    # symmetric diff: {1,2,5,6}

# Subset/Superset
a <= b   # issubset
a >= b   # issuperset
a < b    # proper subset
a.isdisjoint(b)  # no common elements

# Methods
a.add(5)
a.update({5,6,7})
a.remove(1)      # KeyError if missing
a.discard(1)     # no error if missing
a.pop()          # random element
a.clear()
a.copy()

# frozenset
fs = frozenset([1,2,3])  # hashable, immutable

# Comprehension
{x**2 for x in range(5)}
```

---

## ➡️ ถัดไป: Part 011 - Strings ขั้นสูง

ใน Part ถัดไป เราจะเรียนรู้:
- String formatting ขั้นสูง (f-strings, format spec)
- String methods ครบทุก method
- Regular Expressions พื้นฐาน
- String encoding
- Bytes และ bytearray

---

*Part 010/100+ | Python Course - Beginner to World-Class*
