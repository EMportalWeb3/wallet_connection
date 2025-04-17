# wallet_connection
Взаимодействие Events Mint portal с кошельком Phantom.
Обзор проекта
Events Mint Portal — это web3-приложение на блокчейне Solana, предлагающее торговых ботов и инструменты для безопасной и удобной торговли SPL-токенами. Пользователи аутентифицируются через кошельки Solana и совершают транзакции для доступа к премиум-функциям. Безопасность и удобство — наши ключевые приоритеты.

Мы сознательно отказались от классических способов авторизации и оплаты, чтобы минимизировать риски, связанные с хранением чувствительной информации. Проект не собирает и не хранит персональные данные пользователей — сохраняются только публичный ключ кошелька Solana, хэши транзакций, персональные настройки и зашифрованные JWT-токены.

Events Mint Portal представляет собой набор графических интерфейсов и алгоритмов, работающих поверх API https://pumpportal.fun/trading-api/. Мы не управляем средствами пользователей и не имеем доступа к их приватным ключам. Все торговые операции (кроме оплаты подписки) выполняются через Lightning Transaction API с использованием персональных API-ключей, которые пользователи генерируют самостоятельно на странице https://pumpportal.fun/trading-api/setup. Этот процесс подробно описан в разделе "Юзер Гайд" на нашем сайте: https://eventsmint.com/user-guide/.

Функционал портала
Портал предлагает инструменты для быстрой и эффективной торговли через Lightning Transaction API:

Bot Rigel — автоматизированная торговля с анализом рынка.
Bot Mira — снайпинг токенов.
Bot Tracker — мониторинг  рынка.
Bot Bustle — ручная HS торговля.
Боты предназначены для снайпинга и торговли SPL-токенами на пулах http://Pump.fun и http://Raydium.io. Они не содержат вредоносных алгоритмов и созданы для честной конкуренции на рынке мемкоинов. Все функции и настройки подробно описаны в инструкциях.

Наша цель — сделать торговлю мемкоинами простой, безопасной и понятной для новичков. Сейчас мы ориентированы на русскоязычную аудиторию, но уже работаем над переводом сайта и инструментов на английский, испанский и китайский языки для глобального расширения. Мы хотим разрушить стереотипы о криптовалюте как о скам-пространстве и создать проект, который привлечет новых пользователей, оставив у них положительный опыт.

Детали интеграции с кошельком Phantom
Мы используем Phantom для двух ключевых функций:

Аутентификация пользователя
Метод: signMessage
Реализация:
Генерируется сообщение с уникальным nonce (для защиты от атак повторного воспроизведения) и текстом о принятии условий (например, "Вы авторизуетесь на портале EM Portal... [nonce:1234]"). Клиент подписывает его через signMessage в Phantom, а подпись проверяется сервером с помощью tweetnacl.sign.detached.verify.
Цель: Подтвердить личность и согласие пользователя без транзакций на блокчейне.
Обоснование: Экономичный (без комиссий) и безопасный метод, соответствующий современным стандартам.
Подпись транзакций для оплаты подписки
Метод: signTransaction
Реализация:
Сервер создает транзакцию (например, перевод 0.1 SOL на адрес SUBSCRIPTION_PAYMENT_ADDRESS через @solana/web3.js), сериализует ее и отправляет клиенту через /create-transaction. Клиент подписывает транзакцию в Phantom и возвращает ее на /broadcast-transaction. Сервер проверяет подпись и транслирует транзакцию через connection.sendRawTransaction.
Цель: Оплата подписки (0.1 SOL за 24 часа доступа).
Обоснование: Подпись на стороне клиента дает пользователю контроль, а серверная трансляция позволяет выполнять дополнительные проверки.
Почему не signAndSendTransaction?
Мы предпочли signTransaction по следующим причинам:

Контроль сервера: Трансляция через сервер обеспечивает повторные попытки при сбоях сети.
Интеграция с бэкендом: После подтверждения транзакции обновляется статус подписки, генерируются токены доступа.
Безопасность: Сервер проверяет сумму транзакции, предотвращая манипуляции.
Примеры кода

Аутентификация (script.js):

const response = await fetch('/get-message');
const { message, nonceToken } = await response.json();
const signedMessage = await provider.signMessage(new TextEncoder().encode(message));
await fetch('/auth-message', { 
  method: 'POST', 
  body: JSON.stringify({ signedMessage: btoa(...), publicKey, message, nonceToken }) 
});


Подпись транзакции (script.js):

const response = await fetch('/create-transaction', { 
  method: 'POST', 
  body: JSON.stringify({ fromPublicKey }) 
});
const { transaction } = await response.json();
const transactionObj = Transaction.from(Uint8Array.from(atob(transaction), c => c.charCodeAt(0)));
const signedTransaction = await provider.signTransaction(transactionObj);
await fetch('/broadcast-transaction', { 
  method: 'POST', 
  body: JSON.stringify({ signedTransaction: btoa(...) }) 
});


Трансляция (app.js):

const transactionBuffer = Buffer.from(signedTransaction, 'base64');
const txid = await connection.sendRawTransaction(transactionBuffer, { 
  skipPreflight: false, 
  preflightCommitment: 'confirmed' 
});


