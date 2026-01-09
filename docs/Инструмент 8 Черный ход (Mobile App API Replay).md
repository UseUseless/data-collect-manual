# Инструмент 8: "Черный ход" (Mobile App API Replay)

**Стек:** `httpx` (Async), `hashlib` (для подписи), `uuid`.
**Суть:** Ты перехватил запрос телефоном (через Charles/Mitmproxy). Ты увидел, что приложение шлет JSON, но там есть странные заголовки типа `X-Signature`, `timestamp`, `nonce`.
**Твоя задача:** Повторить алгоритм генерации этих заголовков на Python, чтобы сервер думал, что ты — официальное приложение на Айфоне.

**Где применять:** Доставки еды, Такси, Маркетплейсы (Ozon/WB мобильные версии), Банки (аккуратно!).

**Предварительные требования:**

1. Ты уже перехватил запрос и знаешь URL и структуру.
2. Если есть подпись (Signature), ты декомпилировал APK (через `jadx-gui`) и нашел "Секретный ключ" (Salt/Secret), которым подписываются запросы.

```python
import asyncio
import httpx
import time
import hashlib
import uuid
from urllib.parse import urlencode

# --- 1. CONFIG (Украдено из APK) ---
# Эти данные мы достаем из декомпилированного приложения или перехвата
API_BASE_URL = "<https://api-mobile.target-shop.com/v3>"
APP_VERSION = "4.20.0"
DEVICE_ID = str(uuid.uuid4()) # Или реальный ID, если сервер его чекает
SECRET_KEY = "super_secret_salt_from_apk_code" # Ключ, зашитый в коде приложения

# --- 2. PARSER CLASS ---
class MobileAppClient:
    def __init__(self):
        self.client = httpx.AsyncClient(http2=True)
        # Мобильные заголовки выглядят иначе, чем браузерные!
        self.base_headers = {
            "Host": "api-mobile.target-shop.com",
            "Accept": "application/json",
            "User-Agent": "TargetShop/4.20.0 (Android 13; Pixel 7 Pro)",
            "X-App-Version": APP_VERSION,
            "X-Device-ID": DEVICE_ID,
            "Connection": "keep-alive",
            # Часто бывает сжатие gzip
            "Accept-Encoding": "gzip",
        }

    def _generate_signature(self, endpoint: str, params: dict, timestamp: str) -> str:
        """
        САМОЕ ГЛАВНОЕ: Имитация подписи запроса.
        Алгоритм подписи у каждого приложения свой. Его нужно найти в коде APK.
        Обычно это MD5 или SHA256 от (URL + Параметры + Timestamp + SecretKey).
        """
        # Пример типичной подписи:
        # sorted_params = строка параметров по алфавиту
        # string_to_sign = endpoint + sorted_params + timestamp + SECRET_KEY

        # Сортируем параметры (частое требование)
        sorted_keys = sorted(params.keys())
        query_string = "&".join([f"{k}={params[k]}" for k in sorted_keys])

        # Собираем строку
        raw_string = f"{endpoint}?{query_string}{timestamp}{SECRET_KEY}"

        # Хешируем (MD5 - классика старых апп, SHA256 - новых)
        signature = hashlib.md5(raw_string.encode('utf-8')).hexdigest()

        return signature

    async def get_items(self, category_id: int):
        endpoint = "/catalog/list"
        url = f"{API_BASE_URL}{endpoint}"

        # Параметры запроса
        params = {
            "category": category_id,
            "offset": 0,
            "limit": 20
        }

        # 1. Генерируем динамические заголовки
        ts = str(int(time.time())) # Текущее время (Timestamp)

        # Генерируем подпись (чтобы сервер поверил, что мы приложение)
        sign = self._generate_signature(endpoint, params, ts)

        headers = self.base_headers.copy()
        headers.update({
            "X-Timestamp": ts,
            "X-Signature": sign, # <-- Вот наш пропуск
            # Иногда нужен X-Auth-Token, если пользователь залогинен
        })

        print(f"📱 Шлем запрос как Android App... (Sign: {sign})")

        try:
            resp = await self.client.get(url, params=params, headers=headers)

            if resp.status_code == 200:
                data = resp.json()
                print("✅ Успех! Чистый JSON:")
                # print(data['items'][0]) # Вывод первого товара
                return data
            elif resp.status_code == 401:
                print("⛔ Ошибка подписи (Signature) или токена.")
            else:
                print(f"⚠️ Статус: {resp.status_code} | {resp.text}")

        except Exception as e:
            print(f"💀 Ошибка: {e}")

    async def close(self):
        await self.client.aclose()

# --- 3. ENTRY ---
async def main():
    mobile_bot = MobileAppClient()
    try:
        await mobile_bot.get_items(category_id=555)
    finally:
        await mobile_bot.close()

if __name__ == "__main__":
    asyncio.run(main())

```

### 🧠 Куда смотреть (Architect Notes):

1. **`User-Agent`**: Обрати внимание, он не начинается с `Mozilla`. Это `okhttp/4.9.0` или `AppName/Version`. Если пошлешь `Mozilla`, сервер поймет, что ты браузер, а не телефон, и может отказать.
2. **`_generate_signature`**: Это сердце метода. Если сервер просто проверяет токен — тебе повезло. Но если есть заголовок `Sign` или `Signature`, тебе придется скачать APK, открыть его в `JADX-GUI`, найти строку "Signature" и переписать логику хеширования на Python.
    - *Подсказка:* Ищи слова `HMAC`, `MD5`, `SHA256`, `append`, `StringBuilder` в коде Java/Kotlin.
3. **SSL Pinning**: Если `mitmproxy` не видит трафик (приложение пишет "Нет сети"), значит внутри стоит защита **SSL Pinning**. Чтобы её обойти, нужен рутованный Android и инструмент **Frida** (скрипт `frida-ssl-pinning-bypass`). Это уже следующий уровень взлома, выходящий за рамки простого Python.