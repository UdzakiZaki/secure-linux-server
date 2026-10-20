# Настройка и защита Linux-сервера

Практическая работа: настроен Nginx с сайтом в `/srv/site`, согласованы Unix-права и метки SELinux, разграничены полномочия пользователей, ограничен SSH и входящий трафик, включён аудит изменений конфигурации Nginx. Отчёт описывает состояние, сохранённое в материалах проверки от 18.10.2026.

## Окружение

**CentOS Stream 10 (Coughlan)**. Пакеты: Nginx `1.26.3-15.el10`, firewalld `2.4.3-4.el10`, openssh-server `9.9p1-33.el10`, audit `4.2.1-3.el10`, sudo `1.9.17-13.p2.el10`, policycoreutils и policycoreutils-python-utils `3.11-1.el10`, selinux-policy-targeted `42.1.27-2.el10`. Полные версии и архитектуры — в [сведениях об окружении](evidence/environment.txt), дополненных исполнителем 20.10.2026. [Четыре службы](evidence/final/01-services.txt) имеют состояние `enabled active`; [SELinux](evidence/final/02-selinux.txt) работает в режиме `Enforcing`.

## Проверки

| Проверка | Сохранённый результат | Подтверждение |
|---|---|---|
| Сайт | HTTP 200 локально и с хоста, страница `Test site` | [Локально](evidence/final/03-site-headers.txt), [с хоста](evidence/final/08-site-from-host.txt), [страница](evidence/final/09-site-body-from-host.txt) |
| SELinux | После исправления меток вместо 403 получен 200; есть постоянное правило | [Диагностика](evidence/selinux/selinux-04-diagnosis-supplement.txt), [правило](evidence/selinux/selinux-02-local-rules.txt) |
| Пользователи и sudo | deploy читает сайт и выполняет reload без пароля; stop без пароля не проходит | [Чтение](evidence/users/05-owner-can-read.txt), [reload](evidence/users/09-deploy-reload-allowed.txt), [stop](evidence/users/10-deploy-stop-denied.txt) |
| SSH | Вход admin по ключу подтверждён; root исключён через AllowUsers, пароль отключён в настройках | [Журнал](evidence/ssh/ssh-07-journal-denied.txt), [параметры](evidence/ssh/ssh-03-sshd-effective.txt) |
| firewalld | Порты 22/80 доступны с тестового хоста, 8080 недоступен; итоговое правило ограничивает SSH по источнику | [Тесты портов](evidence/firewall/), [итог](evidence/final/06-firewall.txt) |
| auditd | Правило `nginx_config` загружено, тестовая запись изменения найдена | [Правило](evidence/final/07-audit-rules.txt), [событие](evidence/audit/05-ausearch-key.txt) |

## С чем столкнулся

При настройке сайта в `/srv/site` возник HTTP 403. Журнал Nginx показал `Permission denied`, а AVC — запрет `getattr` для процесса `httpd_t` к файлу с типом `var_t`. Постоянное правило контекстов и `restorecon` обеспечили нужный тип `httpd_sys_content_t`, после чего получен HTTP 200. [Разбор проблемы](docs/selinux-nginx.md) объясняет диагностику и исправление.

После разграничения Unix-прав обычный admin также получил [отказ при просмотре сайта](evidence/users/03b-admin-denied-on-site-dir.txt). Это ожидаемое ограничение режимов 750/640: владелец deploy читает страницу, а Nginx продолжает обслуживать её; [проверки пользователей](evidence/users/) подтверждают результат.

## Воспроизводимость

ОС и версии указаны выше, конфигурации находятся в configs/, порядок действий описан в docs/work-log.md.

[Журнал работы](docs/work-log.md) связывает действия с проверками; содержимое тестовой страницы сохранено в [результате запроса](evidence/final/04-site-body.txt).
