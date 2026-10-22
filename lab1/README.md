## Цель работы

Получить практические навыки сбора данных из открытых API социальной сети ВКонтакте на примере сообщества Самарского университета.

## Задание

Собрать информацию о постах сообщества [Самарского университета](https://vk.ru/samara_university) за период **с 01.08.2026 по 31.08.2026** и сохранить её в файл.

Для каждого поста необходимо собрать:
- дату публикации;
- текст поста;
- количество лайков;
- количество комментариев;
- количество репостов;
- ссылку на пост;
- (дополнительно) количество просмотров.

## Выбранный подход

Использован **официальный VK API** через библиотеку `vk_api` для Python (подход №1 из задания).

## Пояснения к коду

### 1. Импорты и настройки

```python
import vk_api
import datetime
import json
import csv
import time

TOKEN = ''
GROUP_DOMAIN = 'samara_university'

START_DATE = datetime.datetime(2026, 8, 1, 0, 0, 0)
END_DATE = datetime.datetime(2026, 8, 31, 23, 59, 59)

start_ts = int(START_DATE.timestamp())
end_ts = int(END_DATE.timestamp())
```

- `vk_api` - обёртка над VK API, избавляет от ручной сборки HTTP-запросов.
- `datetime` - нужен для перевода дат в Unix timestamp и обратно в читаемый вид.
- `json`, `csv` - модули сохранения результата.
- `GROUP_DOMAIN` - короткое имя сообщества из ссылки `vk.ru/samara_university`.
- `START_DATE` / `END_DATE` - границы периода. VK API сравнивает даты в формате Unix timestamp, поэтому обе границы сразу переводятся в `start_ts` и `end_ts`.

### 2. Авторизация

```python
vk_session = vk_api.VkApi(token=TOKEN)
vk = vk_session.get_api()
```

`VkApi(token=...)` создаёт сессию с готовым токеном. Метод `get_api()` возвращает объект, у которого методы называются так же, как методы VK API.

### 3. Цикл постраничной загрузки

```python
offset = 0
count = 100
stop_flag = False

while not stop_flag:
    response = vk.wall.get(
        domain=GROUP_DOMAIN,
        count=count,
        offset=offset,
        extended=0,
    )
```

- `count = 100` - максимально допустимый размер страницы.
- `offset` - сдвиг: сколько постов уже пропущено. Первый запрос берёт посты 0–99, второй 100–199 и т.д.
- `domain` - сообщество задаётся коротким именем, а не числовым id.
- `stop_flag` - флаг выхода из цикла. Он поднимается, как только встречается пост старше 01.08.2026.

### 4. Фильтрация по дате и сбор полей

```python
for post in items:
    post_date_ts = post['date']

    if post_date_ts < start_ts:
        stop_flag = True
        break

    if start_ts <= post_date_ts <= end_ts:
        owner_id = post['owner_id']
        post_id = post['id']
        post_url = f"https://vk.com/{GROUP_DOMAIN}?w=wall{owner_id}_{post_id}"
        ...
```

- Стена сообщества отсортирована по убыванию даты: сверху самые свежие посты. Поэтому как только встречается пост с датой **раньше** `start_ts`, дальше идти нет смысла. Здесь поднимается `stop_flag` и цикл прерывается.
- Условие `start_ts <= post_date_ts <= end_ts` отсекает посты, опубликованные **после** 31.08.2026.
- `owner_id` у сообществ **отрицательный**. Именно в таком виде он входит в ссылку на пост: `wall-12345678_999`. Поэтому ссылка собирается как `wall{owner_id}_{post_id}`.
- Поля `likes`, `comments`, `reposts`, `views` - вложенные словари. Обращение `.get('count', 0)` защищает от `KeyError`: если счётчик скрыт или отсутствует, подставится 0.

### 5. Обработка ошибок

```python
except vk_api.exceptions.ApiError as e:
    print(f"Ошибка API: {e}")
    break
except Exception as e:
    print(f"Неизвестная ошибка: {e}")
    break
```

- `ApiError` - ошибки самого VK API (неверный токен, нет доступа к сообществу, превышен лимит). Печатаются и прерывают цикл, чтобы не зациклиться на повторяющейся ошибке.
- Общий `Exception` ловит всё остальное (проблемы сети, таймауты).

### 6. Сохранение в JSON

```python
with open(json_filename, 'w', encoding='utf-8') as f:
    json.dump(all_posts, f, ensure_ascii=False, indent=4)
```

- `encoding='utf-8'` - обязательна для кириллицы и эмодзи.
- `ensure_ascii=False` - иначе русский текст станет нечитаемым.
- `indent=4` - форматирование с отступами, удобно для проверки глазами.

### 7. Сохранение в CSV

```python
fieldnames = ['id', 'owner_id', 'url', 'date', 'text', 'likes', 'comments', 'reposts', 'views']
with open(csv_filename, 'w', encoding='utf-8-sig', newline='') as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(all_posts)
```

- `encoding='utf-8-sig'` - для корректного открытия файла.
- `newline=''` - требование модуля `csv`: без него на Windows между строками появляются пустые строки.
- `DictWriter` берёт словари с одинаковыми ключами и раскладывает их по колонкам; `fieldnames` задаёт порядок колонок и заголовки.

### 8. Точка входа

```python
if __name__ == '__main__':
    main()
```

