# Развертывание web-сервиса на Flask

## Это инструкция по развертыванию web-приложения Flask с помощью Gunicorn и Nginx в Ubuntu 20.04.
#### Иструкция демонстрирует пример развертываия на выделенном сервере
```
```
- Выделенный сервер для развертывания арендован у хостинг-компании FirstVDS;
- Репозиторий для тестирования взят с https://github.com/romsanml/flask_web;
- Сервис является шаблоном для создания чата (вопрос-ответ).
- Основой для материала являются ссылки:
  - https://www.digitalocean.com/community/tutorials/how-to-serve-flask-applications-with-gunicorn-and-nginx-on-ubuntu-20-04
  - https://www.youtube.com/watch?v=KWIIPKbdxD0
```
```

#### Диаграмма развертывания представлена на рисунке
![Alt-текст](https://github.com/Pav9551/flask_test/blob/main/flask_deploy.jpg "Deployment")
## Оглавление

1. [Подключение по ssh](#Подключение-по-ssh)
2. [Создание нового пользователя](#Создание-нового-пользователя)
3. [Установка сервиса на Flask](#Установка-сервиса-на-Flask)
4. [Установка gunicorn](#Установка-gunicorn)
5. [Установка nginx](#Установка-nginx)
## Подключение по ssh
Тестирование сервиса проводилось на операционной системе Ubuntu 20.04 c python 3.8</sup>
1. Подключение к серверу
```curl   
ssh-keygen -R 123.456.789.90 (если нужно очистить ключ) 
ssh root@123.456.789.90
```
1.1 Установить mc
```curl
1. Установить mc
sudo apt -y install mc
```
## Создание нового пользователя
 - Для создания нового пользвателя необходимо:
2. Создать пользователя
```curl
sudo adduser user
```
Ввести новый пароль

3. Даем пользователю права суперпользователя
```curl
usermod -aG sudo user
usermod -aG root user
groups user
```
4. Устанавливаем аутентификацию по паролю
(Аутентифика́ция — процедура проверки подлинности профиля)
```curl
nano /etc/ssh/sshd_config
```
-PasswordAuthentication yes
5. Перезапускаем службу sshd
```curl
systemctl restart sshd
```
6. Выходим из под root
```curl
exit
```
## Установка сервиса на Flask
7. Входим под user
```curl
user@188.120.249.155
```
 и переходим в рабочий каталог home/user.
```curl
cd /home/user
```
8. Копируем файлы проекта
```curl
git clone https://github.com/romsanml/flask_web flask_test
```
8.1 Устанавливаем python 3.8 и необходимые пакеты
```curl
sudo apt update
sudo apt install python3-pip python3-dev build-essential libssl-dev libffi-dev python3-setuptools
sudo apt install python3.8-venv
python3 --version
```
9. Создаем и активируем виртуальную среду
```curl
python3 -m venv ~/env/user
source ~/env/user/bin/activate
```
10. Устанавливаем пакет flask и requests в виртуальное окружение
```curl
pip install wheel
pip install flask requests
```
11. Заходим на каталог
```curl
cd flask_test
```
и меняем в модуле настройки запуска
```curl
nano main.py
if __name__ == "__main__":
    app.run(debug=False)
на    
if __name__ == "__main__":
    app.run(host='0.0.0.0', port=5000)
```
12. Запускаем сервис напрямую для проверки работы 
```curl
python main.py
```
 * Running on all addresses (0.0.0.0)
 * Running on http://127.0.0.1:5000
 * Running on http://123.456.789.90:5000
13. Проверяем с помощью браузера работу сервиса flask
http://123.456.789.90:5000

## Установка gunicorn
14. Устанавливаем пакет gunicorn. 
Gunicorn — это Web-сервер для запуска Web-приложений написанных на Python. Основная его задача — это работа в режиме демона и поддержка постоянной работы Web-приложений. Gunicorn обеспечивает взаимосвязь с NGINX.
```curl  
pip install gunicorn
```

15. Создаем интерфейс шлюза веб-сервера (точка входа)
```curl  
nano wsgi.py
```
```curl  
from main import app 
if __name__ == "__main__":
    app.run()
```
16. Связываем gunicorn c приложением
 
Прежде чем продолжить, нужно убедиться, что Gunicorn может правильно обслуживать приложение.

Для этого нужно просто передать имя нашей точке входа. Оно составляется из имени модуля (без расширения .py) и имени вызываемого элемента приложения. В нашем случае это wsgi:app.

Также мы укажем интерфейс и порт для привязки, чтобы приложение запускалось через общедоступный интерфейс:
```curl
gunicorn --bind 0.0.0.0:5000 wsgi:app
```
0.0.0.0 - это адрес, не подлежащий маршрутизации.

17. Деактивируем виртуальную среду
```curl 
deactivate
```
18. Далее мы созадим файл служебных элементов systemd. Создание файла элементов systemd позволит системе инициализации Ubuntu автоматически запускать Gunicorn и обслуживать приложение Flask при загрузке сервера.
Для этого создаем файл:
```curl 
sudo nano /etc/systemd/system/flask_test.service
```
```curl 
[Unit]
Description=Gunicorn instance to serve flask_test
After=network.target

[Service]
User=user
Group=www-data
WorkingDirectory=/home/user/flask_test
Environment="PATH=/home/user/env/user/bin"
ExecStart=/home/user/env/user/bin/gunicorn --workers 3 --bind unix:flask_test.sock -m 007 wsgi:app

[Install]
WantedBy=multi-user.target
 ```
 - добавить указанного в файле пользователя в группу www-data
```curl
usermod -aG www-data user
groups root
```
 - устанавливаем права на файл настройки сервиса
```curl
sudo chmod 664 /etc/systemd/system/flask_test.service
```
 - устанавливаем права на файл настройки сервиса и перезапускаем его
```curl
sudo systemctl start flask_test
-Failed to start flask_test.service: Unit flask_test.service not found. (нужна перезагрузка сервиса)
sudo systemctl daemon-reload
sudo systemctl start flask_test
sudo systemctl enable flask_test
sudo systemctl status flask_test 
```

## Установка nginx
 - Nginx позволяет увеличить скорость обработки статического контента (в данном примере мы не настраиваем статику)

```curl 
- установка:
sudo apt install nginx
```
Вначале мы создадим новый файл конфигурации серверных блоков в каталоге Nginx sites-available.
```curl 
server {
    listen 80;
    server_name 123.456.789.90;

    location / {
        include proxy_params;
        proxy_pass http://unix:/home/user/flask_test/flask_test.sock;
    }
}
```
В этот блок мы добавим файл proxy_params, определяющий некоторые общие параметры прокси, которые необходимо настроить. После этого запросы будут переданы на сокет, который мы определили с помощью директивы proxy_pass:

Чтобы активировать созданную конфигурацию серверных блоков Nginx, необходимо привязать файл к каталогу sites-enabled символической сылкой:
```curl 
sudo ln -s /etc/nginx/sites-available/flask_test /etc/nginx/sites-enabled
```
Когда файл будет находиться в этом каталоге, можно провести проверку на ошибки синтаксиса:
```curl 
sudo nginx -t
```
20. Если ошибок обнаружено не будет, перезапустите процесс Nginx для чтения новой конфигурации:
```curl 
sudo systemctl restart nginx
sudo systemctl status nginx
```
21. В заключение изменим настройки брандмауэра. Нам больше не потребуется доступ через порт 5000, и мы можем удалить это правило. Затем мы сможем разрешить полный доступ к серверу Nginx:
```curl 
sudo ufw delete allow 5000
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```
22. Устанавливаем права для папки 
```curl 
sudo chmod 775 /home/user/flask_test/
```
23. Проверяем зпущенные сервисы
```curl 
netstat -nlpt
```
