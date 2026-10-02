# Part 039: Web Scraping
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ requests library ดึงข้อมูลจากเว็บได้
- Parse HTML ด้วย BeautifulSoup4
- ใช้ CSS Selectors และ XPath
- จัดการ Pagination และ scraping tables
- ใช้ Selenium สำหรับ Dynamic Content
- เคารพ robots.txt และ Rate Limiting
- หลีกเลี่ยงการถูก Block

---

## 1. Setup และ Installation

```bash
# ติดตั้ง libraries ที่จำเป็น
pip install requests beautifulsoup4 lxml html5lib
pip install selenium webdriver-manager
pip install fake-useragent
pip install scrapy  # Framework สำหรับ scraping ขนาดใหญ่

# สำหรับ async scraping
pip install aiohttp
```

---

## 2. requests Library

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import time

# === Basic GET Request ===
def basic_scraping():
    url = "https://httpbin.org/get"
    
    # Headers ที่ดูเหมือน Browser จริง
    headers = {
        "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
        "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8",
        "Accept-Language": "th-TH,th;q=0.9,en;q=0.8",
        "Accept-Encoding": "gzip, deflate, br",
        "Connection": "keep-alive",
    }
    
    try:
        response = requests.get(url, headers=headers, timeout=10)
        response.raise_for_status()  # Raise exception สำหรับ 4xx, 5xx
        
        print(f"Status: {response.status_code}")
        print(f"Content-Type: {response.headers.get('Content-Type')}")
        print(f"Response size: {len(response.content)} bytes")
        
        return response.text
        
    except requests.exceptions.ConnectionError:
        print("ไม่สามารถเชื่อมต่อได้")
    except requests.exceptions.Timeout:
        print("Request timeout")
    except requests.exceptions.HTTPError as e:
        print(f"HTTP Error: {e}")


# === Session สำหรับ Multiple Requests ===
def scrape_with_session():
    """ใช้ Session เพื่อ reuse connection และ cookies"""
    
    session = requests.Session()
    
    # ตั้ง headers สำหรับทุก requests
    session.headers.update({
        "User-Agent": "Mozilla/5.0 (compatible; MyCrawler/1.0)",
    })
    
    # Retry strategy - retry เมื่อ connection error หรือ 5xx
    retry_strategy = Retry(
        total=3,               # retry สูงสุด 3 ครั้ง
        backoff_factor=1,      # รอ 1s, 2s, 4s
        status_forcelist=[429, 500, 502, 503, 504],
        allowed_methods=["HEAD", "GET", "OPTIONS"]
    )
    
    adapter = HTTPAdapter(max_retries=retry_strategy)
    session.mount("https://", adapter)
    session.mount("http://", adapter)
    
    try:
        # Login (ตัวอย่าง)
        login_data = {"username": "user", "password": "pass"}
        # response = session.post("https://example.com/login", data=login_data)
        
        # ใช้ session ที่ login แล้ว
        r1 = session.get("https://httpbin.org/get")
        r2 = session.get("https://httpbin.org/cookies")
        
        print(f"Request 1: {r1.status_code}")
        print(f"Request 2: {r2.status_code}")
        
    finally:
        session.close()


# === Handling Different Response Types ===
def handle_response_types():
    session = requests.Session()
    
    # JSON Response
    r = session.get("https://httpbin.org/json")
    if r.headers.get("Content-Type", "").startswith("application/json"):
        data = r.json()
        print(f"JSON: {data}")
    
    # Binary Response (ดาวน์โหลดไฟล์)
    # r = session.get("https://example.com/image.jpg")
    # with open("image.jpg", "wb") as f:
    #     f.write(r.content)
    
    # Stream large files
    # r = session.get("https://example.com/large-file.csv", stream=True)
    # with open("large-file.csv", "wb") as f:
    #     for chunk in r.iter_content(chunk_size=8192):
    #         f.write(chunk)
```

---

## 3. BeautifulSoup4

```python
from bs4 import BeautifulSoup
import requests

# HTML ตัวอย่าง
sample_html = """
<!DOCTYPE html>
<html>
<head><title>สินค้าทั้งหมด</title></head>
<body>
    <div class="container">
        <h1 id="title">รายการสินค้า</h1>
        <nav>
            <a href="/page/1" class="page-link active">1</a>
            <a href="/page/2" class="page-link">2</a>
            <a href="/page/3" class="page-link">3</a>
        </nav>
        <div class="products">
            <div class="product-card" data-id="1">
                <h2 class="product-name">Laptop Pro</h2>
                <span class="price">฿35,000</span>
                <span class="rating" data-score="4.5">⭐ 4.5</span>
                <p class="description">Laptop รุ่นใหม่ประสิทธิภาพสูง</p>
                <a href="/product/1" class="product-link">ดูรายละเอียด</a>
            </div>
            <div class="product-card" data-id="2">
                <h2 class="product-name">Wireless Mouse</h2>
                <span class="price">฿800</span>
                <span class="rating" data-score="4.2">⭐ 4.2</span>
                <p class="description">เมาส์ไร้สายใช้งานได้นาน</p>
                <a href="/product/2" class="product-link">ดูรายละเอียด</a>
            </div>
            <div class="product-card" data-id="3">
                <h2 class="product-name">Mechanical Keyboard</h2>
                <span class="price">฿2,500</span>
                <span class="rating" data-score="4.8">⭐ 4.8</span>
                <p class="description">คีย์บอร์ด Mechanical คุณภาพดี</p>
                <a href="/product/3" class="product-link">ดูรายละเอียด</a>
            </div>
        </div>
    </div>
</body>
</html>
"""

# === Parser Choices ===
# html.parser - Built-in Python, ช้ากว่า lxml แต่ไม่ต้องติดตั้งเพิ่ม
# lxml - เร็วที่สุด, แนะนำสำหรับ production
# html5lib - แม่นยำที่สุดสำหรับ HTML5, ช้าที่สุด

soup = BeautifulSoup(sample_html, "lxml")

# === Finding Elements ===
def demo_beautifulsoup():
    
    # find() - หาตัวแรก
    title = soup.find("h1")
    print(f"Title: {title.text}")
    
    # find() ด้วย attributes
    title_by_id = soup.find("h1", id="title")
    print(f"By ID: {title_by_id.text}")
    
    # find_all() - หาทั้งหมด
    all_products = soup.find_all("div", class_="product-card")
    print(f"Number of products: {len(all_products)}")
    
    # Nested navigation
    for product in all_products:
        name = product.find("h2", class_="product-name").text
        price = product.find("span", class_="price").text
        rating = product.find("span", class_="rating")["data-score"]
        link = product.find("a", class_="product-link")["href"]
        product_id = product["data-id"]
        
        print(f"[ID:{product_id}] {name} - {price} (Rating: {rating}) -> {link}")
    
    # CSS Selectors ด้วย select()
    print("\n--- CSS Selectors ---")
    
    # select() คืน list เสมอ
    products_by_css = soup.select("div.product-card")
    print(f"Products by CSS: {len(products_by_css)}")
    
    # Complex selectors
    names = soup.select("div.product-card h2.product-name")
    for name in names:
        print(f"  - {name.text}")
    
    # select_one() - หาตัวแรก
    first_price = soup.select_one(".product-card .price")
    print(f"First price: {first_price.text}")
    
    # Attribute selector
    links = soup.select("a[href^='/product/']")  # href ขึ้นต้นด้วย /product/
    for link in links:
        print(f"  Link: {link['href']}")
    
    # Pagination links
    page_links = soup.select("nav a.page-link")
    for link in page_links:
        is_active = "active" in link.get("class", [])
        print(f"  Page {link.text}: {link['href']} {'(active)' if is_active else ''}")


demo_beautifulsoup()
```

---

## 4. CSS Selectors และ XPath

```python
from bs4 import BeautifulSoup
from lxml import etree

# === CSS Selector Reference ===
def css_selector_examples():
    soup = BeautifulSoup(sample_html, "lxml")
    
    # Basic selectors
    print("=== Basic Selectors ===")
    print(soup.select("h1"))                    # element
    print(soup.select(".product-card"))          # class
    print(soup.select("#title"))                 # id
    print(soup.select("div.product-card h2"))    # descendant
    print(soup.select("div > .price"))           # direct child
    
    # Attribute selectors
    print("\n=== Attribute Selectors ===")
    print(soup.select("[data-id]"))              # has attribute
    print(soup.select("[data-id='1']"))          # exact value
    print(soup.select("[href^='/product']"))     # starts with
    print(soup.select("[href$='/1']"))           # ends with
    print(soup.select("[href*='product']"))      # contains
    
    # Pseudo-classes
    print("\n=== Pseudo-classes ===")
    print(soup.select("a.page-link:first-child"))
    print(soup.select("a.page-link:last-child"))
    print(soup.select("div.product-card:nth-child(2)"))
    
    # Combining
    first_product_name = soup.select_one(
        "div.products > div.product-card:first-child .product-name"
    )
    print(f"First product: {first_product_name.text}")


# === XPath ด้วย lxml ===
def xpath_examples():
    # lxml ต้องการ bytes หรือ lxml tree
    tree = etree.fromstring(sample_html.encode())
    
    # XPath Basics
    # // - anywhere in document
    # / - direct child
    # @ - attribute
    # text() - text content
    # contains() - contains string
    # starts-with() - starts with
    
    # หาทุก product names
    names = tree.xpath("//h2[@class='product-name']/text()")
    print(f"Product names: {names}")
    
    # หาทุก prices
    prices = tree.xpath("//span[@class='price']/text()")
    print(f"Prices: {prices}")
    
    # หาด้วย data attribute
    product_ids = tree.xpath("//div[@data-id]/@data-id")
    print(f"Product IDs: {product_ids}")
    
    # หา links ที่ href contains 'product'
    links = tree.xpath("//a[contains(@href, '/product/')]/@href")
    print(f"Product links: {links}")
    
    # หา ratings ที่ score >= 4.5
    high_rated = tree.xpath("//span[@class='rating'][@data-score >= 4.5]")
    for elem in high_rated:
        print(f"High rated: {elem.text}")
    
    # Navigate parent/sibling
    prices_with_context = tree.xpath(
        "//div[@class='product-card'][.//span[@class='price']]//h2/text()"
    )
    print(f"Products with prices: {prices_with_context}")


css_selector_examples()
# xpath_examples()  # Uncomment to test XPath
```

---

## 5. Scraping Tables

```python
table_html = """
<table class="data-table" id="sales-table">
    <thead>
        <tr>
            <th>วันที่</th>
            <th>สินค้า</th>
            <th>จำนวน</th>
            <th>ราคา/หน่วย</th>
            <th>รวม</th>
        </tr>
    </thead>
    <tbody>
        <tr class="row-data">
            <td>2024-01-15</td>
            <td>Laptop Pro</td>
            <td>2</td>
            <td>35,000</td>
            <td>70,000</td>
        </tr>
        <tr class="row-data">
            <td>2024-01-16</td>
            <td>Wireless Mouse</td>
            <td>5</td>
            <td>800</td>
            <td>4,000</td>
        </tr>
        <tr class="row-data">
            <td>2024-01-17</td>
            <td>Keyboard</td>
            <td>3</td>
            <td>2,500</td>
            <td>7,500</td>
        </tr>
    </tbody>
    <tfoot>
        <tr>
            <td colspan="4">รวมทั้งหมด</td>
            <td>81,500</td>
        </tr>
    </tfoot>
</table>
"""

def scrape_table():
    """แปลง HTML table เป็น list of dicts"""
    soup = BeautifulSoup(table_html, "lxml")
    table = soup.find("table", id="sales-table")
    
    if not table:
        return []
    
    # ดึง headers
    headers = []
    header_row = table.find("thead")
    if header_row:
        headers = [th.text.strip() for th in header_row.find_all("th")]
    
    print(f"Headers: {headers}")
    
    # ดึง data rows
    rows = []
    tbody = table.find("tbody")
    if tbody:
        for tr in tbody.find_all("tr", class_="row-data"):
            cells = [td.text.strip() for td in tr.find_all("td")]
            
            # สร้าง dict จาก headers + cells
            if len(cells) == len(headers):
                row_dict = dict(zip(headers, cells))
                rows.append(row_dict)
    
    print(f"\nScraped {len(rows)} rows:")
    for row in rows:
        print(f"  {row}")
    
    return rows


# === แปลงเป็น pandas DataFrame ===
def table_to_dataframe():
    """ใช้ pandas อ่าน HTML tables โดยตรง"""
    try:
        import pandas as pd
        
        # pandas สามารถ read HTML tables ได้โดยตรง
        tables = pd.read_html(table_html, header=0)
        
        if tables:
            df = tables[0]
            print(f"DataFrame shape: {df.shape}")
            print(df.to_string())
            return df
    except ImportError:
        print("pandas not installed, install with: pip install pandas")
    
    return None


scrape_table()
```

---

## 6. Handling Pagination

```python
import requests
from bs4 import BeautifulSoup
from typing import Generator, List
from dataclasses import dataclass
import time

@dataclass
class Product:
    name: str
    price: str
    url: str
    page: int


def scrape_all_pages(base_url: str, max_pages: int = 10) -> List[Product]:
    """
    Scrape หลายหน้าจาก website ที่มี pagination
    ตัวอย่าง: https://example.com/products?page=1
    """
    session = requests.Session()
    session.headers.update({
        "User-Agent": "Mozilla/5.0 (compatible; PriceBot/1.0)"
    })
    
    all_products = []
    
    for page_num in range(1, max_pages + 1):
        # สร้าง URL สำหรับแต่ละหน้า
        url = f"{base_url}?page={page_num}"
        
        print(f"Scraping page {page_num}: {url}")
        
        try:
            response = session.get(url, timeout=15)
            response.raise_for_status()
        except requests.RequestException as e:
            print(f"Error on page {page_num}: {e}")
            break
        
        soup = BeautifulSoup(response.text, "lxml")
        
        # ดึงสินค้าในหน้านี้
        products = extract_products(soup, page_num)
        
        if not products:
            print(f"No products found on page {page_num}, stopping")
            break
        
        all_products.extend(products)
        print(f"  Found {len(products)} products")
        
        # ตรวจสอบว่ายังมีหน้าถัดไปไหม
        next_link = soup.select_one("a.next-page, a[rel='next'], .pagination .next")
        if not next_link:
            print("No next page link found, done")
            break
        
        # Rate limiting - สำคัญมาก!
        time.sleep(1)  # รอ 1 วินาทีระหว่างหน้า
    
    return all_products


def extract_products(soup: BeautifulSoup, page: int) -> List[Product]:
    """Extract products จาก soup"""
    products = []
    
    for card in soup.select(".product-card"):
        name_el = card.select_one(".product-name")
        price_el = card.select_one(".price")
        link_el = card.select_one("a.product-link")
        
        if name_el and price_el:
            products.append(Product(
                name=name_el.text.strip(),
                price=price_el.text.strip(),
                url=link_el.get("href", "") if link_el else "",
                page=page
            ))
    
    return products


# === Generator-based Scraper (ประหยัด Memory) ===
def scrape_pages_generator(
    base_url: str, 
    max_pages: int = 100
) -> Generator[Product, None, None]:
    """
    Generator version - yield products ทีละตัว
    เหมาะสำหรับ dataset ขนาดใหญ่
    """
    session = requests.Session()
    session.headers.update({"User-Agent": "Mozilla/5.0"})
    
    page = 1
    while page <= max_pages:
        try:
            response = session.get(f"{base_url}?page={page}", timeout=10)
            response.raise_for_status()
            soup = BeautifulSoup(response.text, "lxml")
            
            products = extract_products(soup, page)
            if not products:
                return
            
            for product in products:
                yield product  # yield ทีละตัว
            
            # Check next page
            if not soup.select_one("a[rel='next']"):
                return
            
            page += 1
            time.sleep(1)
            
        except Exception as e:
            print(f"Error on page {page}: {e}")
            return
    
    session.close()


# ใช้ Generator
def process_large_dataset():
    # ไม่ต้อง load ทั้งหมดในหน้า memory
    for product in scrape_pages_generator("https://example.com/products"):
        # Process ทีละตัว
        print(f"Processing: {product.name}")
        # save_to_db(product)
```

---

## 7. Selenium สำหรับ Dynamic Content

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service
from webdriver_manager.chrome import ChromeDriverManager
import time

def create_driver(headless: bool = True) -> webdriver.Chrome:
    """สร้าง Chrome WebDriver"""
    options = Options()
    
    if headless:
        options.add_argument("--headless")  # ไม่แสดง browser window
    
    # ป้องกันการถูก detect ว่าเป็น bot
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--disable-blink-features=AutomationControlled")
    options.add_experimental_option("excludeSwitches", ["enable-automation"])
    options.add_experimental_option("useAutomationExtension", False)
    options.add_argument(
        "--user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"
    )
    
    # ติดตั้ง ChromeDriver อัตโนมัติ
    service = Service(ChromeDriverManager().install())
    driver = webdriver.Chrome(service=service, options=options)
    
    # Execute script เพื่อซ่อน webdriver
    driver.execute_script(
        "Object.defineProperty(navigator, 'webdriver', {get: () => undefined})"
    )
    
    return driver


def scrape_dynamic_page(url: str):
    """Scrape เว็บที่ใช้ JavaScript render content"""
    driver = create_driver(headless=True)
    
    try:
        driver.get(url)
        
        # รอให้ element โหลด (timeout 10 วินาที)
        wait = WebDriverWait(driver, 10)
        
        # รอจน element ปรากฏ
        products_container = wait.until(
            EC.presence_of_element_located((By.CLASS_NAME, "products"))
        )
        
        # รอจนโหลดเสร็จ
        wait.until(
            EC.presence_of_all_elements_located((By.CSS_SELECTOR, ".product-card"))
        )
        
        # ดึงข้อมูลหลัง JavaScript render
        products = driver.find_elements(By.CSS_SELECTOR, ".product-card")
        
        data = []
        for product in products:
            name = product.find_element(By.CSS_SELECTOR, ".product-name").text
            price = product.find_element(By.CSS_SELECTOR, ".price").text
            data.append({"name": name, "price": price})
        
        return data
        
    finally:
        driver.quit()


def scroll_and_scrape(url: str):
    """Scrape เว็บที่ใช้ Infinite Scroll"""
    driver = create_driver(headless=True)
    
    try:
        driver.get(url)
        time.sleep(2)
        
        all_products = set()
        last_height = 0
        
        while True:
            # Scroll down
            driver.execute_script("window.scrollTo(0, document.body.scrollHeight);")
            time.sleep(2)  # รอให้ content โหลด
            
            # ดึง products ที่มีอยู่
            products = driver.find_elements(By.CSS_SELECTOR, ".product-card")
            for p in products:
                name = p.find_element(By.CSS_SELECTOR, ".product-name").text
                all_products.add(name)
            
            # ตรวจสอบว่า scroll ถึงด้านล่างหรือยัง
            new_height = driver.execute_script("return document.body.scrollHeight")
            if new_height == last_height:
                break  # ถึงด้านล่างแล้ว
            last_height = new_height
        
        print(f"Total unique products found: {len(all_products)}")
        return list(all_products)
        
    finally:
        driver.quit()


def click_and_scrape(url: str):
    """Scrape เว็บที่ต้องกดปุ่มเพื่อโหลด content"""
    driver = create_driver(headless=True)
    wait = WebDriverWait(driver, 10)
    
    try:
        driver.get(url)
        all_data = []
        
        while True:
            # ดึง data จากหน้าปัจจุบัน
            products = driver.find_elements(By.CSS_SELECTOR, ".product-card")
            for p in products:
                all_data.append(p.text)
            
            # หาปุ่ม "Load More" หรือ "Next"
            try:
                load_more = wait.until(
                    EC.element_to_be_clickable((By.CSS_SELECTOR, ".load-more"))
                )
                load_more.click()
                time.sleep(2)
            except:
                break  # ไม่มีปุ่มอีกแล้ว
        
        return all_data
        
    finally:
        driver.quit()
```

---

## 8. Rate Limiting และ Robots.txt

```python
import time
import random
import urllib.robotparser
from threading import Semaphore
from functools import wraps
from dataclasses import dataclass

# === ตรวจสอบ robots.txt ===
def check_robots_txt(base_url: str, path: str) -> bool:
    """ตรวจสอบว่า crawl ได้ไหมตาม robots.txt"""
    rp = urllib.robotparser.RobotFileParser()
    rp.set_url(f"{base_url}/robots.txt")
    
    try:
        rp.read()
        user_agent = "MyCrawler"
        can_fetch = rp.can_fetch(user_agent, path)
        
        if not can_fetch:
            print(f"⚠️  robots.txt ไม่อนุญาตให้ crawl: {path}")
        
        # ตรวจสอบ Crawl-delay
        crawl_delay = rp.crawl_delay(user_agent)
        if crawl_delay:
            print(f"Crawl-delay: {crawl_delay} seconds")
        
        return can_fetch
    except Exception as e:
        print(f"Cannot read robots.txt: {e}")
        return True  # Default: allow ถ้าอ่าน robots.txt ไม่ได้


# === Rate Limiter ===
@dataclass
class RateLimiter:
    """จำกัดจำนวน requests ต่อวินาที"""
    requests_per_second: float = 1.0
    
    def __post_init__(self):
        self._min_interval = 1.0 / self.requests_per_second
        self._last_request_time = 0.0
    
    def wait(self):
        """รอให้ถึงเวลา request ครั้งต่อไป"""
        now = time.time()
        elapsed = now - self._last_request_time
        wait_time = self._min_interval - elapsed
        
        if wait_time > 0:
            # เพิ่ม jitter เพื่อไม่ให้ pattern ชัดเจนเกินไป
            jitter = random.uniform(0, wait_time * 0.1)
            time.sleep(wait_time + jitter)
        
        self._last_request_time = time.time()


class PoliteScraperConfig:
    """Configuration สำหรับ Polite Scraping"""
    
    def __init__(
        self,
        delay_between_requests: float = 2.0,
        max_requests_per_domain: int = 100,
        respect_robots_txt: bool = True,
        random_delay: bool = True
    ):
        self.delay_between_requests = delay_between_requests
        self.max_requests_per_domain = max_requests_per_domain
        self.respect_robots_txt = respect_robots_txt
        self.random_delay = random_delay


class PoliteScraper:
    """Scraper ที่เคารพ website กฎต่างๆ"""
    
    def __init__(self, config: PoliteScraperConfig = None):
        self.config = config or PoliteScraperConfig()
        self._request_count: dict = {}
        self._session = requests.Session()
        self._session.headers.update({
            "User-Agent": "PoliteCrawler/1.0 (+https://mysite.com/bot)",
            "Accept": "text/html,application/xhtml+xml",
        })
    
    def _get_delay(self) -> float:
        """คำนวณ delay ระหว่าง requests"""
        base_delay = self.config.delay_between_requests
        if self.config.random_delay:
            # Random delay ระหว่าง 1x-3x ของ base delay
            return base_delay * random.uniform(1.0, 3.0)
        return base_delay
    
    def fetch(self, url: str) -> requests.Response:
        """Fetch URL ด้วย politeness rules"""
        from urllib.parse import urlparse
        domain = urlparse(url).netloc
        
        # Check robots.txt
        if self.config.respect_robots_txt:
            parsed = urlparse(url)
            base = f"{parsed.scheme}://{parsed.netloc}"
            path = parsed.path
            if not check_robots_txt(base, path):
                raise ValueError(f"robots.txt disallows: {url}")
        
        # Check request limit per domain
        self._request_count[domain] = self._request_count.get(domain, 0) + 1
        if self._request_count[domain] > self.config.max_requests_per_domain:
            raise ValueError(f"Max requests exceeded for domain: {domain}")
        
        # Apply delay
        time.sleep(self._get_delay())
        
        # Make request
        response = self._session.get(url, timeout=15)
        response.raise_for_status()
        
        print(f"  [{self._request_count[domain]}] Fetched: {url}")
        return response
    
    def scrape_products(self, urls: list) -> list:
        """Scrape หลาย URLs"""
        results = []
        
        for url in urls:
            try:
                response = self.fetch(url)
                soup = BeautifulSoup(response.text, "lxml")
                products = extract_products(soup, 0)
                results.extend(products)
            except Exception as e:
                print(f"Error scraping {url}: {e}")
        
        return results


# ตัวอย่างการใช้งาน
def demo_polite_scraping():
    config = PoliteScraperConfig(
        delay_between_requests=1.5,
        max_requests_per_domain=50,
        respect_robots_txt=True,
        random_delay=True
    )
    
    scraper = PoliteScraper(config)
    
    # ตรวจสอบ robots.txt
    can_crawl = check_robots_txt("https://httpbin.org", "/get")
    print(f"Can crawl httpbin.org/get: {can_crawl}")
    
    print("Polite scraper initialized with:")
    print(f"  Delay: {config.delay_between_requests}s")
    print(f"  Max requests/domain: {config.max_requests_per_domain}")
    print(f"  Respect robots.txt: {config.respect_robots_txt}")


demo_polite_scraping()
```

---

## 9. Complete Scraping Project

```python
import requests
from bs4 import BeautifulSoup
from dataclasses import dataclass, field, asdict
from typing import List, Optional
import csv
import json
import time
import logging

# ตั้ง logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s"
)
logger = logging.getLogger(__name__)


@dataclass
class ScrapedProduct:
    name: str
    price: float
    url: str
    rating: Optional[float] = None
    category: Optional[str] = None
    description: Optional[str] = None
    scraped_at: str = field(default_factory=lambda: 
                             time.strftime("%Y-%m-%d %H:%M:%S"))
    
    def clean_price(self, price_str: str) -> float:
        """แปลง price string เป็น float"""
        cleaned = price_str.replace("฿", "").replace(",", "").strip()
        try:
            return float(cleaned)
        except ValueError:
            return 0.0


class ProductScraper:
    """Complete product scraper"""
    
    BASE_URL = "https://books.toscrape.com"  # Website สำหรับฝึก scraping
    
    def __init__(self):
        self.session = requests.Session()
        self.session.headers.update({
            "User-Agent": "Mozilla/5.0 (compatible; ProductScraper/1.0)",
        })
        self.scraped_products: List[ScrapedProduct] = []
    
    def scrape_book_listing(self, url: str, page: int = 1) -> List[dict]:
        """Scrape book listing page"""
        page_url = f"{url}/catalogue/page-{page}.html" if page > 1 else url
        
        try:
            logger.info(f"Scraping page {page}: {page_url}")
            response = self.session.get(page_url, timeout=15)
            response.raise_for_status()
            response.encoding = "utf-8"
            
            soup = BeautifulSoup(response.text, "lxml")
            
            books = []
            for article in soup.select("article.product_pod"):
                book = self._extract_book(article)
                if book:
                    books.append(book)
            
            logger.info(f"Found {len(books)} books on page {page}")
            return books
            
        except requests.RequestException as e:
            logger.error(f"Request failed for page {page}: {e}")
            return []
    
    def _extract_book(self, article) -> Optional[dict]:
        """Extract book data จาก article element"""
        try:
            # ชื่อหนังสือ
            title_elem = article.select_one("h3 > a")
            title = title_elem.get("title", title_elem.text) if title_elem else "Unknown"
            
            # ราคา
            price_elem = article.select_one(".price_color")
            price_text = price_elem.text if price_elem else "£0"
            price = float(price_text.replace("£", "").replace("Â", "").strip())
            
            # Rating (class name เช่น "One", "Two", ... "Five")
            rating_elem = article.select_one(".star-rating")
            rating_map = {"One": 1, "Two": 2, "Three": 3, "Four": 4, "Five": 5}
            rating = None
            if rating_elem:
                classes = rating_elem.get("class", [])
                for cls in classes:
                    if cls in rating_map:
                        rating = rating_map[cls]
                        break
            
            # Availability
            availability_elem = article.select_one(".availability")
            in_stock = availability_elem and "In stock" in availability_elem.text
            
            # URL
            link_elem = article.select_one("h3 > a")
            url = ""
            if link_elem:
                href = link_elem.get("href", "")
                url = f"{self.BASE_URL}/catalogue/{href.replace('../', '')}"
            
            return {
                "title": title,
                "price": price,
                "rating": rating,
                "in_stock": in_stock,
                "url": url
            }
        except Exception as e:
            logger.warning(f"Error extracting book: {e}")
            return None
    
    def scrape_all_pages(self, max_pages: int = 5) -> List[dict]:
        """Scrape ทุกหน้า"""
        all_books = []
        
        for page in range(1, max_pages + 1):
            books = self.scrape_book_listing(self.BASE_URL, page)
            
            if not books:
                logger.info(f"No books found on page {page}, stopping")
                break
            
            all_books.extend(books)
            
            if page < max_pages:
                time.sleep(1)  # Rate limiting
        
        return all_books
    
    def save_to_csv(self, data: List[dict], filename: str):
        """บันทึกเป็น CSV"""
        if not data:
            return
        
        with open(filename, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=data[0].keys())
            writer.writeheader()
            writer.writerows(data)
        
        logger.info(f"Saved {len(data)} records to {filename}")
    
    def save_to_json(self, data: List[dict], filename: str):
        """บันทึกเป็น JSON"""
        with open(filename, "w", encoding="utf-8") as f:
            json.dump(data, f, ensure_ascii=False, indent=2)
        
        logger.info(f"Saved {len(data)} records to {filename}")
    
    def get_statistics(self, books: List[dict]) -> dict:
        """คำนวณ statistics"""
        if not books:
            return {}
        
        prices = [b["price"] for b in books if b["price"] > 0]
        ratings = [b["rating"] for b in books if b["rating"]]
        
        return {
            "total_books": len(books),
            "avg_price": sum(prices) / len(prices) if prices else 0,
            "min_price": min(prices) if prices else 0,
            "max_price": max(prices) if prices else 0,
            "avg_rating": sum(ratings) / len(ratings) if ratings else 0,
            "in_stock": sum(1 for b in books if b.get("in_stock", False)),
        }


# ทดสอบ Scraper (จะ work ถ้ามี internet connection)
def demo_scraper():
    scraper = ProductScraper()
    
    print("Starting scraper...")
    print(f"Target: {scraper.BASE_URL}")
    
    # Scrape 2 pages เพื่อ demo
    # books = scraper.scrape_all_pages(max_pages=2)
    
    # Mock data สำหรับ demo ไม่มี internet
    books = [
        {"title": "Book A", "price": 12.99, "rating": 4, "in_stock": True, "url": "url1"},
        {"title": "Book B", "price": 8.50, "rating": 3, "in_stock": True, "url": "url2"},
        {"title": "Book C", "price": 25.00, "rating": 5, "in_stock": False, "url": "url3"},
    ]
    
    stats = scraper.get_statistics(books)
    print(f"\nStatistics:")
    for key, value in stats.items():
        if isinstance(value, float):
            print(f"  {key}: {value:.2f}")
        else:
            print(f"  {key}: {value}")


demo_scraper()
```

---

## 10. สรุป Part 039

✅ **requests Library** - HTTP requests, sessions, retry, headers  
✅ **BeautifulSoup4** - Parse HTML, find elements, navigate tree  
✅ **CSS Selectors** - select(), select_one(), complex selectors  
✅ **XPath** - ด้วย lxml สำหรับ complex queries  
✅ **Scraping Tables** - แปลง HTML tables เป็น data  
✅ **Pagination** - Scrape หลายหน้าอย่างมีประสิทธิภาพ  
✅ **Selenium** - Dynamic content ที่ต้องการ JavaScript  
✅ **Rate Limiting** - Polite scraping, robots.txt, delays  

**Best Practices:**
- เสมอตรวจสอบ robots.txt ก่อน
- ใช้ delay ระหว่าง requests
- ใช้ realistic User-Agent
- Handle errors gracefully
- เก็บ data ในรูปแบบที่เหมาะสม

## ➡️ ถัดไป: Part 040 - HTTP and Requests
*Part 039/100+ | Python Course - Beginner to World-Class*
