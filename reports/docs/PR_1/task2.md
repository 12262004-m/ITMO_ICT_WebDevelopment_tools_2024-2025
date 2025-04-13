# Практика 1.2. Настройка БД, SQLModel и миграции через Alembic

### Ход работы

1.Устанавливаем SQLModel

```
pip install sqlmodel
pip install psycopg2-binary
```
![Alt текст](images_2/3.png)

![Alt текст](images_2/4.png)

2.После установки всех зависимостей создаем файл с реализацией подключения к БД:
![Alt текст](images_2/1.png)

3.Не забывыаем про созданием самих моделей:
![Alt текст](images_2/2.png)

4.Редактируем main.py, добавив этот код для инициализации БД:
```
@app.on_event("startup")
def on_startup():
    init_db()
```

Смотрим на результат. После запуска приложения в консоли выведется SQL-запрос на создание сущностей, описанных в models.py
![Alt текст](images_2/5.png)

Также зайдем в PostgreSQL и посомтрим на созданные базы данных:
![Alt текст](images_2/6.png)

![Alt текст](images_2/7.png)


4.Напишем теперь запросы:
```
@app.get("/warriors_list")
def warriors_list(session=Depends(get_session)) -> List[Warrior]:
    return session.exec(select(Warrior)).all()


@app.get("/warrior/{warrior_id}", response_model=WarriorProfessions)
def warriors_get(warrior_id: int, session=Depends(get_session)) -> Warrior:
    warrior = session.get(Warrior, warrior_id)
    return warrior


@app.patch("/warrior{warrior_id}")
def warrior_update(warrior_id: int, warrior: WarriorDefault, session=Depends(get_session)) -> WarriorDefault:
    db_warrior = session.get(Warrior, warrior_id)
    if not db_warrior:
        raise HTTPException(status_code=404, detail="Warrior not found")
    warrior_data = warrior.model_dump(exclude_unset=True)
    for key, value in warrior_data.items():
        setattr(db_warrior, key, value)
    session.add(db_warrior)
    session.commit()
    session.refresh(db_warrior)
    return db_warrior


@app.get("/professions_list")
def professions_list(session=Depends(get_session)) -> List[Profession]:
    return session.exec(select(Profession)).all()


@app.get("/profession/{profession_id}")
def profession_get(profession_id: int, session=Depends(get_session)) -> Profession:
    return session.get(Profession, profession_id)
```

И протестируем их:
![Alt текст](images_2/8.png)

![Alt текст](images_2/9.png)

![Alt текст](images_2/10.png)

![Alt текст](images_2/11.png)

![Alt текст](images_2/12.png)

![Alt текст](images_2/13.png)

![Alt текст](images_2/14.png)

![Alt текст](images_2/15.png)

В качестве проверки вновь зайдем в PostgreSQl м проверим одну из таблиц:
![Alt текст](images_2/16.png)

