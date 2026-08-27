# 💳 Техническая интеграция Stripe Checkout без Webflow E-commerce

Данный документ описывает, как реализовать онлайн-оплату картами и Apple Pay со скидкой **-20%** для одной услуги и для всей корзины на стандартном тарифе **Webflow CMS**, экономя сотни долларов в год на тарифах E-commerce и комиссиях.

---

## 🎯 1. Почему этот подход лучше Webflow E-commerce

| Параметр | Webflow E-commerce Plan | Наш Stripe Checkout + Webflow CMS |
|---|---|---|
| **Стоимость тарифа** | $42 – $235 / месяц | $23 / месяц (стандартный CMS план) |
| **Комиссия платформы** | 2% сверх комиссии Stripe | **0% комиссии** (только стандартный Stripe) |
| **Динамические скидки -20%** | Сложно настроить без промокодов | **Автоматически встроены** в JS-логику |
| **Передача слотов и даты** | Требуются костыли | **Нативная передача** в Metadata Stripe |
| **Apple Pay & Google Pay** | Ограничено | **Встроено из коробки** в Stripe Checkout |

---

## ⚙️ 2. Как технически работает оплата

### Сценарий А: Оплата 1 услуги (со страницы услуги или из каталога)
1. Клиент выбирает длительность (например, *Authentic Thai 90 min* = 1 350 ฿) и количество гостей (2 гостя).
2. Базовая цена: `2 × 1 350 = 2 700 ฿`.
3. Онлайн-цена со скидкой 20%: `2 160 ฿`.
4. Клиент нажимает **«Оплатить онлайн со скидкой -20% (2 160 THB)»**.
5. JavaScript обращается к защищенному эндпоинту создания Stripe Checkout Session.

### Сценарий Б: Оплата всей корзины (Мульти-заказ)
1. В корзине 3 разные услуги:
   - *Authentic Thai (60 min)* — 1 шт. × 760 ฿ (было 950 ฿)
   - *Gua Sha Facial (60 min)* — 2 шт. × 840 ฿ (было 1 050 ฿)
   - *Pre-Tan Scrub (30 min)* — 1 шт. × 600 ฿ (было 750 ฿)
2. Итого к оплате онлайн: `760 + 1680 + 600 = 3 040 ฿` (вместо 3 800 ฿).
3. JavaScript формирует массив `line_items` со всеми позициями и скидками и открывает чекаут.

---

## 💻 3. Формирование Stripe Checkout Session (API)

Для создания сессии используется стандартный Stripe API (`POST /v1/checkout/sessions`):

### Пример Payload для Мульти-корзины:

```json
{
  "payment_method_types": ["card"],
  "mode": "payment",
  "success_url": "https://iamnua.com/success?session_id={CHECKOUT_SESSION_ID}",
  "cancel_url": "https://iamnua.com/shop",
  "customer_email": "client@example.com",
  "line_items": [
    {
      "price_data": {
        "currency": "thb",
        "product_data": {
          "name": "Authentic Thai Massage (60 min)",
          "description": "Скидка 20% применена (Базовая цена: 950 THB)",
          "images": ["https://iamnua.com/assets/services/Authentic%20Thai%20Massage/Cover_authentic.PNG"]
        },
        "unit_amount": 76000
      },
      "quantity": 1
    },
    {
      "price_data": {
        "currency": "thb",
        "product_data": {
          "name": "Gua Sha Facial Care (60 min)",
          "description": "Скидка 20% применена (Базовая цена: 1 050 THB)",
          "images": ["https://iamnua.com/assets/services/Gua%20Sha/cover.png"]
        },
        "unit_amount": 84000
      },
      "quantity": 2
    }
  ],
  "metadata": {
    "order_id": "NUA-938210",
    "location": "Rawai Fisherman Way",
    "booking_date": "2026-08-29",
    "time_slot": "День (13:00 - 17:00)",
    "client_name": "Александр",
    "client_phone": "+66 81 234 5678"
  }
}
```

> **Важно по валюте THB в Stripe:** Суммы передаются в минимальных единицах (`satang`): `760 THB = 76000`.

---

## 🔔 4. Обработка успешного платежа (Stripe Webhook)

После того как клиент оплатил картой или Apple Pay:

1. **Stripe мгновенно шлёт событие `checkout.session.completed`** на Webhook в Make.com.
2. **Make.com извлекает метаданные и отправляет отчет в Telegram админам:**

```html
✅ <b>ОПЛАТА ПОДТВЕРЖДЕНА! БРОНЬ #NUA-938210</b>
━━━━━━━━━━━━━━━━━━━━━━
💳 <b>Оплачено через Stripe:</b> <b>3 040 THB</b> (Скидка 20% учтена)
🧖‍♀️ <b>Услуги:</b>
• Authentic Thai (60 min) × 1
• Gua Sha Facial (60 min) × 2

📅 <b>Дата и слот:</b> 29 августа 2026, День (13:00 - 17:00)
📍 <b>Локация:</b> Раваи (Fisherman Way)
👤 <b>Гость:</b> Александр (+66 81 234 5678, client@example.com)
━━━━━━━━━━━━━━━━━━━━━━
⚡ <i>Мастера зарезервированы. Квитанция отправлена клиенту на Email.</i>
```

3. **Запись в Google Таблицу** помечается зеленым статусом `ОПЛАЧЕНО ОНЛАЙН (STRIPE)`.

---

## 🛠️ 5. Как подключить на практике (3 простых шага):

1. **Stripe Account:** Зарегистрировать аккаунт Stripe (тайский юридический аккаунт салона или международный) и получить `Publishable Key` и `Secret Key`.
2. **Бесплатный Micro-Backend (Serverless) для генерации сессий:**
   - Либо через 1 вебхук в Make.com (`Create Stripe Checkout Session`);
   - Либо бесплатный Cloudflare Worker / Netlify Function (10 строк кода на JS).
3. **Frontend в Webflow:**
   - При клике на кнопку «Оплатить» фронтенд вызывает сессию и делает редирект:
   ```javascript
   const stripe = Stripe('pk_live_...');
   const session = await createStripeSession(orderData);
   stripe.redirectToCheckout({ sessionId: session.id });
   ```

---

## 🔒 6. Безопасность и соответствие PCI-DSS
* Все данные кредитных карт вводятся **только внутри защищенного фрейма Stripe**.
* Сайт IAMNUA и Webflow не имеют доступа к номерам карт клиентов и не хранят их (100% безопасность уровня Tier 1 PCI-DSS).
