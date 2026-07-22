markdown
# Часть 1. Реагирование на инциденты

В этой части описаны три инцидента информационной безопасности, с которыми я столкнулся в компании Layer 8, и действия по их расследованию и устранению.

## Выполненные задачи

1. **Массовые блокировки пользователей веб-сайта**
2. **Подозрительная активность на хосте pc_vkrutikov**
3. **Подозрительная активность на хостах srv_waf и srv_ldap**

---

## Инцидент №1. Массовые блокировки пользователей

### Описание проблемы

Клиенты жаловались на ошибки `503 Service Unavailable` и `429 Too Many Requests` при работе с сайтом. При анализе журналов было обнаружено:

**В access.log nginx:**
192.168.2.15 - - [12/Dec/2025:10:23:45 +0000] "GET /candidates?page=2 HTTP/1.1" 200 15432
192.168.2.15 - - [12/Dec/2025:10:23:46 +0000] "GET /candidates?page=3 HTTP/1.1" 200 15800
... (30+ запросов за 2 секунды)
192.168.2.15 - - [12/Dec/2025:10:23:48 +0000] "GET /candidates?page=15 HTTP/1.1" 503 0

text

**В журнале ModSecurity:**
ModSecurity: Access denied with code 503 (phase 2). Operator GE matched 60 at IP: rate.
[msg "Too Many Requests"] [data "current request rate 62/min - exceeded limit 60/min"]

text

### Причина

Ложное срабатывание механизма ограничения частоты запросов. Легитимные пользователи (сотрудники кадрового агентства) делали большое количество запросов (активный поиск, просмотр анкет), что превышало установленный лимит 60 запросов в минуту.

### Решение

Настроена гибкая политика rate limiting с исключениями для доверенных сегментов:

```nginx
limit_req_zone $binary_remote_addr zone=dynamic:10m rate=120r/m;
limit_req_zone $binary_remote_addr zone=static:10m rate=300r/m;

geo $limited_ip {
    default 1;
    192.168.2.0/24 0;   # Пользовательский сегмент — не ограничивать
    192.168.6.0/24 0;   # Сегмент разработки — не ограничивать
}

server {
    location /candidates {
        if ($limited_ip = 0) {
            set $rate_limit_bypass 1;
        }
        limit_req zone=dynamic burst=30 nodelay;
    }
}
