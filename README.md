# Kittygram - социальная сеть для размещение фотографий котиков. Обучение на Яндекс.Практикум.

## Описание проекта
Можно регистрироваться, загружать фотографии свих котов с описанием, и смотреть котов других пользователей.

## Установка проекта на компьютер 
 - Необходимо клонировать репозиторий `git clone https://github.com/VTolstik/infra_sprint1.git`
 - перейти в директорию с клонированным репозиторием
 - создать и активировать виртуальное окружение `python3 -m venv venv` `source venv/bin/activate` (для windows `python -m venv venv` `source venv/scripts/activate`)
 - установить зависимости `pip install -r requirements.txt`
 - выполнить миграции `python manage.py migrate`
 - создать суперюзера `python manage.py createsuperuser`
 - в директории проекта создать файл _.env_
 - в файл .env заполнить SECRET_KEY и т.д. по аналогии с примером _.env.example_ в директории проекта

# Деплой проекта на удаленный сервер

## Подключение сервера к аккаунту на GitHub
- на сервере должен стоять Git. Для проверки можно выполнить `git --version`
- если Git не установлен - установить командой `sudo apt install git`
- находясь на сервере сгенерировать SSH-ключи `ssh-keygen`
- сохранить открытый ключ в вашем аккаунте на GitHub. Для этого вывести ключ в терминал командой `cat .ssh/id_rsa.pub`. Скопировать ключ от символов ssh-rsa, включительно, и до конца. Добавить это ключ к вашему аккаунту на GitHub.
- клонировать проект с GitHub на сервер: `git clone git@github.com:VTolstik/infra_sprint1.git`

## Запуск backend проекта на сервере
- Установить пакетный менеджер и утилиту для создания виртуального окружения `sudo apt install python3-pip python3-venv -y`
- находясь в директории с проектом создать и активировать виртуальное окружение `python3 -m venv venv`  `source venv/bin/activate` 
- установить зависимости `pip install -r requirements.txt`
- выполнить миграции `python manage.py migrate`
- создать суперюзера `python manage.py createsuperuser`
- в директории проекта создать файл _.env_
- в файл .env заполнить SECRET_KEY и т.д. по аналогии с примером _.env.example_ в директории проекта

## Запуск frontend проекта на сервере
- установить на сервер `Node.js`   командами
`curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash - &&\`
`sudo apt-get install -y nodejs`
- установить зависимости frontend приложения. Из директории `<ваш проект>/frontend/` выполнить команду: `npm i`

## Установка и запуск Gunicorn
- активировать виртуальное окружение проекта и установить gunicorn `pip install gunicorn==20.1.0`
- В директории _/etc/systemd/system/_ создайте файл _gunicorn.service_ `sudo nano /etc/systemd/system/gunicorn.service`  со следующим кодом:

	    [Unit]
    
	    Description=gunicorn daemon
    
	    After=network.target
    
	    [Service]
    
	    User=yc-user
    
	    WorkingDirectory=/home/<имя пользователя в системе>/<имя проекта>/backend/
    
	    ExecStart=/home/<имя пользователя в системе>/<имя проекта>/venv/bin/gunicorn --bind 0.0.0.0:8080 kittygram_backend.wsgi
    
	    [Install]
    
	    WantedBy=multi-user.target

Чтобы точно узнать путь до Gunicorn можно при активированном виртуальном окружении использовать команду `which gunicorn`

## Установка и настройка Nginx

- На сервере из любой директории выполнить команду: `sudo apt install nginx -y`
- Для установки ограничений на открытые порты выполнить по очереди команды: `sudo ufw allow 'Nginx Full'`  `sudo ufw allow OpenSSH`
- включить файервол `sudo ufw enable`
	### собрать и разместить статику frontend-приложения.
- Перейти в директорию _<имя_проекта>/frontend/_  и выполнить команду `npm run build` Результат сохранится в директории ..._/frontend/build/_.  В системную директорию сервера _/var/www/_ скопировать содержимое папки _/frontend/build/_
- открыть файл конфигурации веб-сервера `sudo nano /etc/nginx/sites-enabled/default` и заменить его содержимое следующим кодом:

    	
        server {
    
	        listen 80;
	        server_name публичный_ip_вашего_удаленного_сервера;
        
	        location / {
            root   /var/www/<имя проекта>;
            index  index.html index.htm;
            try_files $uri /index.html;
	        }
    
	    }
- проверить корректность конфигурации `sudo nginx -t`
- перезагрузить конфигурацию Nginx `sudo systemctl reload nginx`
	### настроить проксирование запросов
- Открыть файл конфигурации Nginx _/etc/nginx/sites-enabled/default_ и добавить в него ещё один блок `location`

	    server {
    
	        listen 80;
	        server_name публичный_ip_вашего_удаленного_сервера;
    
	        location /api/ {
	            proxy_pass http://127.0.0.1:8080;
	        }
	        
		    location /admin/ {
			    proxy_pass http://127.0.0.1:8000;
				}
			
	        location / {
	            root   /var/www/<имя_проекта>;
	            index  index.html index.htm;
	            try_files $uri /index.html;
	        }
    
	    }

- Сохранить изменения, проверить и перезагрузить конфигурацию веб-сервера:

	    sudo nginx -t
	    sudo systemctl reload nginx

	### собрать и настроить статику для backend-приложения.
- в файле _settings.py_ прописать настройки 
	

	    STATIC_URL = 'static_backend'
	    STATIC_ROOT = BASE_DIR / 'static_backend'

- активировать виртуальное окружение проекта, перейти в директорию с файлом _manage.py_ и выполнить команду `python manage.py collectstatic`
- в директории_<имя_проекта>/backend/_ будет создана директория _static_backend/_ 
- Скопировать директорию _static_backend/_ в директорию _/var/www/<имя_проекта>/_

## Добавление доменного имени в настройки Django
- в файле _.env_ добавить в список `ALLOWED_HOSTS` доменное имя: 
	ALLOWED_HOSTS = ['ip_адрес_вашего_сервера', '127.0.0.1', 'localhost', 'ваш-домен'] 
- сохранить изменения и перезапустить gunicorn `sudo systemctl restart gunicorn`
- внести изменения в конфигурацию Nginx. Открыть конфигурационный файл командой: `sudo nano /etc/nginx/sites-enabled/default`
- Добавьте в строку `server_name` выданное вам доменное имя (через пробел, без `< >`):

		server {
		...
		    server_name <ваш-ip> <ваш-домен>;
		...
		}
- Проверить конфигурацию `sudo nginx -t` и перезагрузить её командой `sudo systemctl reload nginx`, чтобы изменения вступили в силу.

 ## Получение и настройка SSL-сертификата
 **Установка certbot**
 - Зайдите на сервер и последовательно выполните команды:

	    sudo apt install snapd
	    sudo snap install core; sudo snap refresh core
	    sudo snap install --classic certbot
	    sudo ln -s /snap/bin/certbot /usr/bin/certbot
- запустить certbot и получить SSL-сертификат:

		sudo certbot --nginx
- сертификат автоматически сохранится на вашем сервере в системной директории _/etc/ssl/_  Также будет автоматически изменена конфигурация Nginx: в файл _/etc/nginx/sites-enabled/default_ добавятся новые настройки и будут прописаны пути к сертификату.
- перезагрузить конфигурацию Nginx `sudo systemctl reload nginx`

## Автор
Владимир Толстик - [GitHub](https://github.com/VTolstik)



# infra_sprint1
