# Сайт команды: https://spbunited.ru/

Официальный гайд на MkDocs: https://www.mkdocs.org/getting-started/

### Установка
```
sudo apt install mkdocs
pip install mkdocs-bootswatch  #для тем
pip install python-markdown-math  #для формул
```

### Отладка
```
mkdocs serve
```
После этой команды можно сразу видеть изменения по адресу http://127.0.0.1:8000/

### Обновление сайта 
```
mkdocs gh-deploy
```
Новая версия автоматически запушится на github-pages
