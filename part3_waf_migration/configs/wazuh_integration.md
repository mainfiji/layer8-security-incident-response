# Интеграция BunkerWeb с Wazuh

## 1. Настройка отправки логов в syslog

В веб-интерфейсе BunkerWeb:
- Settings → Logs
- Включить "Send logs to syslog"
- Указать IP Wazuh-менеджера и порт (UDP 514)
- Формат: JSON

## 2. Настройка приёма в Wazuh

В /var/ossec/etc/ossec.conf:
```xml
<remote>
    <connection>syslog</connection>
    <port>514</port>
    <protocol>udp</protocol>
    <allowed-ips>192.168.0.0/16</allowed-ips>
</remote>
