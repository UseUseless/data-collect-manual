# Инструмент 1: "Чистый выстрел" (Direct API)

**Стек:** `httpx` (Async), `Pydantic`, `Asyncio`.
**Суть:** Максимальная скорость. Мы имитируем запрос фронтенда к бэкенду. Никакого HTML.

```python
import asyncio
import httpx
from pydantic import BaseModel, Field, ValidationError
from typing import List, Optional

# --- 1. CONFIG (Настройки) ---
# Сюда вставляем заголовки, которые мы украли из Network Tab (Copy as cURL -> Python)
HEADERS = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 ...",
    "Accept": "application/json",
    # "Authorization": "Bearer eyJ...", # Если нужен токен
    # "Referer": "<https://site.com/>",   # Часто обязательно
}

BASE_API_URL = "<https://api.site.com/v2/catalog/products>"

# --- 2. DATA MODEL (Валидация) ---
# Описываем данные, которые хотим получить. Грязь отсеется сама.
class ProductItem(BaseModel):
    id: int
    title: str = Field(alias="name") # Если в JSON ключ "name", а мы хотим "title"
    price: float
    url_suffix: str = Field(alias="slug")
    is_available: bool = True

    # Можно добавить валидатор для URL
    @property
    def full_url(self):
        return f"<https://site.com/product/{self.url_suffix}>"

class APIResponse(BaseModel):
    items: List[ProductItem]
    total_pages: int
    # cursor: Optional[str] = None # Если пагинация через курсор

# --- 3. PARSER CLASS ---
class ApiParser:
    def __init__(self):
        # Включаем http2=True, чтобы быть похожими на браузер
        self.client = httpx.AsyncClient(headers=HEADERS, http2=True, timeout=10.0)

    async def fetch_page(self, page_num: int) -> Optional[APIResponse]:
        """Делает запрос и возвращает валидированный объект"""
        params = {
            "page": page_num,
            "limit": 20,
            "category_id": 123
        }

        try:
            response = await self.client.get(BASE_API_URL, params=params)
            response.raise_for_status() # Если 403 или 500 - выкинет ошибку

            # Магия Pydantic: сырой JSON превращаем в объект
            return APIResponse(**response.json())

        except httpx.HTTPStatusError as e:
            print(f"🔴 Ошибка сети: {e.response.status_code}")
            if e.response.status_code == 403:
                print("⛔ Нас забанили (WAF). Проверь заголовки или смени IP.")
        except ValidationError as e:
            print(f"⚠️ API изменилось! Данные не подходят под модель: {e}")
        except Exception as e:
            print(f"💀 Неизвестная ошибка: {e}")

        return None

    async def close(self):
        await self.client.aclose()

    async def run(self):
        page = 1
        max_pages = 5 # Заглушка, реально узнаем из первого ответа

        while page <= max_pages:
            print(f"🚀 Парсим страницу {page}...")

            data = await self.fetch_page(page)

            if not data:
                break # Ошибка или конец

            # Обновляем инфу о страницах (если API это отдает)
            if page == 1:
                max_pages = data.total_pages

            # --- СОХРАНЕНИЕ ---
            # Тут вызываем наш класс Saver (из предыдущих уроков)
            for item in data.items:
                # print(f"✅ Товар: {item.title} | {item.price}")
                pass # save_to_db(item)

            page += 1
            # Не DDOS-им!
            await asyncio.sleep(0.5)

# --- 4. ENTRY POINT ---
async def main():
    parser = ApiParser()
    try:
        await parser.run()
    finally:
        await parser.close()

if __name__ == "__main__":
    asyncio.run(main())

```

**Куда смотреть:**

1. `HEADERS`: Это 90% успеха. Если не работает — значит, ты не все заголовки скопировал.
2. `ProductItem`: Меняй поля под свой JSON.
3. `params`: В методе `fetch_page` настраивай пагинацию (`offset`, `cursor`, `page`).