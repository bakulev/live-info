#Создание административного интерфейса Symfony 2. Sonata Admin Bundle.

@(Блокнот)[programming|Symfony2]

[TOC]

##Глобальная установка Symfony2
Используется [инструкция](http://symfony.com/doc/current/quick_tour/the_big_picture.html):

```bash
curl -LsS http://symfony.com/installer > symfony.phar
sudo mv symfony.phar /usr/local/bin/symfony
chmod a+x /usr/local/bin/symfony
```
##Создание нового проекта:

```bash
symfony new live-info
```

## Подготовка git

Используются команды из статьи [Ежедневная работа с Git](http://habrahabr.ru/post/174467/)

создаётся файл `.gitignore`:
```text
/web/bundles/
/app/bootstrap.php.cache
/app/cache/*
/app/config/parameters.yml
/app/logs/*
!app/cache/.gitkeep
!app/logs/.gitkeep
/build/
/vendor/
/bin/
/composer.phar
```
инициализируется git
```bash
git init
```
создаётся репозиторий на github (https://github.com/bakulev/live-info)

добавление первичных файлов:
```bash
touch README.md
git add README.md
git add .
git commit -am "symfony and git init"
```
создаётся ссылка на репозиторий в сети, проверка и заливка: 
```bash
git remote add github https://bakulev:password@github.com/bakulev/live-info.git
git remote show
git remote show github
git push github master
heroku config:set COMPOSER_GITHUB_OAUTH_TOKEN=githuboauthtoken
```
возврат к ранней версии
```bash
git checkout <commit>
```

## Создание приложения Heroku
Создание приложения в регионе EU:
```bash
heroku create live-info-symfony --region eu
```
Проверка, что добавилась ссылка на репозиторий heroku
```bash
git remote -v
```
```text
github  https://bakulev:password@github.com/bakulev/live-info.git (fetch)
github  https://bakulev:password@github.com/bakulev/live-info.git (push)
heroku  git@heroku.com:live-info-symfony.git (fetch)
heroku  git@heroku.com:live-info-symfony.git (push)
```
Для установка нужных расширений php добавить в `composer.json` указанные разделы:
```json
"require": {
        "php": ">=5.3.3",
        "ext-memcached": "*",
        "ext-mbstring": "*",
        }
```
Создать файл `Procfile` с указанием параметров запуска приложения:
```
web: bin/heroku-php-apache2 web/
```

## Первый пробный запуск проекта по умолчанию

Используются инструкции из статьи [How to Deploy a Symfony Application](http://symfony.com/doc/current/cookbook/deployment/tools.html)
Проверка, что система удовлетворяет требованиям:
```bash
php app/check.php
```
Указание рабочей конфигурации
```bash
heroku config:set SYMFONY_ENV=prod
```
Инициализация
```
composer update
composer install --optimize-autoloader
php app/console cache:clear
php app/console assetic:dump
```
Содержимое файла `bin/chmod-www-data.sh`, Изменение прав для локального запуска и проверка:
```bash
#!/bin/bash
sudo chown -R www-data.www-data app/cache
sudo chown -R www-data.www-data app/logs
```
```bash
bin/chmod-www-data.sh
```
Добавляются разрешённые IP-адреса для просмотра dev версии в файле `web/app_dev.php`:
```php
if (isset($_SERVER['HTTP_CLIENT_IP'])
    || !(
        in_array(@$_SERVER['REMOTE_ADDR'], array('127.0.0.1', '109.188.125.21', 'fe80::1', '::1'))
        || in_array(@$_SERVER['HTTP_X_FORWARDED_FOR'], array('109.188.125.21'))
        || php_sapi_name() === 'cli-server'
  )
) {
     header('HTTP/1.0 403 Forbidden');
     exit('You are not allowed to access this file. Check '.basename(__FILE__).' for more information.');
}
```
в файле `web/config.php`:
```php
if (!in_array(@$_SERVER['REMOTE_ADDR'], array(
     '127.0.0.1',
     '::1',
  '109.188.125.21',
))
    || !(in_array(@$_SERVER['HTTP_X_FORWARDED_FOR'], array('109.188.125.21'))) {
     header('HTTP/1.0 403 Forbidden');
     exit('This script is only accessible from localhost.');
 }
```
В файле `composer.json` добавляется в раздел `require` пункт из раздела `require-dev` для того, чтобы на heroku подгружались нужные модули для просмотра dev:
```json
"sensio/generator-bundle": "~2.3"
```
Как описано в инструкции (https://devcenter.heroku.com/articles/getting-started-with-symfony2) при загрузке на heroku dev не подгружается, только prod.

добавляется раздел `depositories` для загрузки исправленной версии Twig:
```json
"repositories": [
        {
             "type": "vcs",
             "url": "https://github.com/bakulev/TwigBundle"
         }
     ],
```
и добавляется в разел `require` новый исправленный Twig:
```json
"symfony/twig-bundle": "dev-patch-1",
```
Теперь можно посмотреть на результат (http://virt-srv1.mipt.ru/live-info/web/app_dev.php/demo/hello/asdfsad)
Изменение прав на cache и log обратно, чтобы cache:clear работал нормально:
```bash
bin/chmod-bakulev.sh
```

## Исправление ошибки в TwigBundle

В оригинальном файле `vendor/symfony/symfony/src/Symfony/Bundle/TwigBundle/Resources/views/layout.html.twig` указывается абсолютный путь к css:
```
<link href="{{ asset('bundles/framework/css/structure.css', absolute=true) }}" rel="stylesheet" />
<link href="{{ asset('bundles/framework/css/body.css', absolute=true) }}" rel="stylesheet" />
```
Из-за этого не работает загрузка через https после инсталляции на heroku.

Приходится заходить на (https://github.com/symfony/TwigBundle/blob/master/Resources/views/layout.html.twig) и нажимать кнопку "Редактировать". Удалить `, absolute=true` и сохранить изменения.

Автоматически после редактирования создаётся форк с веткой "patch-1".

Чтобы её использовать нужно согласно [документации](https://getcomposer.org/doc/05-repositories.md#vcs) в файле `composer.json` добавить:
```
    "repositories": [
        {
            "type": "vcs",
            "url": "https://github.com/bakulev/TwigBundle"
        }
    ],
```
И в разделе "require" изменить или добавить указание на исправленный репозиторий.
```
    "symfony/twig-bundle": "dev-patch-1",
```
И запустить `composer update`

##Установка **FOSUserBundle**
Для хранения пользователей в базе данных нужно установить **FOSUserBundle**. Для этого используется инструкция (https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/index.md)

Добавляется пакет:
```bash
sudo bin/chmod-bakulev.sh
composer require friendsofsymfony/user-bundle "~2.0@dev"
```
Добавляется пакет в файл `app/AppKernel.php`:
```php
public function registerBundles()
{
    $bundles = array(
        // ...
        new FOS\UserBundle\FOSUserBundle(),
    );
}
```
Создаём пакет как в примере:
```bash
php app/console generate:bundle --namespace=Acme/UserBundle
```
Добавляем файл `src/Acme/UserBundle/Entity/User.php`.
Генерируем entity:
```bash
php app/console doctrine:generate:entities Acme/UserBundle/Entity/User
```
Изменяем файлы `app/config/security.yml`.
Добавляем конфигурацию `fos_user` и исправление ошибки как описано в (http://symfony.com/doc/2.7/cookbook/doctrine/dbal.html) в файл `app/config/config.yml`:
```json
fos_user:
    db_driver: orm
    firewall_name: main
    user_class: Acme\UserBundle\Entity\User
doctrine:
    dbal:
        #Для обхода ошибки `Unknown database type enum requested`
        mapping_types:
            enum: string
```
Добавляются пути `app/config/routing.yml`:
```json
fos_user:
    resource: "@FOSUserBundle/Resources/config/routing/all.xml"
```
Добавляется add-on mysql для heroku:
```bash
heroku addons:add cleardb:ignite
```
В файле `app/config/parameters.yml` и `app/config/parameters.yml.dist` (его использует модуль `incenteev/composer-parameter-handler` при установке на heroku) указываются параметры подключения к БД.
Обновляется схема БД:
```bash
php app/console doctrine:schema:update --force
```
Добавить нового пользователя с правами администратора и пробного пользователя согласно документации (https://github.com/FriendsOfSymfony/FOSUserBundle/blob/master/Resources/doc/command_line_tools.md):
```bash
php app/console fos:user:create admin --super-admin
php app/console fos:user:create testuser test@example.com p@ssword
```
Можно смотреть что получилось предварительно установив права:
```bash
sudo bin/chmod-www-data.sh
```

## Установка необходимых зависимостей для AdminBundle

Используется инструкция (http://sonata-project.org/bundles/admin/master/doc/reference/installation.html)

```bash
composer require sonata-project/admin-bundle
composer require sonata-project/block-bundle
composer require knplabs/knp-menu-bundle
composer require sonata-project/doctrine-orm-admin-bundle
```

Добавить в `AppKernel.php`:
```php
<?php

// app/AppKernel.php

public function registerBundles()
{
    return array(
        // ...
        // set up basic sonata requirements
        // ...
        // The admin requires some twig functions defined in the security
        // bundle, like is_granted
        new Symfony\Bundle\SecurityBundle\SecurityBundle(),
        // Add your dependencies
        new Sonata\CoreBundle\SonataCoreBundle(),
        new Sonata\BlockBundle\SonataBlockBundle(),
        new Knp\Bundle\MenuBundle\KnpMenuBundle(),
        // Database specifics bundles
        new Sonata\DoctrineORMAdminBundle\SonataDoctrineORMAdminBundle(),
        // Then add SonataAdminBundle
        new Sonata\AdminBundle\SonataAdminBundle(),
   );
}
```

Добавить необходимую конфигурация в `app/config/config.yml`
```yaml
sonata_block:
    default_contexts: [cms]
    blocks:
        # Enable the SonataAdminBundle block
        sonata.admin.block.admin_list:
            contexts:   [admin]
        # Your other blocks
knp_menu:
    twig:  # use "twig: false" to disable the Twig extension and the TwigRenderer
        template: knp_menu.html.twig
    templating: false # if true, enables the helper for PHP templates
    default_renderer: twig # The renderer to use, list is also available by default
```

Install assets from the bundles:
```bash
php app/console assets:install web
Обычно после инсталляции assets хорошо бы очистить кеш:
php app/console cache:clear
```

Добавить пути соответсвтующие в `app/config/routing.yml`:
```yaml
admin:
    resource: '@SonataAdminBundle/Resources/config/routing/sonata_admin.xml'
    prefix: /admin

_sonata_admin:
    resource: .
    type: sonata_admin
    prefix: /admin
```

Для просмотра как это работает:
```bash
sudo chown -R www-data.www-data app/cache
sudo chown -R www-data.www-data app/logs
```
И можно обращаться по адресу: (http://virt-srv1.mipt.ru/live-info/web/app_dev.php/admin)

Добавление нужных файлов:
```bash
vi src/Acme/DemoBundle/Resources/config/admin.yml
vi app/config/config.yml
mkdir src/Acme/DemoBundle/Admin
vi src/Acme/DemoBundle/Admin/PostAdmin.php
vi src/Acme/DemoBundle/DependencyInjection/AcmeDemoBundleExtension.php
```

Установка **SonataUserBundle**
```bash
sudo chown -R bakulev.bakulev app/cache
sudo chown -R bakulev.bakulev app/logs
composer require sonata-project/easy-extends-bundle
composer require sonata-project/user-bundle --no-update
composer update
```

Добавление в `app/AppKernel.php`:
```php
new FOS\UserBundle\FOSUserBundle(),
new Sonata\UserBundle\SonataUserBundle('FOSUserBundle'),
```

Добавление конфигурации в `app/config/config.yml`

Установка авторизации через соц. сети: (https://github.com/hwi/HWIOAuthBundle/blob/master/Resources/doc/index.md)
Интеграция **FOSUserBundle** и **HWIOAuthBundle** (https://gist.github.com/danvbe/4476697)



