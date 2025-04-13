# Практика 1.1. Создание базового приложения на FastAPI

### Ход работы

1. Устанавливаем FastAPI

![Alt текст](images_1/1.png)

2.В файле main.py пишем следующий код:
```
from fastapi import FastAPI

app = FastAPI()


@app.get("/")
def hello():
    return "Hello, [username]!"
```
Запускаем сервер и смотрим на результат:
```
uvicorn main:app --reload
```
![Alt текст](images_1/3.png)

![Alt текст](images_1/2.png)

3.Созаддим временную бд прямо в файле main.py 
```
temp_bd = [{
    "id": 1,
    "race": "director",
    "name": "Мартынов Дмитрий",
    "level": 12,
    "profession": {
        "id": 1,
        "title": "Влиятельный человек",
        "description": "Эксперт по всем вопросам"
    },
},
    {
        "id": 2,
        "race": "worker",
        "name": "Андрей Косякин",
        "level": 12,
        "profession": {
            "id": 1,
            "title": "Дельфист-гребец",
            "description": "Уважаемый сотрудник"
        },
    },
]
```

И для новой бд напишем запросы:
![Alt текст](images_1/4.png)

4.Тестируем написанные запросы, вновь запускаем сервер и смотрим на результат:
![Alt текст](images_1/5.png)

![Alt текст](images_1/6.png)

![Alt текст](images_1/7.png)

![Alt текст](images_1/8.png)

![Alt текст](images_1/9.png)

![Alt текст](images_1/10.png)
