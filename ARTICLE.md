Использование Amazon S3 для Загрузки Файлов с помощью Java и Play 2
=====================================================

Использование сервиса хранения, такого как [AWS S3](http://aws.amazon.com/s3/) для хранения загруженных файлов предоставляет возможности для неограниченной расширяемости , надежности и скорости, в отличие от обычного хранения файлов в локальной файловой системе. S3, или подобные сервисы хранения важны для проектирования архитектуры приложений и идеально сочетаются с [эфемерной файловой системой](https://devcenter.heroku.com/articles/dynos#ephemeral-filesystem) Heroku.

Эта инструкция покажет вам как создать Java веб-приложение на Play 2 которая сохраняет файлы в облаке  Amazon S3. Прежде чем читать эту статью, ознакомьтесь со статьей  [Использование AWS S3 для Хранения Статических Ресурсов и Загрузка Файлов](https://devcenter.heroku.com/articles/s3) которая рассказывает, как установить необходимые настройки/ключи и более подробно обсуждает преимущества таого подхода.

<div class="note" markdown="1">
Исходный текст данной статьи доступен на <a href="https://github.com/heroku/devcenter-java-play-s3">GitHub</a>.
</div>

Если вы новичок в использовании Play 2 для Heroku тогда вы захотите взглянуть на документацию Play 2 по [Развертыванию в Heroku](http://www.playframework.com/documentation/2.2.x/ProductionHeroku).


Библиотека AWS
----------------------

S3 предоставляет RESTful API для взаимодействия с сервисом.  Есть Java библиотека, которая оборачивает это API, упрощая взаимодействие с ним из Java кода. В Play 2 проекте вы можете добавить зависимость от `aws-java-sdk`, изменив раздел `appDependencies` в [project/Build.scala](https://github.com/heroku/devcenter-java-play-s3/blob/master/project/Build.scala#L10):

 ```scala
    val appDependencies = Seq(
      "com.amazonaws" % "aws-java-sdk" % "1.3.11"
    )
```

После обновления зависимостей в проекте Play 2 вам потребуется перезапустить Play 2 и перегенерировать конфигурационные файлы для Вашей IDE (Eclipse & IntelliJ).


S3 плагин для Play 2
--------------------

В Play 2 есть способ создания плагинов, которые могут быть автоматически запущены при запуске сервера.  Еще пока нет официального  S3 плагина для Play 2, но вы можете сделать свой собственный, создав файл [app/plugins/S3Plugin.java](https://github.com/heroku/devcenter-java-play-s3/blob/master/app/plugins/S3Plugin.java) со следующим содержимым:

 ```java
    package plugins;
    
    import com.amazonaws.auth.AWSCredentials;
    import com.amazonaws.auth.BasicAWSCredentials;
    import com.amazonaws.services.s3.AmazonS3;
    import com.amazonaws.services.s3.AmazonS3Client;
    import play.Application;
    import play.Logger;
    import play.Plugin;
    
    public class S3Plugin extends Plugin {
    
        public static final String AWS_S3_BUCKET = "aws.s3.bucket";
        public static final String AWS_ACCESS_KEY = "aws.access.key";
        public static final String AWS_SECRET_KEY = "aws.secret.key";
        private final Application application;
    
        public static AmazonS3 amazonS3;
    
        public static String s3Bucket;
    
        public S3Plugin(Application application) {
            this.application = application;
        }
    
        @Override
        public void onStart() {
            String accessKey = application.configuration().getString(AWS_ACCESS_KEY);
            String secretKey = application.configuration().getString(AWS_SECRET_KEY);
            s3Bucket = application.configuration().getString(AWS_S3_BUCKET);
            
            if ((accessKey != null) && (secretKey != null)) {
                AWSCredentials awsCredentials = new BasicAWSCredentials(accessKey, secretKey);
                amazonS3 = new AmazonS3Client(awsCredentials);
                amazonS3.createBucket(s3Bucket);
                Logger.info("Using S3 Bucket: " + s3Bucket);
            }
        }
    
        @Override
        public boolean enabled() {
            return (application.configuration().keys().contains(AWS_ACCESS_KEY) &&
                    application.configuration().keys().contains(AWS_SECRET_KEY) &&
                    application.configuration().keys().contains(AWS_S3_BUCKET));
        }
        
    }
```	

`S3Plugin` считывает три параметра конфигурации, устанавливает соединение с S3 и создает S3 Bucket для хранения файлов.  Чтобы включить плагин создайте новый файл с именем [conf/play.plugins](https://github.com/heroku/devcenter-java-play-s3/blob/master/conf/play.plugins) который содержит:

    1500:plugins.S3Plugin

Это дает `S3Plugin` инструкцию стартовать с приоритетом `1500`, означая, что он запустится после всех дефолтных Плагинов Play. 


Сконфигурируйте S3Plugin
----------------------

`S3Plugin` нуждается в трех [конфигурационных парамтрах](https://devcenter.heroku.com/articles/s3#credentials) для корректной работы.  `aws.access.key` содержит AWS Access Key и `aws.secret.key` содержит AWS Secret Key.  Вам также необходимо специфицировать глобально уникальный bucket с помощью параметра `aws.s3.bucket`.  Чтобы установить эти параметры конфигурации вы можете добавить их в файл [conf/application.conf](https://github.com/heroku/devcenter-java-play-s3/blob/master/conf/application.conf#L58):

    aws.access.key=${?AWS_ACCESS_KEY}
    aws.secret.key=${?AWS_SECRET_KEY}
    aws.s3.bucket=com.something.unique

[Не рекомендуется](http://www.12factor.net/config) помещать важную секретную информацию прямо в конфигурационные файлы. Вместо этого `aws.access.key` и `aws.secret.key` берутся из переменных среды окружения с именами `AWS_ACCESS_KEY` и `AWS_SECRET_KEY`.  Вы можете установить эти значения локально, проэкспортировав их следующим образом:

```bash
    $ export AWS_ACCESS_KEY=<Your AWS Access Key>
    $ export AWS_SECRET_KEY=<Your AWS Secret Key>
```
или в Windows
```bash
    $ SET AWS_ACCESS_KEY=<Your AWS Access Key>
    $ SET AWS_SECRET_KEY=<Your AWS Secret Key>
```


Имя `aws.s3.bucket` должно быть изменено на что-нибудь уникальное и связанное с вашим приложением. Например, демо-приложение использует значение  `com.heroku.devcenter-java-play-s3` которое нужно бы изменить на что-то другое, если вы хотите запускать его самостоятельно.


S3File модель
------------

Простой [Объект модели `S3File`](https://github.com/heroku/devcenter-java-play-s3/blob/master/app/models/S3File.java) будет загружать файлы в S3 и сохранять метаданные о файлах в базу данных:

```java
    package models;
    
    import com.amazonaws.services.s3.model.CannedAccessControlList;
    import com.amazonaws.services.s3.model.PutObjectRequest;
    import play.Logger;
    import play.db.ebean.Model;
    import plugins.S3Plugin;
    
    import javax.persistence.Entity;
    import javax.persistence.Id;
    import javax.persistence.Transient;
    import java.io.File;
    import java.net.MalformedURLException;
    import java.net.URL;
    import java.util.UUID;
    
    @Entity
    public class S3File extends Model {
    
        @Id
        public UUID id;
    
        private String bucket;
    
        public String name;
    
        @Transient
        public File file;
    
        public URL getUrl() throws MalformedURLException {
            return new URL("https://s3.amazonaws.com/" + bucket + "/" + getActualFileName());
        }
    
        private String getActualFileName() {
            return id + "/" + name;
        }
    
        @Override
        public void save() {
            if (S3Plugin.amazonS3 == null) {
                Logger.error("Could not save because amazonS3 was null");
                throw new RuntimeException("Could not save");
            }
            else {
                this.bucket = S3Plugin.s3Bucket;
                
                super.save(); // assigns an id
    
                PutObjectRequest putObjectRequest = new PutObjectRequest(bucket, getActualFileName(), file);
                putObjectRequest.withCannedAcl(CannedAccessControlList.PublicRead); // public for all
                S3Plugin.amazonS3.putObject(putObjectRequest); // upload file
            }
        }
    
        @Override
        public void delete() {
            if (S3Plugin.amazonS3 == null) {
                Logger.error("Could not delete because amazonS3 was null");
                throw new RuntimeException("Could not delete");
            }
            else {
                S3Plugin.amazonS3.deleteObject(bucket, getActualFileName());
                super.delete();
            }
        }
    
    }
```

Класс `S3File` имеет четыре поля: `id` который является первичным ключом; `bucket`, в который будет сохранен файл; `name` - имя файла; и актуальный `file` который на самом деле не будет сохранен в базе данных поэтому он `Transient`.

Класс `S3File` перекрывает метод `save`, где он получает сконфигурированное имя бакета из `S3Plugin` и затем сохраняет `S3File` в базу данных, которая присваивает ему новый `id`.  Затем, когда файл загружается в облако S3, используя  Java библиотеку S3.

<div class="callout" markdown="1">
Обратите внимание, что данный пример устанавливает для файла публичный доступ (доступен всем, у кого есть ссылка).
</div>

Наоборот, класс `S3File` также перекрывает метод `delete` для того чтобы сначала удалить файл на S3 перед тем, как `S3File` удалится из базы данных.

Реальное имя файла на извлекается с помощью метода `getActualFileName` , что представляет собой `id` и оригинальное имя файла, соединенное через`/`.  S3 не имеет понятия директории, но данный способ симлирует их и позволяет избежать коллизии в именах файлов.

Класс `S3File` также имеет метод `getUrl` который возвращает URL до файла используя веб-сервис S3.  Это самый прямой путь для пользователя получить фаайл от S3, но он работает только потому, что файл настроен для публичного доступа.

<div class="note" markdown="1">
В качестве альтернативы вы могли бы не делать файлы публичными и создать другой метод  `S3File` который бы использовал вызов S3 API для получения файла.
</div>

## Установка Базы Данных

Теперь, поскольку вы используете базу данных, необходимо сконфигурировать EBean и соединение с базой данных в [conf/application.conf](https://github.com/heroku/devcenter-java-play-s3/blob/master/conf/application.conf#L25):

    db.default.driver=org.h2.Driver
    db.default.url="jdbc:h2:mem:play"
    ebean.default="models.*"

Эти значения используются для локальной разработки, но для запуска на Heroku вы можете использовать [Heroku Postgres Add-on](https://devcenter.heroku.com/articles/heroku-postgres-starter-tier) который автоматически предоставляется для приложений Play.  Чтобы добавить PostgreSQL JDBC драйвер в ваш проект, добавьте следующую зависимость в ваш файл [project/Build.scala](https://github.com/heroku/devcenter-java-play-s3/blob/master/project/Build.scala#L12):

```scala
    "postgresql" % "postgresql" % "9.1-901-1.jdbc4"
```

Чтобы сказать Play фреймворку использовать базу данных PostgreSQL создайте файл с именем `Procfile` содержащий:
```
web: target/start -Dhttp.port=$PORT -DapplyEvolutions.default=true -Ddb.default.driver=org.postgresql.Driver -Ddb.default.url=$DATABASE_URL
```
Это перекроет настройки конфигурации базы данных (чтобы использоваться PostgreSQL (to use PostgreSQL) когда приложение запущено на Heroku.


Контроллер Application
----------------------

Теперь, когда у вас есть модель, которая хранит метаданные о файле и подгружает файл в S3, давайте создадим контроллер, который будет управлять рендерингом веб-страницы с уже загруженными файлами и обрабатывать непосредственно загрузку файлов.  Создайте (или измените) файл с именем [app/controllers/Application.java](https://github.com/heroku/devcenter-java-play-s3/blob/master/app/controllers/Application.java), содержащий:

```java
    package controllers;
    
    import models.S3File;
    import play.db.ebean.Model;
    import play.mvc.Controller;
    import play.mvc.Result;
    import play.mvc.Http;
    
    import views.html.index;
    
    import java.util.List;
    import java.util.UUID;
    
    public class Application extends Controller {
    
        public static Result index() {
            List<S3File> uploads = new Model.Finder(UUID.class, S3File.class).all();
            return ok(index.render(uploads));
        }
    
        public static Result upload() {
            Http.MultipartFormData body = request().body().asMultipartFormData();
            Http.MultipartFormData.FilePart uploadFilePart = body.getFile("upload");
            if (uploadFilePart != null) {
                S3File s3File = new S3File();
                s3File.name = uploadFilePart.getFilename();
                s3File.file = uploadFilePart.getFile();
                s3File.save();
                return redirect(routes.Application.index());
            }
            else {
                return badRequest("File upload error");
            }
        }
    
    }
```

Метод `index` класса `Application` производит выборку объектов `S3File` из базы данных и затем передает их в `index` шаблон для рендеринга.  Метод `upload` принимает загруженный файл, создает новый `S3File` для него, сохраняет его, затем перенаправляет обратно на главную страницу.


Index шаблон
----------

Теперь давайте содадим простую главну страницу, которая будет содержать форму, позволяющую пользователю загрузить файл, а также список загруженных файлов.   Создайте, или обновите файл с именем [app/views/index.scala.html](https://github.com/heroku/devcenter-java-play-s3/blob/master/app/views/index.scala.html) содержащий:
```scala
    @(uploads: List[Upload])
    <!DOCTYPE html>
    
    <html>
    <head>
        <title>File Upload with Java, Play 2, and S3</title>
        <link rel="shortcut icon" type="image/png" href="@routes.Assets.at("images/favicon.png")">
    </head>
    <body>
        <h1>Upload a file:</h1>
        @helper.form(action = routes.Application.upload, 'enctype -> "multipart/form-data") {
            <input type="file" name="upload">
            <input type="submit">
        }
    
        <h1>Uploads:</h1>
        <ul>
        @for(upload <- uploads) {
            <li><a href="@upload.getUrl()">@upload.name</a></li>
        }
        </ul>
    
    </body>
    </html>
```
Этот шаблон содержит форму загрузки файла (создается с использованием метода `helper.form`) и списка файлов.


Маршруты
------

Последняя вещь, которая нуждается в настройке - это маршруты.  Файл [conf/routes](https://github.com/heroku/devcenter-java-play-s3/blob/master/conf/routes) содержит отображение HTTP запросов в методы контроллера.  чтобы задать для GET запросов метод `Application.index`, адля  POST запросов - метод `Application.upload` добавьте [следующее](https://github.com/heroku/devcenter-java-play-s3/blob/master/conf/routes#L5) в ваш файл `conf/routes`:
```
    GET     /                           controllers.Application.index()
    POST    /                           controllers.Application.upload()
```

Запуск в облаке Heroku
-------------

Если вы не склонировали исходники из проекта с примером [git репозиторий](https://github.com/heroku/devcenter-java-play-s3.git), тогда вам потребуется добавить свои файлы в новый репозиторий Git и закоммитить их:

```bash
    $ git init
    $ git add .
    $ git commit -m init
```
Теперь вы можете разместить приложение в облаке Heroku:

```bash
    $ heroku create
```
Установите ваши ключи для соединения с AWS:

```bash
    $ heroku config:add AWS_ACCESS_KEY=<Your AWS Access Key> AWS_SECRET_KEY=<Your AWS Secret Key>
```
Чтобы развернуть приложение в облаке Heroku, сделайте push вашего репозитория Git в Heroku:

```bash
    $ git push heroku master
```
Теперь проверьте, что риложение работает:

```bash
    $ heroku open
```

Информация для дальнейшего изучения и лучшения
----------------

Это лишь очень простой пример, поэтому есть несколько моментов, которые могут быть улучшены для использования в production. В этом примере скачивание файлов обрабатывается Amazon S3.  Лучше настроить кэш загруженных файлов с помощью [Amazon CloudFront](http://aws.amazon.com/cloudfront/).

Данный пример делает двушаговый аплоад так как файл идет сначала в Play, затем в S3.  Вы можете пропустить первый шаг и загружать файлы напрямую в  S3 [POSTя напрямую в S3](http://aws.amazon.com/articles/1434?_encoding=UTF8&jiveRedirect=1).

И, наконец, так как аплоад (и весь ввод-вывод) является блокирующими операциями, вы, возможно, захотите [увеличить размер пула тредов для сервера Play ](http://www.playframework.org/documentation/2.0.2/AkkaCore) чтобы обрабатывать большее число параллельных запросов, так как по-умолчанию предоставляется только 4.
