# Networking basics #1

## المفاهيم

**`/etc/hosts`** — ملف محلي يربط أسماء النطاقات بعناوين IP. النظام يقرأه
**قبل** ما يسأل خادم DNS، فأي سطر فيه يتجاوز الـ DNS تماماً.

**`ip` و `ifconfig`** — أوامر لعرض وإدارة واجهات الشبكة وعناوينها.
`ip -4 -o addr show` تعرض كل عناوين IPv4 في سطر واحد لكل واجهة.

**`netcat` (nc)** — أداة تقرأ وتكتب عبر الشبكة. مع `-l` تصير سيرفر يستمع
على منفذ معيّن، وتقدر تتصل فيه بـ `telnet`.

## الملفات

| الملف | المهمة |
| --- | --- |
| `0-change_your_home_IP` | يجعل `localhost` يشير إلى `127.0.0.2` و `facebook.com` إلى `8.8.8.8` |
| `1-show_attached_IPs` | يعرض كل عناوين IPv4 النشطة على الجهاز |
| `2-port_listening_on_localhost` | يستمع على المنفذ 98 على localhost |

## التشغيل

```bash
sudo ./0-change_your_home_IP
./1-show_attached_IPs | cat -e

# طرفية 1
sudo ./2-port_listening_on_localhost
# طرفية 2
telnet localhost 98
```

⚠️ `0-change_your_home_IP` يعدّل `/etc/hosts`. لو شغّلتيه على جهازك الأساسي،
رجّعي `localhost` إلى `127.0.0.1` بعدها وإلا بتتعطّل أشياء كثيرة.
