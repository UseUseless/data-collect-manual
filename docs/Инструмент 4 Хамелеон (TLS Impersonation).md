# Инструмент 4: "Хамелеон" (TLS Impersonation)

**Стек:** `curl_cffi` (Async), `Pydantic`.
**Суть:** Обычный `httpx` палится на этапе "рукопожатия" (SSL Handshake), даже если заголовки идеальные. `curl_cffi` подменяет отпечаток (JA3 fingerprint) на отпечаток реального Хрома.
**Где применять:** Avito, Cloudflare-protected сайты, сайты с защитой Akamai/Datadome. Если `httpx` дает 403, а в браузере сайт работает — бери этот шаблон.

**Внимание:** Для этого нужна библиотека: `pip install curl_cffi`

```python
import asyncio
from curl_cffi.requests import AsyncSession, RequestsError
from pydantic import BaseModel
from typing import Optional

# --- 1. CONFIG ---
# Заголовки берем из браузера. Они должны соответствовать версии браузера в impersonate!
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36",
    "Accept": "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8",
    "Accept-Language": "en-US,en;q=0.9",
}

# --- 2. PARSER CLASS ---
class TlsKiller:
    def __init__(self, proxy_url: Optional[str] = None):
        self.proxy = proxy_url
        # Создаем сессию один раз. Она будет хранить Cookies.
        # ВАЖНО: impersonate="chrome120" — мы притворяемся Хромом 120 версии.
        self.session = AsyncSession(
            impersonate="chrome120",
            headers=HEADERS,
            proxies={"http": self.proxy, "https": self.proxy} if self.proxy else None
        )

    async def get_protected_json(self, url: str):
        """Пример получения JSON с защищенного API"""
        print(f"🕵️ Пробуем пробить защиту: {url}")

        try:
            # Запрос выглядит как requests, но под капотом магия C-кода
            response = await self.session.get(url)

            if response.status_code == 200:
                print("✅ Пробито! 200 OK")
                # print(response.json()) # Если ждем JSON
                return response

            elif response.status_code == 403:
                print("⛔ Cloudflare всё равно не пускает.")
                print("Совет: Проверь IP прокси или смени версию impersonate (на chrome110, safari15_3)")

            else:
                print(f"⚠️ Странный статус: {response.status_code}")

        except RequestsError as e:
            print(f"💀 Ошибка сети (возможно, прокси умер): {e}")

    async def close(self):
        # Обязательно закрываем сессию
        self.session.close() # curl_cffi требует явного закрытия (в новых версиях мб авто)

# --- 3. ENTRY ---
async def main():
    # Без хороших прокси Cloudflare может забанить сам IP, даже с правильным TLS
    # proxy = "<http://user:pass@ip>:port"
    proxy = None

    parser = TlsKiller(proxy)
    try:
        # Тестовый сайт, который показывает твой TLS Fingerprint (JA3)
        await parser.get_protected_json("<https://tls.browserleaks.com/json>")
    finally:
        await parser.close()

if __name__ == "__main__":
    asyncio.run(main())

```

**Куда смотреть:**

1. **`impersonate="chrome120"`**: Это самый главный параметр. Если сайт не пускает, попробуй поменять на `chrome110`, `safari15_3` или `edge101`. Список доступных версий есть в документации `curl_cffi`.
2. **`AsyncSession`**: Не перепутай с `httpx.AsyncClient`. У них похожий интерфейс, но разные внутренности.
3. **Прокси:** Синтаксис `proxies` тут такой же, как в `requests` (словарь `{"http": ..., "https": ...}`).
4. **Cookie:** Эта сессия сама сохраняет куки. Если ты пробил Cloudflare Challenge (вас перенаправило), сессия запомнит `cf_clearance` куку и дальше будет пускать без проблем.