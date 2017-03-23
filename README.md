Запуск локально
-----------

Установите в качестве значения `aws.s3.bucket` что-нибудь глобально-уникальное в файле `conf/application.conf`.

Установите ключи соединения с AWS:

```bash
    export AWS_ACCESS_KEY=<Ваш AWS Access Key>
    export AWS_SECRET_KEY=<Ваш AWS Secret Key>
```
или
```bash
    SET AWS_ACCESS_KEY=<Ваш	AWS Access Key>
    SET AWS_SECRET_KEY=<Ваш AWS Secret Key>
```


Запустите Play:

    play ~run
	
или

	sbt ~run

Откройте в браузере:

> [http://localhost:9000](http://localhost:9000)


Запуск в облаке Heroku
-------------

Создайте новое пиложение:

    heroku create

Установите ваши ключи соединения с AWS:

    heroku config:add AWS_ACCESS_KEY=<Ваш AWS Access Key> AWS_SECRET_KEY=<Ваш AWS Secret Key>

Push приложения в Heroku:

    git push heroku master

Откройте:

    heroku open
