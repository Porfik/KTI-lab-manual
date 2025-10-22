# Инфраструктура в контейнерах
1. Docker
	1. Установка Docker
	2. Запуск контейнеров
	3. Сборка образа контейнера
2. Docker Compose
	1. Работа с контейнерами
	2. Переменные окружения
3. Автоматизация развёртки приложения
	1. Предварительные настройки окружения
	2. 

---

##### Цель работы:
>Получение навыков работы с контейнерами при помощи Docker и Docker Compose, а также автоматизации развёртки приложений.

---

## Docker
### Установка Docker

>[!WARNING]
>Данная лабораторная работа выполняется на новой виртуальной машине, созданной по [аналогии с первой](Lab_1/#создание-новой-вм).

![](../images/lab_4/4.png)

![](../images/lab_4/4.1.png)

![](../images/lab_4/4.2.png)

![](../images/lab_4/4.3.png)

![](../images/lab_4/4.4.png)

### Запуск контейнеров

![](../images/lab_4/4.5.png)

![](../images/lab_4/4.6.png)

![](../images/lab_4/4.7.png)

![](../images/lab_4/4.8.png)

![](../images/lab_4/4.9.png)

![](../images/lab_4/4.10.png)

![](../images/lab_4/4.11.png)

![](../images/lab_4/4.12.png)

![](../images/lab_4/4.13.png)

![](../images/lab_4/4.14.png)

![](../images/lab_4/4.15.png)

![](../images/lab_4/4.16.png)

![](../images/lab_4/4.17.png)

![](../images/lab_4/4.18.png)

![](../images/lab_4/4.19.png)

```bash
docker container run -d -p 80:80 --rm --name nginx -v '/home/batman/KTI_lab_4/conf:/etc/nginx/conf.d' -v '/home/batman/KTI_lab_4/html:/usr/share/nginx/html' nginx
```

![](../images/lab_4/4.20.png)

![](../images/lab_4/4.21.png)

### Сборка образа контейнера

![](../images/lab_4/4.22.png)

![](../images/lab_4/4.23.png)

![](../images/lab_4/4.24.png)

![](../images/lab_4/4.25.png)

![](../images/lab_4/4.26.png)

![](../images/lab_4/4.27.png)

![](../images/lab_4/4.28.png)

![](../images/lab_4/4.29.png)

---

## Docker Compose
### Работа с контейнерами

![](../images/lab_4/4.30.png)

![](../images/lab_4/4.31.png)

![](../images/lab_4/4.32.png)

![](../images/lab_4/4.33.png)

![](../images/lab_4/4.34.png)

![](../images/lab_4/4.35.png)

![](../images/lab_4/4.36.png)

![](../images/lab_4/4.37.png)

![](../images/lab_4/4.38.png)

![](../images/lab_4/4.39.png)

![](../images/lab_4/4.40.png)

![](../images/lab_4/4.41.png)

![](../images/lab_4/4.42.png)

![](../images/lab_4/4.43.png)

![](../images/lab_4/4.44.png)

![](../images/lab_4/4.45.png)

![](../images/lab_4/4.46.png)

![](../images/lab_4/4.47.png)

### Переменные окружения



---

## Автоматизация развёртки приложения
### Предварительные настройки окружения