---
description: Установка mapnik 3 и mapnik-python
---

# Установка mapnik

Mapnik v 3.0.21 доступен в  репозитории community

```
$ pacman -Sy mapnik
```

Далее, для работы с python понадобится связка python-mapnik. Проект доступен по ссылке [здесь](https://github.com/mapnik/python-mapnik). Переходим по ссылке, клонируем или скачиваем ZIP архив проекта и распаковываем, далее переходим на ветку v3.0.X,. Активируем свое виртуальное окружение, заходим в корневую папку mapnik-python из консоли и вводим команды

```text
$ export MASON_BUILD=true
$ sudo python setup.py develop install
```

Чтобы проверить, что все правильно установилось, запустите тесты командой

```text
$ python setup.py test
```



