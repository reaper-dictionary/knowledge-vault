
## Термины

`systemd` — главный менеджер системы и сервисов Linux. Системный `systemd` работает как процесс с `PID 1`.

`Unit` — объект, которым управляет systemd.

Основные типы:

- `.service` — сервис или программа;
- `.socket` — сетевой или файловый сокет;
- `.timer` — запуск по расписанию;
- `.target` — группа связанных юнитов.

`systemctl` — команда для управления сервисами.

`journalctl` — команда для чтения системных логов.

`MainPID` — PID главного процесса сервиса.

`Transient unit` — временный юнит, созданный через `systemd-run`. Его unit-файл не сохраняется постоянно.

## Состояния сервиса

```
active       — работает
inactive     — остановлен
failed       — завершился с ошибкой

enabled      — включён автозапуск
disabled     — автозапуск отключён
masked       — запуск полностью запрещён
```

Важно:

```
active ≠ enabled
```

Сервис может работать сейчас, но не запускаться автоматически после перезагрузки.

## Основные команды

Проверить сервис:

```
systemctl status docker
```

Управлять сервисом:

```
sudo systemctl start docker
sudo systemctl stop docker
sudo systemctl restart docker
```

Управлять автозапуском:

```
sudo systemctl enable docker
sudo systemctl disable docker
```

Запустить и сразу включить автозапуск:

```
sudo systemctl enable --now docker
```

Посмотреть упавшие сервисы:

```
systemctl --failed
systemctl --user --failed
```

Сбросить отметку `failed`:

```
systemctl --user reset-failed
```

`reset-failed` не исправляет ошибку — только очищает сохранённое состояние.

## Системные и пользовательские сервисы

Системный сервис:

```
systemctl status docker
```

Пользовательский сервис:

```
systemctl --user status dev-tr
```

Если сервис создан с `systemd-run --user`, управлять им тоже нужно с `--user`.

## Логи

Последние логи сервиса:

```
journalctl -u docker -n 20
```

Для пользовательского сервиса:

```
journalctl --user -u dev-fail -n 20
```

Логи текущей загрузки системы:

```
journalctl -u docker -b
```

Логи за определённое время:

```
journalctl --since "10 minutes ago"
```

Логи в реальном времени:

```
journalctl -u docker -f
```

Выйти из режима наблюдения:

```
Ctrl+C
```

## Диагностика сервиса

Обычно достаточно двух команд:

```
systemctl status имя-сервиса
journalctl -u имя-сервиса -n 20
```

В `status` ищем:

- `active`, `inactive` или `failed`;
- `Main PID`;
- `status=...`;
- последние сообщения об ошибке.

Главная формула:

```
systemd    — управляет сервисами
systemctl  — управляет systemd
journalctl — показывает логи

status → узнать состояние
journalctl -u → найти причину ошибки
```