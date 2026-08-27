# 📱 Архитектура сбора и маршрутизации заявок: Webflow ➔ Telegram ➔ WhatsApp ➔ Google Sheets

Данный документ описывает техническую реализацию сбора заявок с сайта **IAMNUA Thai Massage (Rawai)** без необходимости программирования сложного бэкенда.

---

## 🏗️ 1. Общая схема движения данных

```text
┌────────────────────────────────────────────────────────────────────────┐
│                          ФРОНТЕНД (WEBFLOW)                            │
│  1. Заявка на 1 услугу (Модалка / Страница услуги)                     │
│  2. Заявка на Мульти-корзину (Несколько услуг / Парный массаж)         │
└───────────────────────────────────┬────────────────────────────────────┘
                                    │ POST JSON
                                    ▼
┌────────────────────────────────────────────────────────────────────────┐
│                        ХАБ МАРШРУТИЗАЦИИ (MAKE.COM)                    │
│                     Бесплатный сценарий Webhook                        │
└───────┬───────────────────────────┬────────────────────────────┬───────┘
        │                           │                            │
        ▼                           ▼                            ▼
┌───────────────┐           ┌───────────────┐            ┌───────────────┐
│   TELEGRAM    │           │ GOOGLE SHEETS │            │   WHATSAPP    │
│  Бот в группе │           │  Таблица для  │            │  Уведомление  │
│  администраторов          │  учёта салона │            │  администратору
└───────────────┘           └───────────────┘            └───────────────┘
```

---

## 📦 2. Какие данные передаются с сайта в вебхук

При отправке заявки без предоплаты (или при оформлении корзины) JavaScript собирает объект:

```json
{
  "order_type": "FREE_BOOKING",
  "order_id": "REQ-748291",
  "created_at": "2026-08-28T14:30:00+07:00",
  "client": {
    "name": "Александр",
    "phone": "+66 81 234 5678",
    "email": "alex@gmail.com"
  },
  "booking_details": {
    "date": "2026-08-29",
    "time_slot": "День (13:00 - 17:00)",
    "guest_count": 2,
    "location": "Салон IAMNUA Раваи (Fisherman Way)"
  },
  "items": [
    {
      "service_id": "ath",
      "title": "Authentic Thai Massage",
      "duration": "90 min",
      "quantity": 1,
      "price_per_item": 1350,
      "total_price": 1350
    },
    {
      "service_id": "gs",
      "title": "Gua Sha Facial Care",
      "duration": "60 min",
      "quantity": 1,
      "price_per_item": 1050,
      "total_price": 1050
    }
  ],
  "financials": {
    "total_base_price": 2400,
    "discount_amount": 0,
    "final_payable_at_salon": 2400,
    "currency": "THB"
  }
}
```

---

## 🤖 3. Настройка Telegram-бота (0$ / мес)

1. **Создание бота:**
   - В Telegram открываем `@BotFather` ➔ команда `/newbot` ➔ получаем `BOT_TOKEN` (например `7182938192:AAFe...`).
2. **Создание рабочей группы:**
   - Создаем группу в Telegram `NUA Rawai | Бронирования` и добавляем туда всех администраторов и управляющего.
   - Добавляем созданного бота в группу и делаем его администратором.
   - Получаем `CHAT_ID` группы (через `@getidsbot` или `@myidbot`, например `-1001928374829`).
3. **Шаблон сообщения в Telegram (HTML):**

```html
🔔 <b>НОВАЯ ЗАЯВКА НА БРОНЬ #{{order_id}}</b>
━━━━━━━━━━━━━━━━━━━━━━
🧖‍♀️ <b>Услуги:</b>
{{#items}}
• {{title}} ({{duration}}) × {{quantity}} шт. — {{total_price}} ฿
{{/items}}

👥 <b>Гости:</b> {{booking_details.guest_count}} чел.
📅 <b>Дата и время:</b> {{booking_details.date}}, {{booking_details.time_slot}}
📍 <b>Локация:</b> {{booking_details.location}}
━━━━━━━━━━━━━━━━━━━━━━
👤 <b>Клиент:</b> {{client.name}}
📞 <b>Телефон:</b> <a href="tel:{{client.phone}}">{{client.phone}}</a>
✉️ <b>Email:</b> {{client.email}}
━━━━━━━━━━━━━━━━━━━━━━
💵 <b>К оплате на месте:</b> <b>{{financials.final_payable_at_salon}} THB</b>
⏳ <i>Статус: Требует подтверждения администратором (звонок/мессенджер)</i>
```

---

## 📊 4. Настройка Google Таблицы (Учёт)

В Google Sheets создаётся таблица `IAMNUA_Bookings_Rawai` со столбцами:
1. `A: ID Заказа`
2. `B: Дата создания`
3. `C: Имя клиента`
4. `D: Телефон`
5. `E: Дата визита`
6. `F: Время (Слот)`
7. `G: Гостей`
8. `H: Список услуг`
9. `I: Сумма (THB)`
10. `J: Способ оплаты` (Оплата в салоне / Stripe онлайн)
11. `K: Статус` (Новая / Подтверждена / Завершена)

В Make.com модуль `Google Sheets: Add a Row` автоматически вставляет каждую заявку в конец таблицы.

---

## 💬 5. Варианты интеграции WhatsApp

### Вариант А: Direct WhatsApp Link (Без API и абонентской платы)
На странице подтверждения или в модальном окне выводится кнопка:
`[ 💬 Написать администратору в WhatsApp ]`
Ссылка генерируется динамически:
```javascript
const waPhone = '66812345678'; // Тайский номер салона
const text = encodeURIComponent(`Здравствуйте! Оформил бронь #${orderNumber} на ${date} (${timeSlot}) на услугу ${serviceTitle}. Жду подтверждения.`);
window.open(`https://wa.me/${waPhone}?text=${text}`, '_blank');
```

### Вариант Б: Автоматическая отправка в WhatsApp администратора через Green-API / WATI
В Make.com добавляется модуль Green-API (стоимость от 10$/мес), который сам шлет сообщение на тайский номер салона.

---

## ⚡ 6. Подключение к Webflow

В Webflow код отправки в Make.com вставляется в **Page Settings ➔ Before `</body>`**:

```javascript
async function sendBookingToWebhook(bookingData) {
  const WEBHOOK_URL = 'https://hook.eu2.make.com/your-unique-webhook-id';
  
  try {
    const response = await fetch(WEBHOOK_URL, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(bookingData)
    });
    return response.ok;
  } catch (error) {
    console.error('Ошибка отправки вебхука:', error);
    return false;
  }
}
```
