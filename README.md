## FR
- Публикация постов с фото, описанием и гео-меткой
- Оценка и комментирование постов
- Подписка/отписка на других путешественников
- Просмотр ленты (хронологической по подпискам)
- Поиск популярных мест и постов по гео

## NFR
- Доступность: чтение ленты 99.95%, запись (пост/коммент/реакция) 99.9%
- Латентность: p95 <= 300 мс (чтение), p95 <= 400 мс (запись)
- Консистентность: изменения видны <= 5 c (счётчики, лента)
- Хранение: посты/комментарии не удаляются, бэкапы >= 30 дней
- Безопасность: rate-limits на follow/like, идемпотентность
- Региональность: СНГ, часовые пояса, мультиязычность

## Basic Calculations

**Активность:**
- Посты: 10 % пользователей, 1 пост/день → 1 000 000/день
- Комментарии: 9 %, 2/день → 1 800 000/день
- Реакции: 9 %, 5/день → 4 500 000/день
- Просмотры ленты: 5/день → 50 000 000/день

**RPS по компонентам:**
- Лента: 50 000 000 ÷ 86 400 = **579 RPS**
- Посты: 1 000 000 ÷ 86 400 = **12 RPS**
- Комментарии: 1 800 000 ÷ 86 400 = **21 RPS**
- Реакции: 4 500 000 ÷ 86 400 = **52 RPS**

**Трафик:**

| **Компонент** | **Операций/день** | **RPS** | **Метаданные** | **API-трафик/день** | **Медиа**          | **Media-трафик/день** |
|---------------|------------------:|--------:|----------------|--------------------:|--------------------|----------------------:|
| Лента (GET)   |        50 000 000 |     579 | 30 KB          |            ~1.50 TB | 20 превью × 150 KB |               ~150 TB |
| Посты (POST)  |         1 000 000 |      12 | 4.4 KB         |              ~44 GB | 1 фото × 1 MB      |                 ~1 TB |
| Комментарии   |         1 800 000 |      21 | 2 KB           |             ~3.6 GB | –                  |                     – |
| Реакции       |         4 500 000 |      52 | 40 B           |            ~0.18 GB | –                  |                     – |
| **ИТОГО**     |                 – | **653** | –              |        **~1.51 TB** | –                  |         **~150.1 TB** |

## Data Model

- публикация поста 
```json


{ 
  "id": "uuid",                 // 16
  "user_id": "uint64",          // 8
  "geo": {                      // 24
    "lat": "float64",           // 8
    "lon": "float64"            // 8
  },
  "describe": "string",         // avg 200, max 2048
  "photo": "link_string",       // avg 100, max 256
  "created_at": "timestamp",    // 8
  "updated_at": "timestamp",    // 8
  "counters": {                 // 8
    "likes": "uint32",          // 4
    "comments": "uint32"        // 4
  }
}

```
 - коммент оценка и реакция 
```json
{
  "comment_id": "uuid",         // 16
  "post_id": "uuid",            // 16
  "author_id": "uint64",        // 8
  "text": "string",             // avg 180, max 1024
  "created_at": "timestamp",    // 8
  "updated_at": "timestamp",    // 8

  "Reaction": {                 
    "post_id": "uuid",          // 16
    "user_id": "uint64",        // 8
    "type": "enum(uint8)"       // 1
  }
}

```

 - Подписка 
```json
{
  "follower_id": "uint64",      // 8
  "followee_id": "uint64",      // 8
  "created_at": "timestamp",    // 8
  "status": "enum(uint8)"       // 1
}
```
 - Поиск популярных мест
```json

{
  "limit": "uint16",            // 2
  "offset": "uint32",           // 4
  "sort": {
    "field": "enum(uint8)",     // 1
    "order": "enum(uint8)"      // 1
  },
  "q": "string",                // 8
  "bbox": {                     // 32
    "lat_min": "float64",       // 8
    "lat_max": "float64",       // 8
    "lon_min": "float64",       // 8
    "lon_max": "float64"        // 8
  }
}

```

### Модель хранения данных
![модель](/diagrams/ССДП_1.svg)
#### Расчет сколько потребуется дисков для хранения и обработки всех данных приложения на 1 год


```
Replicatio Factor = 3 
Service operation time = 1 years

feed:
    Disks_for_capacity = 0 (лента не хранит)
    Disks_for_throughput = 18 MB/S / 100 MB/S = 1 
    Disks_for_iops = 579 / 100 = 6
    rf = 18 дисков 
    disk = HDD 32 ТБ (100 IOPS, 100 МБ/с)
feeds(media):  
    capacity = 54.75 PB  
    Disks_for_capacity = 54.75 PB / 100 TB = 548  
    Disks_for_throughput = 1737 MB/s / 500 MB/s = 4  
    Disks_for_iops = 579 / 1000 = 1  
    rf = 548 * 2 = 1096 дисков (Replication Factor = 2)  
    disk = HDD 32 ТБ (100 IOPS, 100 МБ/с)    
poosts:
    capacity = 4.4 * 1.6 TB 
    Disks_for_capacity = 1.6 / 2 = 1
    Disks_for_throughput = 51KB/s / 100 MB/s = 1 
    Disks_for_iops = 12/ 100 = 1
    rf = 3 диска
    disk = SSD (NVMe) 30 ТБ (10 000 IOPS, 3 ГБ/с)
poosts(media):
    capacity = 365 TB 
    Disks_for_capacity = 4
    Disks_for_throughput = 1
    Disks_for_iops = 1
    rf = 12 диска
    disk = HDD 32 ТБ (100 IOPS, 100 МБ/с)  
    
comments: 
    capacity = 1.3 TB 
    Disks_for_capacity = 1.3 / 2 = 1
    Disks_for_throughput = 42KB/s / 100 MB/s = 1 
    Disks_for_iops = 21/ 100 = 1
    rf = 3 диска
    disk = SSD (NVMe) 30 ТБ (10 000 IOPS, 3 ГБ/с)   
    
reactions: 
    capacity = 0.065 TB 
    Disks_for_capacity = 0.065 / 2 = 1
    Disks_for_throughput = 2KB/s / 100 MB/s = 1 
    Disks_for_iops = 52/ 100 = 1
    rf = 3 диска
    disk = SSD (NVMe) 30 ТБ (10 000 IOPS, 3 ГБ/с)     
```

### Партиционирование

- **Партиционирование** — применить внутреннее **RANGE-партиционирование по полю `created_at`** для таблиц  
  `posts`, `comments`, `reactions`, `media` (ежемесячные партиции для ускорения запросов и упрощения архивации).

### 🧮 Расчёт количества хостов

| Подсистема      | Всего дисков (rf) | Дисков на хост | Расчёт             | Хостов |
|-----------------|------------------:|---------------:|--------------------|-------:|
| **posts**       |                 3 |              4 | ceil(3/4) = 1      |  **1** |
| **comments**    |                 3 |              4 | ceil(3/4) = 1      |  **1** |
| **reactions**   |                 3 |              4 | ceil(3/4) = 1      |  **1** |
| **feed**        |                18 |             10 | ceil(18/10) = 2    |  **2** |
| **feeds(media)**|              1096 |             24 | ceil(1096/24) = 46 | **69** |
| **posts(media)**|                12 |             10 | ceil(12/10) = 2    |  **2** |

---

**Итого:**
- Хостов для баз данных (posts, comments, reactions): **3**
- Хостов для медиа-хранилища: **48**
- **Всего: ~51 хост**

