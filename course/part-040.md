# Part 040: HTTP and Requests
## หลักสูตร Python, Django, Flask, FastAPI

---

## 🎯 เป้าหมายของ Part นี้

- ใช้ requests library ทุก HTTP methods ได้
- จัดการ Headers, Authentication, Sessions
- ทำงานกับ Cookies และ Redirects
- ตั้งค่า Timeout และ Retry
- Handle Responses อย่างถูกต้อง
- ทดสอบ API calls ด้วย Mock

---

## 1. HTTP Methods ครบทุกประเภท

```python
import requests
import json

BASE_URL = "https://jsonplaceholder.typicode.com"

# === GET Request ===
def demo_get():
    # Basic GET
    response = requests.get(f"{BASE_URL}/posts/1")
    print(f"Status: {response.status_code}")
    post = response.json()
    print(f"Post: {post['title'][:50]}")
    
    # GET with query parameters
    params = {
        "userId": 1,
        "_limit": 5,
        "_sort": "id",
        "_order": "desc"
    }
    response = requests.get(f"{BASE_URL}/posts", params=params)
    print(f"\nPosts by user 1: {len(response.json())} posts")
    print(f"URL built: {response.url}")
    
    # GET all resources
    response = requests.get(
        f"{BASE_URL}/users",
        params={"_fields": "id,name,email"}
    )
    users = response.json()
    for user in users[:3]:
        print(f"  {user}")


# === POST Request ===
def demo_post():
    new_post = {
        "title": "บทความใหม่",
        "body": "เนื้อหาบทความ",
        "userId": 1
    }
    
    # POST JSON data
    response = requests.post(
        f"{BASE_URL}/posts",
        json=new_post,  # จะ set Content-Type: application/json อัตโนมัติ
        headers={"X-Custom-Header": "MyApp/1.0"}
    )
    
    print(f"POST Status: {response.status_code}")
    created = response.json()
    print(f"Created post ID: {created['id']}")
    
    # POST Form data
    form_data = {"username": "testuser", "password": "secret"}
    response = requests.post(
        "https://httpbin.org/post",
        data=form_data  # จะ set Content-Type: application/x-www-form-urlencoded
    )
    
    # POST Multipart form (file upload)
    # with open("image.jpg", "rb") as f:
    #     files = {"photo": ("image.jpg", f, "image/jpeg")}
    #     response = requests.post(url, files=files)


# === PUT Request ===
def demo_put():
    updated_post = {
        "id": 1,
        "title": "อัปเดตบทความ",
        "body": "เนื้อหาใหม่",
        "userId": 1
    }
    
    response = requests.put(
        f"{BASE_URL}/posts/1",
        json=updated_post
    )
    print(f"PUT Status: {response.status_code}")
    print(f"Updated: {response.json()['title']}")


# === PATCH Request (Partial Update) ===
def demo_patch():
    # ส่งเฉพาะ fields ที่ต้องการ update
    partial_update = {"title": "ชื่อใหม่เท่านั้น"}
    
    response = requests.patch(
        f"{BASE_URL}/posts/1",
        json=partial_update
    )
    print(f"PATCH Status: {response.status_code}")
    print(f"Patched title: {response.json()['title']}")


# === DELETE Request ===
def demo_delete():
    response = requests.delete(f"{BASE_URL}/posts/1")
    print(f"DELETE Status: {response.status_code}")
    # 200 = deleted successfully


# === HEAD Request (ดู headers โดยไม่ดาวน์โหลด body) ===
def demo_head():
    response = requests.head("https://httpbin.org/get")
    print(f"Content-Type: {response.headers.get('Content-Type')}")
    print(f"Server: {response.headers.get('Server')}")


# === OPTIONS Request ===
def demo_options():
    response = requests.options("https://httpbin.org/anything")
    print(f"Allowed methods: {response.headers.get('Allow')}")


# Run all demos
print("=== GET ===")
demo_get()
print("\n=== POST ===")
demo_post()
print("\n=== PUT ===")
demo_put()
print("\n=== PATCH ===")
demo_patch()
print("\n=== DELETE ===")
demo_delete()
```

---

## 2. Headers

```python
import requests

# === Custom Headers ===
def demo_headers():
    headers = {
        # Standard headers
        "User-Agent": "MyApp/1.0 (Python requests)",
        "Accept": "application/json",
        "Accept-Language": "th-TH,th;q=0.9,en;q=0.8",
        "Accept-Encoding": "gzip, deflate",
        "Connection": "keep-alive",
        "Cache-Control": "no-cache",
        
        # Custom application headers
        "X-App-Version": "2.1.0",
        "X-Request-ID": "abc-123-def",
        "X-Client-Platform": "python",
    }
    
    response = requests.get("https://httpbin.org/headers", headers=headers)
    
    # ดู headers ที่ส่งไป
    sent_headers = response.json()["headers"]
    print("Sent headers:")
    for key, value in sent_headers.items():
        print(f"  {key}: {value}")
    
    # ดู response headers
    print("\nResponse headers:")
    for key, value in response.headers.items():
        print(f"  {key}: {value}")
    
    # ตรวจสอบ specific header
    content_type = response.headers.get("Content-Type", "unknown")
    print(f"\nContent-Type: {content_type}")
    
    # case-insensitive (Content-Type == content-type)
    print(f"Same: {response.headers['content-type'] == response.headers['Content-Type']}")


# === Accept Headers สำหรับ Content Negotiation ===
def demo_content_negotiation():
    # Request JSON
    r_json = requests.get(
        "https://httpbin.org/anything",
        headers={"Accept": "application/json"}
    )
    
    # Request specific version
    r_versioned = requests.get(
        "https://api.example.com/users",  # ตัวอย่าง
        headers={"Accept": "application/vnd.myapi.v2+json"}
    )


demo_headers()
```

---

## 3. Authentication

```python
import requests
from requests.auth import HTTPBasicAuth, HTTPDigestAuth
import base64

# === Basic Authentication ===
def demo_basic_auth():
    # วิธีที่ 1: ใช้ auth parameter
    response = requests.get(
        "https://httpbin.org/basic-auth/user/pass",
        auth=("user", "pass")  # (username, password)
    )
    print(f"Basic Auth: {response.status_code}")
    
    # วิธีที่ 2: ใช้ HTTPBasicAuth class
    response = requests.get(
        "https://httpbin.org/basic-auth/user/pass",
        auth=HTTPBasicAuth("user", "pass")
    )
    print(f"HTTPBasicAuth: {response.status_code}")
    
    # วิธีที่ 3: ใส่ใน header โดยตรง
    credentials = base64.b64encode("user:pass".encode()).decode()
    response = requests.get(
        "https://httpbin.org/basic-auth/user/pass",
        headers={"Authorization": f"Basic {credentials}"}
    )
    print(f"Manual Basic: {response.status_code}")


# === Bearer Token Authentication (JWT) ===
def demo_bearer_auth():
    token = "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
    
    response = requests.get(
        "https://api.example.com/protected",
        headers={"Authorization": f"Bearer {token}"}
    )


# === API Key Authentication ===
def demo_api_key():
    api_key = "your-api-key-here"
    
    # วิธีที่ 1: ใน header
    response = requests.get(
        "https://api.example.com/data",
        headers={"X-API-Key": api_key}
    )
    
    # วิธีที่ 2: ใน query parameter
    response = requests.get(
        "https://api.example.com/data",
        params={"api_key": api_key}
    )
    
    # วิธีที่ 3: ใน Authorization header
    response = requests.get(
        "https://api.example.com/data",
        headers={"Authorization": f"ApiKey {api_key}"}
    )


# === OAuth 2.0 (ด้วย requests-oauthlib) ===
# pip install requests-oauthlib
def demo_oauth2():
    """
    OAuth 2.0 Client Credentials Flow
    ใช้สำหรับ server-to-server authentication
    """
    from requests.auth import HTTPBasicAuth
    
    # Step 1: Get access token
    token_url = "https://oauth.example.com/token"
    client_id = "your-client-id"
    client_secret = "your-client-secret"
    
    token_response = requests.post(
        token_url,
        auth=HTTPBasicAuth(client_id, client_secret),
        data={
            "grant_type": "client_credentials",
            "scope": "read write"
        }
    )
    
    # token_data = token_response.json()
    # access_token = token_data["access_token"]
    
    # Step 2: Use access token
    # response = requests.get(
    #     "https://api.example.com/resource",
    #     headers={"Authorization": f"Bearer {access_token}"}
    # )


# === Custom Auth Class ===
class HMACAuth(requests.auth.AuthBase):
    """Custom HMAC authentication"""
    
    def __init__(self, api_key: str, api_secret: str):
        self.api_key = api_key
        self.api_secret = api_secret
    
    def __call__(self, request):
        import hashlib
        import hmac
        import time
        
        timestamp = str(int(time.time()))
        
        # สร้าง signature
        message = f"{timestamp}{request.method}{request.url}"
        signature = hmac.new(
            self.api_secret.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()
        
        # เพิ่ม headers
        request.headers["X-API-Key"] = self.api_key
        request.headers["X-Timestamp"] = timestamp
        request.headers["X-Signature"] = signature
        
        return request


def demo_custom_auth():
    auth = HMACAuth("my-api-key", "my-secret")
    response = requests.get("https://api.example.com/data", auth=auth)


demo_basic_auth()
```

---

## 4. Sessions และ Cookies

```python
import requests
from http.cookiejar import MozillaCookieJar
import json

# === Session ===
def demo_session():
    """Session รักษา cookies และ connection pool"""
    
    session = requests.Session()
    
    # ตั้ง persistent headers
    session.headers.update({
        "User-Agent": "MyApp/1.0",
        "Accept": "application/json",
    })
    
    # ตั้ง persistent auth
    # session.auth = ("user", "pass")
    
    # Login และ cookies จะถูกเก็บอัตโนมัติ
    # login_response = session.post("/login", data={"user": "u", "pass": "p"})
    
    # Subsequent requests ใช้ cookies เดิม
    response1 = session.get("https://httpbin.org/cookies/set/session_id/abc123")
    response2 = session.get("https://httpbin.org/cookies")
    
    print("Cookies in session:")
    cookies_data = response2.json()
    print(f"  {cookies_data}")
    
    # ดู cookies ทั้งหมด
    for cookie in session.cookies:
        print(f"  Cookie: {cookie.name}={cookie.value} (domain: {cookie.domain})")
    
    session.close()


# === Manual Cookie Management ===
def demo_cookies():
    session = requests.Session()
    
    # ตั้ง cookies โดยตรง
    session.cookies.set("theme", "dark", domain="httpbin.org")
    session.cookies.set("lang", "th", domain="httpbin.org")
    
    response = session.get("https://httpbin.org/cookies")
    print(f"Sent cookies: {response.json()}")
    
    # ส่ง cookies แบบ dict (ใน single request)
    cookies = {"tracking_id": "xyz789", "visited": "true"}
    response = requests.get("https://httpbin.org/cookies", cookies=cookies)
    print(f"One-time cookies: {response.json()}")
    
    # บันทึก cookies ลงไฟล์
    cookie_jar = MozillaCookieJar("cookies.txt")
    for cookie in session.cookies:
        cookie_jar.set_cookie(cookie)
    # cookie_jar.save(ignore_discard=True)
    
    # โหลด cookies จากไฟล์
    # new_jar = MozillaCookieJar()
    # new_jar.load("cookies.txt", ignore_discard=True)
    # session.cookies = new_jar
    
    session.close()


# === Handling Redirects ===
def demo_redirects():
    # ตามรอย redirects (default)
    response = requests.get("https://httpbin.org/redirect/3")
    print(f"Final URL: {response.url}")
    print(f"Status: {response.status_code}")
    
    # ดู redirect history
    print("Redirect history:")
    for r in response.history:
        print(f"  {r.status_code} -> {r.url}")
    
    # ไม่ตาม redirects
    response_no_redirect = requests.get(
        "https://httpbin.org/redirect/1",
        allow_redirects=False
    )
    print(f"\nNo redirect status: {response_no_redirect.status_code}")
    print(f"Location header: {response_no_redirect.headers.get('Location')}")


demo_session()
demo_cookies()
demo_redirects()
```

---

## 5. Timeout และ Retry

```python
import requests
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry
import time
from typing import Optional, Callable

# === Timeout ===
def demo_timeout():
    # Connection timeout vs Read timeout
    try:
        # (connect_timeout, read_timeout) in seconds
        response = requests.get(
            "https://httpbin.org/delay/1",
            timeout=(5, 10)  # 5s connect, 10s read
        )
        print(f"Got response in time: {response.status_code}")
        
    except requests.exceptions.ConnectTimeout:
        print("Connection timed out")
    except requests.exceptions.ReadTimeout:
        print("Read timed out")
    except requests.exceptions.Timeout:
        print("Request timed out")
    
    # Single timeout (applies to both connect and read)
    try:
        response = requests.get(
            "https://httpbin.org/get",
            timeout=10  # 10 seconds total
        )
    except requests.exceptions.Timeout as e:
        print(f"Timeout: {e}")


# === Retry with Exponential Backoff ===
def create_retry_session(
    retries: int = 3,
    backoff_factor: float = 0.5,
    status_forcelist: tuple = (429, 500, 502, 503, 504)
) -> requests.Session:
    """สร้าง Session พร้อม Retry logic"""
    
    session = requests.Session()
    
    retry_strategy = Retry(
        total=retries,
        read=retries,
        connect=retries,
        backoff_factor=backoff_factor,
        # Retry เมื่อได้รับ status codes เหล่านี้
        status_forcelist=status_forcelist,
        # Retry เฉพาะ idempotent methods
        allowed_methods=frozenset(["HEAD", "GET", "PUT", "DELETE", "OPTIONS", "TRACE"]),
        # Raise exception หลัง retry หมด
        raise_on_status=False
    )
    
    adapter = HTTPAdapter(
        max_retries=retry_strategy,
        pool_connections=10,  # Connection pool size
        pool_maxsize=20
    )
    
    session.mount("https://", adapter)
    session.mount("http://", adapter)
    
    return session


# === Custom Retry Logic ===
class APIClient:
    """API Client พร้อม retry และ error handling"""
    
    def __init__(
        self,
        base_url: str,
        api_key: str,
        max_retries: int = 3,
        timeout: int = 30
    ):
        self.base_url = base_url.rstrip("/")
        self.max_retries = max_retries
        self.timeout = timeout
        
        self._session = create_retry_session(retries=max_retries)
        self._session.headers.update({
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
            "Accept": "application/json",
        })
    
    def get(self, path: str, params: dict = None) -> dict:
        return self._request("GET", path, params=params)
    
    def post(self, path: str, data: dict = None) -> dict:
        return self._request("POST", path, json=data)
    
    def put(self, path: str, data: dict = None) -> dict:
        return self._request("PUT", path, json=data)
    
    def delete(self, path: str) -> bool:
        response = self._request("DELETE", path, raw=True)
        return response.status_code in (200, 204)
    
    def _request(
        self, 
        method: str, 
        path: str, 
        raw: bool = False,
        **kwargs
    ):
        url = f"{self.base_url}/{path.lstrip('/')}"
        
        try:
            response = self._session.request(
                method=method,
                url=url,
                timeout=self.timeout,
                **kwargs
            )
            
            if raw:
                return response
            
            return self._handle_response(response)
            
        except requests.exceptions.ConnectionError as e:
            raise APIConnectionError(f"Connection failed: {e}")
        except requests.exceptions.Timeout:
            raise APITimeoutError(f"Request timed out after {self.timeout}s")
    
    def _handle_response(self, response: requests.Response) -> dict:
        """Handle response และ raise errors ที่เหมาะสม"""
        
        if response.status_code == 200:
            return response.json()
        elif response.status_code == 201:
            return response.json()
        elif response.status_code == 204:
            return {}
        elif response.status_code == 400:
            error_data = response.json()
            raise APIValidationError(f"Validation error: {error_data}")
        elif response.status_code == 401:
            raise APIAuthError("Unauthorized - check your API key")
        elif response.status_code == 403:
            raise APIAuthError("Forbidden - insufficient permissions")
        elif response.status_code == 404:
            raise APINotFoundError(f"Resource not found: {response.url}")
        elif response.status_code == 429:
            retry_after = response.headers.get("Retry-After", 60)
            raise APIRateLimitError(f"Rate limit exceeded, retry after {retry_after}s")
        elif response.status_code >= 500:
            raise APIServerError(f"Server error: {response.status_code}")
        else:
            response.raise_for_status()
            return response.json()
    
    def __enter__(self):
        return self
    
    def __exit__(self, *args):
        self._session.close()


# Custom Exceptions
class APIError(Exception): pass
class APIConnectionError(APIError): pass
class APITimeoutError(APIError): pass
class APIValidationError(APIError): pass
class APIAuthError(APIError): pass
class APINotFoundError(APIError): pass
class APIRateLimitError(APIError): pass
class APIServerError(APIError): pass


# ทดสอบ API Client
def demo_api_client():
    # ใช้ context manager
    with APIClient(
        base_url="https://jsonplaceholder.typicode.com",
        api_key="test-key",
        timeout=10
    ) as client:
        
        try:
            # GET
            post = client.get("/posts/1")
            print(f"Post title: {post['title'][:50]}")
            
            # POST
            new_post = client.post("/posts", data={
                "title": "New Post",
                "body": "Content here",
                "userId": 1
            })
            print(f"Created post ID: {new_post['id']}")
            
            # Error handling
            try:
                client.get("/posts/999999")
            except APINotFoundError as e:
                print(f"Not found (expected): {e}")
                
        except APIError as e:
            print(f"API Error: {e}")


demo_api_client()
```

---

## 6. Handling Responses

```python
import requests

# === Response Object ===
def demo_response_handling():
    response = requests.get("https://httpbin.org/get")
    
    # Status code
    print(f"Status code: {response.status_code}")
    print(f"OK: {response.ok}")          # True ถ้า status < 400
    print(f"Reason: {response.reason}")  # "OK", "Not Found", etc.
    
    # Headers
    print(f"\nHeaders:")
    print(f"  Content-Type: {response.headers['Content-Type']}")
    print(f"  Content-Length: {response.headers.get('Content-Length', 'N/A')}")
    print(f"  Server: {response.headers.get('Server')}")
    
    # Body
    print(f"\nBody:")
    print(f"  Text length: {len(response.text)} chars")
    print(f"  Bytes length: {len(response.content)} bytes")
    print(f"  Encoding: {response.encoding}")
    
    # JSON
    data = response.json()
    print(f"\nJSON keys: {list(data.keys())}")
    
    # Elapsed time
    print(f"\nElapsed: {response.elapsed.total_seconds():.3f}s")
    
    # Request info
    print(f"Request URL: {response.request.url}")
    print(f"Request method: {response.request.method}")


# === Streaming Large Responses ===
def demo_streaming():
    """Download large files ด้วย streaming"""
    
    url = "https://httpbin.org/bytes/1024"  # Download 1KB
    
    with requests.get(url, stream=True) as response:
        response.raise_for_status()
        
        total_size = int(response.headers.get("Content-Length", 0))
        downloaded = 0
        
        chunks = []
        for chunk in response.iter_content(chunk_size=256):
            if chunk:
                chunks.append(chunk)
                downloaded += len(chunk)
                progress = (downloaded / total_size * 100) if total_size else 0
                print(f"Progress: {progress:.1f}% ({downloaded}/{total_size} bytes)")
        
        data = b"".join(chunks)
        print(f"Total downloaded: {len(data)} bytes")


# === Response Validation ===
def validate_response(response: requests.Response) -> bool:
    """Validate response structure"""
    
    # Check status
    if not response.ok:
        print(f"HTTP Error: {response.status_code}")
        return False
    
    # Check content type
    content_type = response.headers.get("Content-Type", "")
    if "application/json" not in content_type:
        print(f"Unexpected content type: {content_type}")
        return False
    
    # Check body
    try:
        data = response.json()
    except ValueError:
        print("Invalid JSON response")
        return False
    
    # Check required fields
    # required_fields = ["id", "name", "email"]
    # missing = [f for f in required_fields if f not in data]
    # if missing:
    #     print(f"Missing fields: {missing}")
    #     return False
    
    return True


demo_response_handling()
demo_streaming()
```

---

## 7. Async HTTP ด้วย aiohttp

```python
import asyncio
import aiohttp
from typing import List

# === Basic Async HTTP ===
async def fetch_post(session: aiohttp.ClientSession, post_id: int) -> dict:
    """Async GET request"""
    url = f"https://jsonplaceholder.typicode.com/posts/{post_id}"
    
    async with session.get(url) as response:
        response.raise_for_status()
        return await response.json()


async def fetch_all_posts(post_ids: List[int]) -> List[dict]:
    """Fetch หลาย posts พร้อมกัน"""
    
    async with aiohttp.ClientSession() as session:
        # สร้าง tasks ทั้งหมดพร้อมกัน
        tasks = [fetch_post(session, pid) for pid in post_ids]
        
        # รันทั้งหมดพร้อมกัน
        posts = await asyncio.gather(*tasks, return_exceptions=True)
        
        # Filter out errors
        successful = [p for p in posts if not isinstance(p, Exception)]
        errors = [p for p in posts if isinstance(p, Exception)]
        
        if errors:
            print(f"Errors: {len(errors)}")
        
        return successful


# === Async with Rate Limiting ===
async def fetch_with_rate_limit(
    urls: List[str], 
    max_concurrent: int = 5
) -> List[dict]:
    """จำกัด concurrent requests ด้วย Semaphore"""
    
    semaphore = asyncio.Semaphore(max_concurrent)
    results = []
    
    async def fetch_one(session, url):
        async with semaphore:  # จำกัดไม่ให้เกิน max_concurrent พร้อมกัน
            try:
                async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as r:
                    return await r.json()
            except Exception as e:
                print(f"Error: {url}: {e}")
                return None
    
    connector = aiohttp.TCPConnector(limit=100)  # Connection pool
    timeout = aiohttp.ClientTimeout(total=60)
    
    async with aiohttp.ClientSession(
        connector=connector, 
        timeout=timeout
    ) as session:
        tasks = [fetch_one(session, url) for url in urls]
        results = await asyncio.gather(*tasks)
    
    return [r for r in results if r is not None]


# Run async functions
async def main():
    print("Fetching posts asynchronously...")
    
    start = __import__("time").time()
    posts = await fetch_all_posts(list(range(1, 11)))  # Fetch 10 posts
    elapsed = __import__("time").time() - start
    
    print(f"Fetched {len(posts)} posts in {elapsed:.2f}s")
    for post in posts[:3]:
        print(f"  Post {post['id']}: {post['title'][:40]}")


# asyncio.run(main())  # Uncomment เพื่อรัน
print("Async HTTP demo ready (uncomment asyncio.run to execute)")
```

---

## 8. Testing HTTP Calls

```python
import requests
from unittest.mock import patch, Mock, MagicMock
import pytest

# === ใช้ responses library ===
# pip install responses

# === Manual Mock ===
def get_user(user_id: int) -> dict:
    """Function ที่เราต้องการ test"""
    response = requests.get(f"https://api.example.com/users/{user_id}")
    response.raise_for_status()
    return response.json()


def fetch_weather(city: str) -> dict:
    response = requests.get(
        "https://api.weather.com/v1/weather",
        params={"city": city, "units": "metric"}
    )
    if response.status_code == 404:
        raise ValueError(f"City not found: {city}")
    response.raise_for_status()
    return response.json()


# Test ด้วย unittest.mock
class TestHTTPCalls:
    """Tests สำหรับ HTTP calls"""
    
    def test_get_user_success(self):
        """Test successful user fetch"""
        mock_user_data = {"id": 1, "name": "Alice", "email": "alice@example.com"}
        
        with patch("requests.get") as mock_get:
            # ตั้ง return value
            mock_response = Mock()
            mock_response.status_code = 200
            mock_response.json.return_value = mock_user_data
            mock_response.raise_for_status = Mock()  # ไม่ throw exception
            mock_get.return_value = mock_response
            
            result = get_user(1)
            
            # Assertions
            assert result == mock_user_data
            mock_get.assert_called_once_with(
                "https://api.example.com/users/1"
            )
    
    def test_get_user_not_found(self):
        """Test 404 response"""
        with patch("requests.get") as mock_get:
            mock_response = Mock()
            mock_response.status_code = 404
            mock_response.raise_for_status.side_effect = requests.HTTPError(
                "404 Not Found"
            )
            mock_get.return_value = mock_response
            
            try:
                get_user(999)
                assert False, "Should have raised"
            except requests.HTTPError:
                pass  # Expected
    
    def test_fetch_weather_success(self):
        """Test weather API"""
        mock_weather = {"temperature": 30, "humidity": 80, "city": "Bangkok"}
        
        with patch("requests.get") as mock_get:
            mock_response = Mock()
            mock_response.status_code = 200
            mock_response.json.return_value = mock_weather
            mock_response.raise_for_status = Mock()
            mock_get.return_value = mock_response
            
            result = fetch_weather("Bangkok")
            
            assert result["city"] == "Bangkok"
            assert result["temperature"] == 30
            
            # ตรวจสอบว่าส่ง params ถูกต้อง
            mock_get.assert_called_with(
                "https://api.weather.com/v1/weather",
                params={"city": "Bangkok", "units": "metric"}
            )
    
    def test_fetch_weather_city_not_found(self):
        """Test city not found"""
        with patch("requests.get") as mock_get:
            mock_response = Mock()
            mock_response.status_code = 404
            mock_get.return_value = mock_response
            
            try:
                fetch_weather("NonExistentCity")
                assert False, "Should have raised ValueError"
            except ValueError as e:
                assert "NonExistentCity" in str(e)


# Run tests manually
def run_tests():
    test = TestHTTPCalls()
    
    print("Running tests...")
    try:
        test.test_get_user_success()
        print("✅ test_get_user_success passed")
    except AssertionError as e:
        print(f"❌ test_get_user_success failed: {e}")
    
    try:
        test.test_get_user_not_found()
        print("✅ test_get_user_not_found passed")
    except AssertionError as e:
        print(f"❌ test_get_user_not_found failed: {e}")
    
    try:
        test.test_fetch_weather_success()
        print("✅ test_fetch_weather_success passed")
    except AssertionError as e:
        print(f"❌ test_fetch_weather_success failed: {e}")
    
    try:
        test.test_fetch_weather_city_not_found()
        print("✅ test_fetch_weather_city_not_found passed")
    except AssertionError as e:
        print(f"❌ test_fetch_weather_city_not_found failed: {e}")


run_tests()
```

---

## 9. สรุป Part 040

✅ **HTTP Methods** - GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS  
✅ **Headers** - Custom headers, Content negotiation  
✅ **Authentication** - Basic, Bearer, API Key, OAuth 2.0, Custom  
✅ **Sessions** - Connection pooling, persistent cookies  
✅ **Cookies** - Manual management, saving/loading  
✅ **Timeout** - Connection timeout, read timeout  
✅ **Retry** - Exponential backoff, custom retry logic  
✅ **Response Handling** - Status codes, streaming, validation  
✅ **Async HTTP** - aiohttp สำหรับ concurrent requests  
✅ **Testing** - Mock HTTP calls ใน unit tests  

## ➡️ ถัดไป: Part 041 - Environment Configuration
*Part 040/100+ | Python Course - Beginner to World-Class*
