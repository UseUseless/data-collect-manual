# Инструмент 5: "Танк" (Headless Browser Automation)

**Стек:** `Playwright` (Async), `Selectolax` (Гибридный подход).
**Суть:** Запускаем реальный движок Chromium (без окна). Он выполняет весь JS, рендерит React/Vue компоненты.
**Главная фишка архитектора:** Мы используем браузер **ТОЛЬКО** для рендеринга и действий (клик, скролл). Как только контент появился, мы забираем HTML (`page.content()`) и парсим его через `Selectolax`. Это в 100 раз быстрее, чем спрашивать текст каждого элемента через API браузера.

**Где применять:** Сложные SPA, сайты с бесконечным скроллом, сайты, где контент появляется только после клика.

**Установка:**`pip install playwright selectolaxplaywright install chromium`

```python
import asyncio
from playwright.async_api import async_playwright, Page, BrowserContext
from selectolax.parser import HTMLParser
from typing import List

# --- 1. CONFIG ---
# Блокируем мусор, чтобы ускорить загрузку в 5 раз
BLOCKED_RESOURCES = [".png", ".jpg", ".jpeg", ".gif", ".svg", ".css", ".woff", ".woff2"]

# --- 2. PARSER CLASS ---
class BrowserWorker:
    def __init__(self):
        self.playwright = None
        self.browser = None

    async def start(self):
        """Запуск тяжелого процесса (делаем 1 раз)"""
        self.playwright = await async_playwright().start()
        # headless=False если хочешь видеть глазами, что происходит
        self.browser = await self.playwright.chromium.launch(headless=True)

    async def stop(self):
        if self.browser:
            await self.browser.close()
        if self.playwright:
            await self.playwright.stop()

    async def _configure_page(self, context: BrowserContext) -> Page:
        """Настройка страницы: перехват картинок и таймауты"""
        page = await context.new_page()

        # Блокировка картинок и шрифтов (Экономия трафика и памяти)
        async def route_handler(route):
            if any(ext in route.request.url for ext in BLOCKED_RESOURCES):
                await route.abort()
            else:
                await route.continue_()

        await page.route("**/*", route_handler)
        return page

    async def parse_dynamic_page(self, url: str):
        print(f"🚜 Танк выехал на: {url}")

        # Создаем изолированный контекст (как инкогнито вкладка)
        # Тут можно задать user_agent, viewport, proxy
        context = await self.browser.new_context(
            user_agent="Mozilla/5.0 (Windows NT 10.0; Win64; x64)..."
        )

        page = await self._configure_page(context)

        try:
            # 1. Переход
            await page.goto(url, wait_until="domcontentloaded", timeout=30000)

            # 2. Действия (Логика "Человека")
            # Например, ждем появления списка товаров
            # Селектор должен быть уникальным для контента
            try:
                await page.wait_for_selector("div.product-card", timeout=10000)
            except Exception:
                print("⚠️ Товары не появились. Возможно, сайт лагает или защита.")
                return

            # Пример: Скроллим вниз, чтобы подгрузить Lazy Load
            for _ in range(3):
                await page.mouse.wheel(0, 5000)
                await asyncio.sleep(1) # Даем JS время на подгрузку

            # 3. ГИБРИДНЫЙ ПАРСИНГ (Самый сок)
            # Не перебираем элементы через await page.locator(...). Это медленно!
            # Забираем весь HTML строкой и отдаем Selectolax.
            html = await page.content()
            self._parse_with_selectolax(html)

        except Exception as e:
            print(f"💀 Авария танка: {e}")
        finally:
            # Обязательно закрываем контекст, чтобы очистить RAM
            await context.close()

    def _parse_with_selectolax(self, html: str):
        """Быстрый парсинг статики"""
        tree = HTMLParser(html)
        products = tree.css("div.product-card")
        print(f"✅ Найдено элементов: {len(products)}")

        for node in products:
            title = node.css_first(".title").text(strip=True)
            print(f"   - {title}")

# --- 3. ENTRY ---
async def main():
    worker = BrowserWorker()
    await worker.start()
    try:
        await worker.parse_dynamic_page("<https://example-spa-shop.com>")
    finally:
        await worker.stop()

if __name__ == "__main__":
    asyncio.run(main())

```

**Куда смотреть:**

1. **`BLOCKED_RESOURCES` + `page.route`**: Это критически важно. Без этого Playwright будет качать рекламные баннеры по 5 Мб. Твой парсер ускорится в 3-5 раз благодаря этим строкам.
2. **`wait_until="domcontentloaded"`**: Мы не ждем полной загрузки (`networkidle`), иначе будем висеть вечно из-за какой-нибудь метрики. Ждем только DOM, а нужные элементы ждем через `wait_for_selector`.
3. **Гибрид:** Обрати внимание, я не использую методы Playwright для извлечения текста. Я забираю HTML и паршу его локально. Это **Best Practice** для высокой производительности.

Жду команду **"Дальше"** (осталось 2 варианта: Стелс и Вебсокеты).