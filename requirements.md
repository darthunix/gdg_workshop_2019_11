# Подготовка окружения

1. Установка docker

Для всех платформ инструкции на сайте docker - https://docs.docker.com/install/

2. Закачка образа postgres

```bash
docker pull postgres:12-alpine
```

3. Проверка, что все работает

```bash
docker run -d --rm --name pg postgres:12-alpine
docker exec -it pg psql -U postgres -c "select 'hello world' as result;"
docker stop pg 
```
В результате должна появиться надпись "hello world" - значит, все настроено корректно.