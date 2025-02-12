# Basic CI/CD

Разработка простого **CI/CD** для проекта *SimpleBashUtils*. Сборка, тестирование, развертывание.

<br>

## Содержание

1. [Настройка gitlab-runner](#part-1-настройка-gitlab-runner) \
  1.1. [Установка виртуальной машины](#поднять-виртуальную-машину-ubuntu-server-2204-lts) \
  1.2. [Установка gitlab-runner](#скачать-и-установить-на-виртуальную-машину-gitlab-runner) \
  1.3. [Регистрация в gitlab-runner](#запустить-gitlab-runner-и-зарегистрировать-его-для-использования-в-текущем-проекте-do6_cicd)
2. [Cборка](#part-2-сборка) \
  2.1. [Добавление этапа сборки](#напиcать-этап-для-ci-по-сборке-приложений-из-проекта-c2_simplebashutils) \
  2.2. [Неподготовленная оболочка](#неподготовленная-оболочка) \
  2.3. [Проверка сборки проекта](#проверка-сборки-проекта)
3. [Тест кодстайла](#part-3-тест-кодстайла) \
  3.1. [Добавление этапа теста кодстайла](#напиcать-этап-для-ci-который-запускает-скрипт-кодстайла-clang-format) \
  3.2. [Ошибочный вывод пайплайна](#проверить-зафейлился-ли-пайплайн-если-совершена-ошибка-в-кодстайле) \
  3.3. [Корректный вывод пайплайна](#исправить-ошибку-в-форматировании-кода-и-проверить-результат)
4. [Интеграционные тесты](#part-4-интеграционные-тесты) \
  4.1. [Добавление этапа интеграционных тестов](#написать-этап-для-ci-который-запускает-интеграционные-тесты-из-того-же-проекта) \
  4.2. [Ошибочный вывод пайплайна](#проверить-зафейлился-ли-пайплайн-если-обнаруживаются-ошибочные-рузльтаты-интеграционных-тестов) \
  4.3. [Корректный вывод пайплайна](#исправить-ошибку-для-успешного-прохождения-тестов-и-проверить-результат)
5. [Этап деплоя](#part-5-этап-деплоя) \
  5.1. [Установка виртуальной машины](#поднять-вторую-виртуальную-машину-ubuntu-server-2204-lts) \
  5.2. [Статическая маршрутизация между двумя машинами](#статическая-маршрутизация-между-двумя-машинами) \
  5.3. [Генерация ssh-ключей](#генерация-ssh-ключей) \
  5.4. [Относительно bash-скрипта](#написать-bash-скрипт-который-при-помощи-ssh-и-scp-копирует-файлы-полученные-после-сборки-артефакты-в-директорию-usrlocalbin-второй-виртуальной-машины) \
  5.5. [Добавление этапа деплоя](#написать-этап-для-cd-который-«разворачивает»-проект-на-другой-виртуальной-машине) \
  5.6. [Настройка ssh-агента](#настройка-ssh-агента) \
  5.7. [Проверка этапа деплоя](#проверка-этапа-деплоя)
6. [Дополнительно. Уведомления](#part-6-дополнительно-уведомления) \
  6.1. [Создание бота и получение данных](#настроить-уведомления-о-успешномнеуспешном-выполнении-пайплайна-через-бота-с-именем-«kossadda-do6-cicd»-в-telegram) \
  6.2. [Настройка бота](#настройка-бота) \
  6.3. [Проверка работы бота](#проверка-работы-бота)

<br>




## [Part 1. Настройка **gitlab-runner**](#содержание)

**== Задание ==**

### Поднять виртуальную машину *Ubuntu Server 22.04 LTS*

![1.1](screenshots/1.1.etc.PNG) 

> В дальнейшем работа на Ubutu Server будет производиться через openssh на декстопной версии Ubuntu

### Скачать и установить на виртуальную машину **gitlab-runner**

> Был выбран метод установки gitlab-runner через [бинарный файл с официального сайта](https://docs.gitlab.com/runner/install/linux-manually.html)

1. Загрузить бинарный файл <br>
```sh
sudo curl -L --output /usr/local/bin/gitlab-runner "https://s3.dualstack.us-east-1.amazonaws.com/gitlab-runner-downloads/latest/binaries/gitlab-runner-linux-amd64"
```
2. Дать файлу разрешение на исполнение: <br>
```sh
sudo chmod +x /usr/local/bin/gitlab-runner
```
3. Создать пользователя GitLab CI <br>
```sh
sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash
```
4. Установить как службу <br>
```sh
sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner
```
5. Запустить службу
```sh
sudo gitlab-runner start
```

![1.2](screenshots/1.2.gitlab_status.PNG) 

### Запустить **gitlab-runner** и зарегистрировать его для использования в текущем проекте (*DO6_CICD*)

> Для регистрации понадобятся URL и токен, которые можно получить на страничке задания на платформе 

- Зарегистрировать gitlab-runner
```ssh
sudo gitlab-runner register
```
> Для этого необходимо ввести данные при регистрации: <br>
> 1. Cвой URL-адрес GitLab
> 2. Cвой регистрационный токен
> 3. Название раннера
> 4. Теги для заданий, разделенные запятыми
> 5. Тип исполнителя

<br>




## [Part 2. Сборка](#содержание)

### Напиcать этап для CI по сборке приложений из проекта *C2_SimpleBashUtils*

- В корне репозитория создать файл `.gitlab-ci.yml`

```ssh
touch .gitlab-ci.yml
```

- Добавить в файл этап запуска сборки через мейк файл из проекта C2.

> Настроим также этап, чтобы файлы, полученные после сборки (артефакты), сохранялись со сроком хранения 30 дней.

![2.1](screenshots/2.1.cat_gitlab_yml.PNG)

### Неподготовленная оболочка

- При пуше мы столкнемся со следующей ошибкой

![2.2](screenshots/2.2.error.PNG)

> Раннер нас предупреждает, что среда не подготовлена к запуску. Причиной послужила дефолтная конфигурация gitlab-runner, производящая очистку терминала при выходе из оболочки shell. Комментирование строк данного скрипта устраняет данную ошибку

- Закомментируем строки в `/home/gitlab-runner/.bash_logout`

![2.3](screenshots/2.3.cat_bash_logout.PNG)

### Проверка сборки проекта

- Перезапустим пайплайн и проверим пропала ли ошибка

![2.4](screenshots/2.4.building_passed.PNG)

- Как можно увидеть, сборка была успешно осуществлена, исполняемые файлы были сохранены на 30 дней

![2.5](screenshots/2.5.building.PNG)

<br>




## [Part 3. Тест кодстайла](#содержание)

### Напиcать этап для CI, который запускает скрипт кодстайла (clang-format)

![3.1](screenshots/3.1.cat_gitlab_yml.PNG)

### Проверить зафейлился ли пайплайн, если совершена ошибка в кодстайле

- Проверим сначала вывод команды локально

![3.2](screenshots/3.4.styletest_local.PNG)

- Как и ожидалось - пайплайн зафейлился

![3.3](screenshots/3.5.pipeline_fale_styletest.PNG)

- Вывод пайплайна совпал с локальным выводом команды

![3.4](screenshots/3.6.styletest2.PNG)

### Исправить ошибку в форматировании кода и проверить результат

- Результат работы пайплайна

![3.5](screenshots/3.2.pipeline.PNG)

- Теперь проект успешно проходит тест кодстайла

![3.6](screenshots/3.3.styletest.PNG)

<br>




## [Part 4. Интеграционные тесты](#содержание)

### Написать этап для CI, который запускает интеграционные тесты из того же проекта

![4.1](screenshots/4.1.catc.PNG)

### Проверить зафейлился ли пайплайн, если обнаруживаются ошибочные рузльтаты интеграционных тестов

- Проверим сначала вывод интеграционных тестов локально

![4.2](screenshots/4.2.error_local.PNG)

- Проверяем, что пайплайн зафейлился

![4.3](screenshots/4.3.test_error.PNG)

- Вывод пайплайна совпал с локальным выводом результатов

![4.4](screenshots/4.4.pipeline_tests.PNG)

### Исправить ошибку для успешного прохождения тестов и проверить результат

- Результат работы пайплайна

![4.5](screenshots/4.5.passed.PNG)

- Проект успешно проходит интеграционные тесты

![4.6](screenshots/4.6.pipeline_tests_no_error.PNG)

<br>




## [Part 5. Этап деплоя](#содержание)

### Поднять вторую виртуальную машину Ubuntu Server 22.04 LTS

### Статическая маршрутизация между двумя машинами

- Настроим адаптеры обоих машин на внутреннюю сеть

> Для удобства раннеровская машина находится в сети с десктопной версией Ubuntu. В дальнейшем консоли обеих машин будут отображаться на десктопной версии

```sh
sudo vim /etc/netplan/00-network-manager-all.yaml
```

![5.2](screenshots/5.1.cat_netplan_server.PNG)
![55.2](screenshots/5.2.cat_netplan_server2.PNG)

- Обязательно принимаем изменения в настройках адаптеров

```sh
sudo netplan apply
```

- Проверим соединение между машинами

![5.3](screenshots/5.3.ping_to_server2.PNG)
![5.25](screenshots/5.4.ping_to_server.PNG)

### Генерация ssh-ключей

- Для начала сгенерируем пары ключей для каждой машины

```sh
ssh-keygen
```

- Добавим открытый ключ второй машины с вывода `cat /home/kossadda/.ssh/id_rsa.pub` в ssh ключи gitlab для работы с проектом на удаленной машине


### Написать bash-скрипт, который при помощи ssh и scp копирует файлы, полученные после сборки (артефакты), в директорию /usr/local/bin второй виртуальной машины

![5.555](screenshots/5.6.cat_deploy.PNG)

### Написать этап для CD, который «разворачивает» проект на другой виртуальной машине

![5.6](screenshots/5.5.cat_yml.PNG)

### Проверка этапа деплоя

- После пуша обновленного `gitlab-ci.yml` проверяем состояние пайплайна

![5.14](screenshots/5.8.all_passed.PNG)

![5.15](screenshots/5.9.deploy_passed.PNG)

- Судя по выводу раннера деплой прошел успешно. Проверим наличие полученных исполняемых файлов в директории `/usr/local/bin` на удаленной машине

![5.16](screenshots/5.7.find.PNG)

<br>




## [Part 6. Дополнительно. Уведомления](#содержание)

### Настроить уведомления о успешном/неуспешном выполнении пайплайна через бота с именем «kossadda DO6 CI/CD» в Telegram

> Текст уведомления будет содержать информацию об успешности прохождения как этапа CI, так и этапа CD. <br>
> В остальном текст уведомления может быть произвольным.

- Найдем в телеграме через поиск `BotFather`
- Запустим бота и напишем `/newbot`

> В диалоге необходимо будет написать: <br>
> - имя бота <br>
> - юзернейм для бота (имя должно быть уникальным и заканчиваться на `bot`)

![6.2](screenshots/6.1.bot_father.PNG)

- В результате мы получили `API` бота. Теперь найдем бота `getmyid_bot` и напишем ему `/start` для получения нашего `ID`

![6.3](screenshots/6.2.id.PNG)

- далее пишем скрипт который будет контролировать работу каждого этама CI/CD: src/telegram.sh

- Добавляем выполнение скрипта после каждого этапа:

![6.3](screenshots/6.3.after_script.PNG)


### Проверка работы бота

- Протестируем уведомления от всех успешных и каждого зафейленного пайплайна

![6.4](screenshots/6.4.tg.PNG)

### [К содержанию](#содержание)