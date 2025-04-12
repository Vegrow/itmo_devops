# Лабораторная работа №1.

## Цели работы:
Развернуть Apache Airflow с помощью docker-compose

## Выполение работы:
### 1. Скачиваем docker-compose файл следующей командой:
```bash
curl -LfO 'https://airflow.apache.org/docs/apache-airflow/2.10.5/docker-compose.yaml'
```
### 2. Подготавливаем docker-compose файл к работе.
1 1. Удаляем лишние сервисы: 

- redis;
- airflow-worker;
- airflow-triggerer;
- airflow-cli;
- flower.

2 2. Заменяем дефолтный образ на apache/airflow:2.7.1
3 3. Меняем значение переменной AIRFLOW__CORE__EXECUTOR с CeleryExecutor на LocalExecutor
4 4. В блоке &airflow-common-depends-on удаляем строки, связанные с сервисом Redis

### 3. Создадим необхомые папки для Airflow
Если заранее не создать нужные папки, то поумолчнию они и всё их содержимое будет создано под root'ом, что может вызвать проблемы в работе Airflow.
```bash
mkdir -p ./dags ./logs ./plugins ./config
```

### 4. Запускаем Airflow
Для запуска Airflow и развёртывания всех необходимых сервисов для его работы выполняем следующую команду:
```bash
docker compose up -d
```

![screenshot](/img/1.png)

Проверим успешность запуска командой `docker ps`:

![screenshot](/img/2.png)

Видим, что некоторые контейнеры находятся в состоянии `(health: starting)`.
Это означает, что они в процессе развёртывания и нужно подождать.
Через некоторое время проверяем снова и видим, что все контейтеры в состоянии `(healthy)`

![screenshot](/img/3.png)

После этого можно попасть в Airflow через браузер по адресу: `http://localhost:8080/ `:

![screenshot](/img/4.png)

Вводим airflow/airflow и видим множество примеров DAG-ов:

![screenshot](/img/5.png)

## Вопросы к работе:
1. Для чего нужен docker-compose?
Ответ: для удобного развёртывания системы, состоящий из нескольких компонентов, например веб сервер и база данных к нему.
2. Как в docker-compose сделать ограничения для контейнера поресурсам (CPU, RAM)?
Ответ: это можно сделать с помощбью добавления следующего блока в секцию сервиса:
```
deploy:
      resources:
        limits:
          cpus: '2'
          memory: '256M'
        reservations:
          cpus: '1'
          memory: '128M'
```

Тут `limits` используется для задания максимального количества ресурсов,
а `reservations` для задания минимальных гарантированных ресурсов, которые будут выделены для контейнера.