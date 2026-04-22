# Selenium -- Browser Automation Skill

**Only use Selenium when requests + API discovery fail.** Browser automation is 10-50x slower and more fragile than direct HTTP.

## Decision Tree

1. **Always try requests first.** Check response for embedded JSON, API endpoints in HTML source, JSON-LD.
2. **If JS rendering is required**, check for a mobile API or embedded `__NEXT_DATA__` / `__PRELOADED_STATE__` before reaching for Selenium.
3. **If all else fails**, Selenium headless Chrome.

## Setup

```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options

def get_driver(headless: bool = True) -> webdriver.Chrome:
    opts = Options()
    if headless:
        opts.add_argument("--headless=new")
    opts.add_argument("--disable-blink-features=AutomationControlled")
    opts.add_argument("--disable-gpu")
    opts.add_argument("--no-sandbox")
    opts.add_argument("--window-size=1920,1080")
    opts.add_argument("user-agent=Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36")
    # Do NOT use excludeSwitches: ["enable-automation"] -- crashes recent Chrome versions
    opts.add_experimental_option("useAutomationExtension", False)
    driver = webdriver.Chrome(options=opts)
    # CDP webdriver override (hides navigator.webdriver)
    driver.execute_cdp_cmd("Page.addScriptToEvaluateOnNewDocument", {
        "source": "Object.defineProperty(navigator, 'webdriver', {get: () => undefined})"
    })
    return driver
```

## Dynamic Waits (not sleep)

```python
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

def wait_for_element(driver, selector: str, timeout: int = 10):
    return WebDriverWait(driver, timeout).until(
        EC.presence_of_element_located((By.CSS_SELECTOR, selector))
    )

def wait_for_text(driver, selector: str, text: str, timeout: int = 10):
    return WebDriverWait(driver, timeout).until(
        EC.text_to_be_present_in_element((By.CSS_SELECTOR, selector), text)
    )
```

## Scroll to Load (infinite scroll)

```python
import time

def scroll_to_load(driver, pause: float = 1.5, max_scrolls: int = 50):
    prev = 0
    for _ in range(max_scrolls):
        curr = driver.execute_script("return document.body.scrollHeight")
        if curr == prev:
            break
        prev = curr
        driver.execute_script("window.scrollTo(0, document.body.scrollHeight)")
        time.sleep(pause)
```

## API Discovery via HAR Capture

Often the fastest way to understand a JS-heavy site is to let Selenium load it, then inspect the network traffic it made. Once you have the API endpoint, switch to requests and skip Selenium entirely.

```python
from selenium.webdriver.common.devtools.v124 import network, fetch
# (or use selenium-wire for simpler network capture)
```

If selenium-wire is available, it's much simpler:

```python
from seleniumwire import webdriver

driver = webdriver.Chrome()
driver.get("https://target-site.com/search")
# ... interact
for request in driver.requests:
    if "api" in request.url:
        print(request.url, request.method, request.response.body[:200])
```

## Cookie Transfer to requests

After Selenium gets past an anti-bot challenge, you can transfer cookies to a requests session and finish the scrape in pure HTTP:

```python
import requests

def selenium_to_requests(driver) -> requests.Session:
    session = requests.Session()
    for cookie in driver.get_cookies():
        session.cookies.set(cookie['name'], cookie['value'], domain=cookie.get('domain'))
    session.headers.update({
        "User-Agent": driver.execute_script("return navigator.userAgent"),
    })
    return session
```

## Form Interaction

```python
from selenium.webdriver.common.keys import Keys

def fill_form(driver, fields: dict):
    for selector, value in fields.items():
        el = wait_for_element(driver, selector)
        el.clear()
        el.send_keys(value)
```

## Troubleshooting

- **Element not found**: usually a timing issue. Use `WebDriverWait`, not `sleep`.
- **Stale element reference**: the DOM re-rendered. Re-find the element.
- **Driver crashes on startup**: incompatible chromedriver version (Selenium 4 usually auto-manages).
- **Detected as automation**: use the setup above with `--disable-blink-features=AutomationControlled` and CDP override.
- **Slow on many pages**: consider if this should actually be requests-only. Selenium is always the fallback, not the first choice.
