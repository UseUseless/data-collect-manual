# Инструмент 3: "Классика" (Static HTML Parsing)

**Стек:** `httpx` (Async), `Selectolax` (скорость), `Pydantic`.
**Суть:** Старый добрый парсинг DOM. Скачиваем HTML, строим дерево, ищем элементы по CSS-селекторам.
**Важно:** Я использую `Selectolax`, а не `BeautifulSoup`, потому что на 100 000 страниц BS4 сожрет всю память и процессор. Selectolax быстрее в 20 раз.

**Где применять:** Блоги, новостные сайты, старые интернет-магазины, Википедия.

```python
import asyncio
import httpx
from selectolax.lexbor import LexborHTMLParser # Lexbor круче Modern, он прощает ошибки HTML
from pydantic import BaseModel, HttpUrl, field_validator
from typing import Optional

# --- 1. DATA MODEL ---
class Article(BaseModel):
    title: str
    link: HttpUrl
    author: str = "Unknown" # Значение по умолчанию
    tags: list[str] = []

    # Чистим данные на лету
    @field_validator('title')
    def clean_title(cls, v):
        return v.strip().replace("\\n", " ")

# --- 2. PARSER CLASS ---
class HtmlParserWorker:
    def __init__(self):
        self.headers = {
            # Всегда свежий User-Agent
            "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) ..."
        }
        self.client = httpx.AsyncClient(headers=self.headers, http2=True, follow_redirects=True)

    def parse_html(self, html_content: str, base_url: str) -> list[Article]:
        """Парсинг DOM дерева (CPU bound операция)"""
        tree = LexborHTMLParser(html_content)
        results = []

        # 1. Ищем контейнеры (карточки товаров/статей)
        # Пример: <div class="article-card">...</div>
        cards = tree.css("div.article-card")

        for card in cards:
            try:
                # 2. Ищем элементы ВНУТРИ карточки
                # Используем css_first (вернет первый найденный или None)

                # Заголовок
                title_node = card.css_first("h2.title a")
                if not title_node:
                    continue # Битая карточка, пропускаем

                title_text = title_node.text(strip=True)

                # Ссылка (вытаскиваем атрибут href)
                link_href = title_node.attributes.get("href")
                if link_href.startswith("/"):
                    link_href = base_url + link_href # Делаем абсолютную ссылку

                # Автор (может не быть)
                author_node = card.css_first("span.author-name")
                author_text = author_node.text(strip=True) if author_node else "Unknown"

                # Теги (список)
                tags = [t.text(strip=True) for t in card.css("ul.tags li")]

                # 3. Валидация через Pydantic
                article = Article(
                    title=title_text,
                    link=link_href,
                    author=author_text,
                    tags=tags
                )
                results.append(article)

            except Exception as e:
                print(f"⚠️ Ошибка парсинга одной карточки: {e}")
                # Не крашим весь процесс из-за одной ошибки

        return results

    async def run(self, url: str):
        print(f"📥 Качаем: {url}")
        try:
            resp = await self.client.get(url)

            if resp.status_code == 200:
                # Парсинг быстрый, но если страниц много - лучше вынести в ThreadPool
                articles = self.parse_html(resp.text, "<https://example-blog.com>")

                print(f"✅ Найдено {len(articles)} статей.")
                for a in articles:
                    print(f"   - {a.title} ({a.link})")
            else:
                print(f"⛔ Ошибка сервера: {resp.status_code}")

        except Exception as e:
            print(f"💀 Сетевая ошибка: {e}")

    async def close(self):
        await self.client.aclose()

# --- 3. ENTRY ---
async def main():
    parser = HtmlParserWorker()
    try:
        await parser.run("<https://example-blog.com/news>")
    finally:
        await parser.close()

if __name__ == "__main__":
    asyncio.run(main())

```

**Куда смотреть:**

1. **`LexborHTMLParser`**: Это движок. Если не ставится, используй `from selectolax.parser import HTMLParser` (это движок Modest). Lexbor чуть лучше понимает кривой HTML.
2. **`.css("selector")`**: Возвращает список элементов.
3. **`.css_first("selector")`**: Возвращает один элемент или `None`. Всегда проверяй на `None` перед тем как брать `.text()`, иначе получишь `AttributeError`.
4. **`attributes.get("href")`**: Так достают ссылки и картинки (`src`).