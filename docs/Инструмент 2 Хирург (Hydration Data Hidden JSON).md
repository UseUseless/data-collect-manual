# Инструмент 2: "Хирург" (Hydration Data / Hidden JSON)

**Стек:** `httpx`, `Selectolax` (для поиска скрипта), `json`, `Pydantic`.
**Суть:** Мы не парсим верстку. Мы ищем тег `<script>`, где лежит состояние страницы (State), вырезаем его и работаем как с чистым API.

**Где применять:** Большинство современных магазинов (Nike, Adidas, Walmart), сайты на Next.js / Nuxt.js.

```python
import asyncio
import httpx
import json
from selectolax.parser import HTMLParser
from pydantic import BaseModel
from typing import List, Dict, Any

# --- 1. MODELS ---
class Product(BaseModel):
    id: str
    name: str
    price: float
    stock_status: str

# --- 2. PARSER CLASS ---
class HydrationParser:
    def __init__(self):
        self.headers = {
            "User-Agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)..."
        }
        self.client = httpx.AsyncClient(headers=self.headers, http2=True)

    def extract_hidden_json(self, html: str) -> Dict[str, Any]:
        """
        Хирургическая операция. Ищем скрипт с данными.
        Частые ID: __NEXT_DATA__ (Next.js), __NUXT__ (Nuxt), initial-state
        """
        tree = HTMLParser(html)

        # 1. Попытка для Next.js (самый частый кейс)
        script_node = tree.css_first("script#__NEXT_DATA__")

        if script_node:
            try:
                # Внутри лежит валидный JSON. Просто парсим.
                return json.loads(script_node.text())
            except json.JSONDecodeError:
                print("💀 JSON внутри скрипта битый!")
                return {}

        # 2. Если не нашли - можно искать Nuxt или через Regex (для window.App = ...)
        print("⚠️ Скрытый контейнер данных не найден. Возможно, это не Next.js.")
        return {}

    async def fetch_product_page(self, url: str):
        print(f"🔎 Скачиваем HTML: {url}")
        resp = await self.client.get(url)

        if resp.status_code != 200:
            print(f"Ошибка сети: {resp.status_code}")
            return

        # --- ЭТАП ХИРУРГИИ ---
        data = self.extract_hidden_json(resp.text)

        if not data:
            return

        # --- ЭТАП НАВИГАЦИИ ПО JSON (Самое сложное) ---
        # Структура Next.js всегда адски вложенная.
        # Придется один раз открыть JSON Viewer и найти путь.
        try:
            # Пример типичного пути в Next.js:
            props = data.get("props", {}).get("pageProps", {})
            initial_state = props.get("initialState", {})
            products_raw = initial_state.get("products", {}).get("list", [])

            # --- ВАЛИДАЦИЯ И СОХРАНЕНИЕ ---
            for item in products_raw:
                # Преобразуем грязный словарь в чистую модель
                product = Product(
                    id=str(item.get("productId")),
                    name=item.get("displayName"),
                    price=float(item.get("currentPrice", 0)),
                    stock_status="IN_STOCK" if item.get("available") else "OUT"
                )
                print(f"✅ Успех: {product.name} — {product.price}")
                # await saver.save(product)

        except Exception as e:
            print(f"💥 Структура JSON изменилась: {e}")

    async def close(self):
        await self.client.aclose()

# --- 3. ENTRY ---
async def main():
    parser = HydrationParser()
    try:
        # Пример: страница категории или товара
        await parser.fetch_product_page("<https://example-shop.com/shoes>")
    finally:
        await parser.close()

if __name__ == "__main__":
    asyncio.run(main())

```

**Куда смотреть:**

1. `tree.css_first("script#__NEXT_DATA__")`: Это ключ к победе. Если сайт на Nuxt, ищи `script` где текст начинается с `window.__NUXT__` (тут понадобится библиотека `chompjs`, про которую я говорил в теории).
2. **Путь к данным:** Строки типа `data['props']['pageProps']...` тебе придется подобрать один раз вручную. Открой сайт, нажми `Ctrl+U`, найди JSON, скопируй в любой онлайн JSON Formatter и найди, где лежит список товаров.
3. **Преимущество:** Мы получаем данные, которые даже не отрисованы на экране (например, `stock_level` или `internal_id`).