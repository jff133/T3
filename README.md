# Сценарий оплаты на компьютере банковской картой (в две стадии):
1.	Пользователь нажимает «Перейти к оформлению» : на данный момент пользователь собрал корзину и перешел к заказу. Система Ozon собрала все данные по заказу (товары, скидки)
   <img width="837" height="332" alt="image" src="https://github.com/user-attachments/assets/ef890421-0360-4783-8bd7-4b62abe3cbdd" />

3.	Выбор способа оплаты и оформление заказа: система Ozon предоставляет выбор способа оплаты и способа получения, показывает данные заказа и итоговую сумму. Пользователь выбирает оплату картой, и нажимает «Оформить заказ».
   <img width="837" height="571" alt="image" src="https://github.com/user-attachments/assets/72297e66-cb22-4431-91f7-0cb2fc69f37a" />

4.	Ozon запрашивает токен платежа в Юкассу: запрос состоит из описания платежа, суммы, объекта confirmation с типом embedded. В объекте платежа ЮКасса вернет уникальный `confirmation_token`
5.	На стороне Ozon (как UI) появляется форма для ввода данных карты. Пользователь вводит номер карты, срок действия, и `CVC/CVV`. Они передаются на бэкенд Юкассы вместе с токеном  `confirmation_token`. Возвращается статус `pending`. 
6.	Верификация `3-D Secure`: Юкасса передает информацию о транзакции и запрашивает авторизацию, банк отправляет смс с одноразовым кодом по номер телефона, который привязан к карте.  Юкасса перенаправляет пользователя в банк для подтверждения операции. В приложении банка пользователь вводит код, банк проверяет, и Юкасса получает подтверждение от банка и завершает холдирование средств на счёте, если код введен верно, иначе после часа ожидания оформление заказа сбрасывается, транзакция не оформляется. 
7.	Система Ozon через webhook получает уведомление от Юкассы о статусе платежа `waiting_for_capture`. То есть средства заморожены и ожидают списания.
8.	Когда заказ собран и готов к отправке, бэкенд Ozon подтверждает запрос на списывание замороженных средств с карты. Меняется статус платежа на `succeeded`.
9.	Пользователь видит страницу с подтверждением оплаты
   <img width="786" height="377" alt="image" src="https://github.com/user-attachments/assets/a0ed3f25-5954-424f-9c45-6c60d9c0651a" />



# ВИЗУАЛИЗАЦИЯ 


<img width="1287" height="895" alt="image" src="https://github.com/user-attachments/assets/2fb6ba5e-a667-406f-a313-d7e92799a406" />



# Вызовы API:
### 1.	Request:
1.	Создание платежа
2.	URL: https://api.yookassa.ru/v3/payments
3.	Метод : POST
4.	Авторизация: Basic Auth (Shop ID и Secret Key)
5.	Формат запроса:

	<img width="500" height="800" alt="image" src="https://github.com/user-attachments/assets/3b632da6-f42c-40d5-8231-ec04c54518d9" />
6.	Формат ответа:
   <img width="500" height="800" alt="image" src="https://github.com/user-attachments/assets/734bb3fa-0224-4117-b293-ab62e3c2f4aa" />
      
### 2.	Capture:
1. Подтверждение платежа
2. URL: https://api.yookassa.ru/v3/payments/{payment_id}/capture
3. Метод: POST
4. Авторизация: Basic Auth (Shop ID и Secret Key)
5. Формат запроса:

   <img width="215" height="124" alt="image" src="https://github.com/user-attachments/assets/5fa16633-8948-40fd-8f78-b43eb48a0149" />
6. Формат ответа:

    <img width="412" height="302" alt="image" src="https://github.com/user-attachments/assets/f7ce8922-e692-478d-8d44-71b1c00d724f" />

### 3.	Cancel:
1. Отмена холдированного платежа
2. URL: https://api.yookassa.ru/v3/payments/{payment_id}/cancel
3. Метод: POST
4. Авторизация: Basic Auth (Shop ID и Secret Key)
5. Формат запроса:
<img width="224" height="126" alt="image" src="https://github.com/user-attachments/assets/bca217b5-5652-47b0-8078-3d5545370afd" />
6. Формат ответа:
<img width="420" height="102" alt="image" src="https://github.com/user-attachments/assets/311a5e5e-3952-4ad9-aa68-ea0e7dd79ec5" />

### 4. Get Status
1. Запрос информации о платеже
2. Метод: GET
3. URL: https://api.yookassa.ru/v3/payments/{payment_id}
4. Формат запроса:  `GET https://api.yookassa.ru/v3/payments/2c510842-000f-5000-8000-1123e4d94d50`
5. Формат ответа: 

<img width="412" height="104" alt="image" src="https://github.com/user-attachments/assets/26867464-541b-4845-b7b5-c4e3cecdaf1d" />

# Белые пятна:
**1.	Webhook не пришел:** Необходимо реализовать механизм поллинга (polling). Если в течение определенного времени не пришел webhook, бэкенд Ozon должен самостоятельно запросить статус платежа.

**2.	Сбой при списании (capture):** Если запрос на списание возвращает ошибку, необходимо логировать ее и уведомить команду поддержки.

# Сценарий для СБП-оплаты*
1. Пользователь выбирает оплату СБП
2. Ozon создает платеж: Бэкенд Ozon отправляет запрос в ЮKassa с типом `payment_method_data.type = 'sbp'`. В ответе ЮKassa возвращает `qr_code_url`.
3. Ozon показывает пользователю QR-код.
4. Пользователь сканирует QR-код через мобильное приложение своего банка.
5. Пользователь подтверждает опату.
6. Уведомление Ozon: ЮKassa отправляет webhook об успешном платеже (succeeded).
7. Завершение: Ozon отображает страницу "Спасибо за покупку".
