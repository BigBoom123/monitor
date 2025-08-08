# Итоговый документ по проекту мониторинга Prozorro (Финальная версия)

Этот документ содержит всю ключевую информацию и пошаговую инструкцию, необходимые для создания и запуска системы мониторинга, и отражает финальную архитектуру рабочего процесса.

## 1. Цель проекта

- **Задача:** Автоматизировать процесс мониторинга публичных закупок Prozorro и получать мгновенные уведомления о релевантных тендерах в Telegram.
- **Источник:** `pdr.md`

## 2. Технический стек и ресурсы

- **Основной инструмент:** n8n (open-source).
- **Платформа для уведомлений:** Telegram Bot API.
- **Источник данных:** OpenProcurement Tendering API.
- **Документация API:** `https://prozorro-api-docs.readthedocs.io/en/latest/`
- **prozorro.sale API:**
  `https://prozorro.sale/opendata/`
  `https://confluence-sale.prozorro.org/pages/viewpage.action?pageId=94635539`
  `https://confluence-sale.prozorro.org/pages/viewpage.action?pageId=106661394`
- **Документация n8n:**
  - **Основная:** `https://docs.n8n.io/`
  - **Работа с кодом (`Code Node`):** `https://docs.n8n.io/code/code-node/`
  - **Встроенные переменные и методы:** `https://docs.n8n.io/code/builtin/overview/`
- **Репозиторий n8n:** `https://github.com/n8n-io/n8n`
- **Среда для тестирования (Sandbox):** `https://public-api-sandbox.prozorro.gov.ua`

---

## 3. Пошаговая инструкция по настройке

### Шаг 1: Подготовка окружения

1.  **Развертывание n8n:**

    - Запустите n8n с помощью Docker:
      ```bash
      docker volume create n8n_data
      docker run -it --rm --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n
      ```
    - Откройте в браузере `http://localhost:5678`.

2.  **Создание Telegram-бота и получение токена:**

    - В Telegram найдите бота **`@BotFather`**.
    - Отправьте ему команду `/newbot` и следуйте инструкциям.
    - **Сохраните API токен**.

3.  **Получение Chat ID:**
    - Создайте Telegram-канал или группу.
    - Добавьте туда вашего нового бота и бота **`@userinfobot`**, который пришлет ID этого чата. **Сохраните ID**.

### Шаг 2: Сборка рабочего процесса (Workflow) в n8n

**Концепция:** Процесс использует "умную" синхронизацию, чтобы запрашивать только новые тендеры с момента последнего успешного запуска. Логика разделена на получение данных, фильтрацию и отправку, чтобы обеспечить надежность.

1.  **Узел 1: Schedule (Запуск по расписанию)**

    - Триггер: **"Schedule"**.
    - Интервал: **"Every Hour"**.

2.  **Узел 2: Code (Get Last Sync Date)**

    - **Назначение:** Получает дату/время последнего обработанного тендера из хранилища n8n. При первом запуске использует дату по умолчанию.
    - **Код:**
      ```javascript
      const lastSyncDate =
        $workflow.store.get("lastSyncDate") || "2000-01-01T00:00:00";
      return [{ json: { lastSyncDate: lastSyncDate } }];
      ```

3.  **Узел 3: HTTP Request (Get Tender List)**

    - **Назначение:** Получает список ID тендеров, которые были изменены после последней синхронизации.
    - **URL:** `=https://public-api-sandbox.prozorro.gov.ua/api/2.5/tenders?offset={{ $('Get Last Sync Date').item.json.lastSyncDate }}`

4.  **Узел 4: HTTP Request (Get Tender Details)**

    - **Назначение:** Для каждого ID из предыдущего шага получает полную информацию о тендере.
    - **URL:** `=https://public-api-sandbox.prozorro.gov.ua/api/2.5/tenders/{{ $json.id }}`

5.  **Узел 5: IF (Filter by Region & Value)**

    - **Назначение:** Фильтрует тендеры по простым критериям. **Соединяется с выходом Узла 4.**
    - **Conditions:**
      - **Number:** `{{ $json.data.value.amount }}` -> `larger` -> `50000`
      - **String:** `{{ $json.data.procuringEntity.address.region }}` -> `contains` -> `Київ`

6.  **Узел 6: Telegram (Send a text message)**

    - **Важно:** Этот узел должен быть соединен с **`true`** выходом узла `IF`.
    - **Text:**

      ```
      *Новий тендер!*

      *Назва:* {{$json.data.title}}

      *Бюджет:* {{$json.data.value.amount}} {{$json.data.value.currency}}
      *Посилання:* https://prozorro.gov.ua/tender/{{$json.data.id}}
      ```

    - **Options -> Parse Mode:** `MarkdownV2`.

7.  **Узел 7: Code (Save Last Sync Date)**
    - **Важно:** Этот узел должен быть соединен с выходом **Узла 3 (Get Tender List)**, чтобы выполняться параллельно с получением деталей. Это гарантирует, что дата сохранится, даже если ни один тендер не пройдет фильтр.
    - **Назначение:** Сохраняет `dateModified` последнего тендера из полученного списка для следующего запуска.
    - **Код:**
      ```javascript
      const items = $items;
      if (items.length > 0) {
        const lastItem = items[items.length - 1];
        $workflow.store.set("lastSyncDate", lastItem.json.dateModified);
      }
      ```
