# Лабораторная работа №2. Gitlab CI/CD

## Цели работы:
Создать и настроить свой первый пайплайн в Gitlab

## Выполение работы:
### 1. Установка и настройка Gitlab
На сервере, где будет целевой деплой создаём файл docker-compose.yml для деплоя раннера:

```bash
	version: '3.7'
	services:
		gitlab-runner:
			image: gitlab/gitlab-runner:alpine
			container_name: gitlab-runner
			restart: always
			volumes:
			  - /var/run/docker.sock:/var/run/docker.sock
```

В терминале выполняем команду `docker-compose up -d` и ждём, когда деплой завершится (проверить можно командой `docker ps`)

![screenshot](img/Screenshot_277.png)

![screenshot](img/Screenshot_278.png)

Далее, в Gitlab в настройках *группы* проектов идём в `Settings - CI/CD - Runners`, там активируем **Allow members of projects and groups to create runners with runner registration tokens**.

![screenshot](img/Screenshot_279.png)

Потом уже в *проекте* переходим в `Settings - CI/CD`, открываем Runners. Копируем **registration token**.
</br>В терминале выполняем следующую команду для инициализации раннера:

```bash
docker exec -it gitlab-runner gitlab-runner register --url "урл_гитлаба" --clone-url "урл_гитлаба"
```

В процесс выполнения команды будут запрошены данные для инициализации, указываем следующие:

- URL - https://gitlab.com/ 
- Description - имя раннера (любое)
- Registration token - вставляем токен, скопированный ранее;
- Tag - придумываем посложнее (например docker_0910), чтоб не совпало с дефолтными раннерами Гитлаба
- Maintenance note - оставляем пустым
- Exectutor - выбраем docker
- Default docker image - docker:stable

![screenshot](img/Screenshot_280.png)

После первичной иниализации раннера, необходимо его донастроить: логинимся внутрь контейнера через `docker exec -it gitlab-runner bash` и открываем файл `/etc/gitlab-runner/config.toml`.
В конце файла добавляем строчку:

```bash
volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/cache"]
```

После чего перезапускаем контейнер командой ` ocker restart gitlab-runner`.
</br> Возвращаемся в веб-интерфейс Gitlab и проверяем, что раннер “подцепился”:

![screenshot](img/Screenshot_281.png)

### 2. Создание пайплайна

Возвращаемся в репозиторий в GitLab. Создаём конфигурационный файл для пайплайна с именем `.gitlab-ci.yml` в корне репозитория:

```bash
	stages:
	   - build # стейдж для билда образа
	   - deploy # стейдж для деплоя сервиса

	deploy-job:
	   stage: deploy
	   image: tmaier/docker-compose:latest # чтоб сделать деплой через композ нам нужен.. композ
	   variables:
	   DOCKER_DRIVER: overlay2
	   tags:
		- docker_0910
	   script:
		  - echo "Deploying application..."
		  - ls -la
		  - docker-compose rm -sf # предварительно чистим уже существующее
		  - docker-compose up -d # деплоим по-старинке
		  - echo "Application successfully deployed!"
		  - 
	clear-job: # дополнительная джоба, которая одноразово сотрет задеплоенное
	   stage: deploy
	   image: tmaier/docker-compose:latest
	   variables:
	   DOCKER_DRIVER: overlay2
	   tags:
		- docker_0910
	   script:
		- docker-compose rm -sf
	   when: manual # только ручное выполнение (после выполнения лабораторной, например)
```

После коммита файла `.gitlab-ci.yml` пайплайн запустится автоматически, ход выполнения можно посмотреть в разделе `CI/CD - Pipelines`

![screenshot](img/Screenshot_282.png)

![screenshot](img/Screenshot_286.png)

Если все окрашено “зеленым”, значит пайплайн успешно отработал, т.е. приложение задеплоил. В терминале можно убедиться, что тестовое приложение успешно развернулось (например, с помощью команды `docker ps`).

### 3. Выполнение задания
Необходимо сделать следующее:
- Добавить стейдж test и соответствующую джобу до этапа деплоя, выполняющую “тест” - можно сделать что-нибудь простое, например проверить что нужные директории в airflow существуют и не потерялись. Главное чтоб этот синтетический тест отрабатывал и не давал
пайплайну выполняться дальше, если джоба упала
- Добавить правило, чтоб пайплайн осуществлял шаг деплоя автоматически только для веток main и develop
- Шаг тестирования выполняется всегда во всех ветках
- Сделать так, чтоб созданный раннер мог использоваться только для тегированных джоб

Добавляем новую джобу для тестового стейджа. Пусть она будет проверять чётно или нечётно колическтво секунд в текущей дате. Назовём её `check_date`:

```bash
	check_date:
	  stage: test
	  variables:
		  DOCKER_DRIVER: overlay2
	  tags:
		- docker_0910
	  script:
		- echo "Checking date..."
		- CURRENT_SECONDS=$(date +%S)
		- echo "Current seconds $CURRENT_SECONDS"
		- if [ $((CURRENT_SECONDS % 2)) -eq 0 ]; then
			   echo "Seconds are even - test passed!";
			else
			   echo "Seconds are odd - test failed!" && exit 1;
			fi
	  allow_failure: false 
```

В stages добавить test до этапе деплоя:

```bash
	stages:
	  - build
	  - test
	  - deploy
```

![screenshot](img/Screenshot_287.png)
![screenshot](img/Screenshot_288.png)

Для джобы deploy-job  укажем ветки, на изменения в которых он будет стартовать:

```bash
   only:
    - main
    - develop
```

Для того, чтобы раннер мог использоваться только для тегированных джоб, идём в  `Settings -  CI/CD -  GitLab Runner`. Тут можно добавлять тэги раннеру.

![screenshot](img/Screenshot_289.png)


Ссылка на репу в гитлабе: https://gitlab.com/itmo_devops_k3250/devops