# Реализация серверного приложения FastAPI

### Цель
научится реализовывать полноценное серверное приложение с помощью фреймворка FastAPI с применением дополнительных средств и библиотек.

### Выбранная тема
"Разработка платформы для поиска людей в команду"
Платформа предоставляет возможность пользователям создавать профили, описывать свои навыки, опыт и интересы, а также искать других участников и команды для участия в проектах.
- Создание профилей: Возможность пользователям создавать профили, указывать информацию о себе, своих навыках, опыте работы и предпочтениях по проектам. 
- Поиск и фильтрация профилей: Реализация функционала поиска пользователей и команд на основе заданных критериев, таких как навыки, опыт, интересы и т.д. 
- Создание и просмотр проектов: Возможность пользователям создавать проекты и описывать их цели, требования и ожидаемые результаты. Возможность просмотра доступных проектов и их участников. 
- Управление командами и проектами: Возможность участникам создавать команды для совместной работы над проектами и управления участниками. Функционал для управления проектами, включая установку сроков, назначение задач, отслеживание прогресса и т.д.

### Схема данных
![Alt текст](images/shema.png)


### Ход работы
1.Устанавливаем необходимые библиотеки: sqlmodel, alembic, python dotenv, fastapi[all], psycopg2-binary


![Alt текст](images/1.png)

![Alt текст](images/4.png)


2.Реализовываем файл main.py, в котором будет код для создания проекта:
```
from fastapi import FastAPI
from endpoints.user_endpoints import user_router
from endpoints.team_platform_endpoints import team_platform_router
from db.connection import init_db


app = FastAPI()
app.include_router(user_router)
app.include_router(team_platform_router)


@app.on_event("startup")
def on_startup():
    init_db()
```


3.Реализоваем файл с подключением к БД:
```
from sqlmodel import SQLModel, Session, create_engine
import os
from dotenv import load_dotenv

load_dotenv()
db_url = os.getenv('DB_ADMIN')
engine = create_engine(db_url, echo=True)


def init_db():
    SQLModel.metadata.create_all(engine)


def get_session():
    session = Session(engine)
    try:
        yield session
    finally:
        session.close()
```

И также передаем в alembic.ini URL базы данных с помощью .env-файла:
```
DB_ADMIN=postgresql+psycopg2://postgres:Masha1226@localhost:5432/team_platform_db
```
```
sqlalchemy.url = ${DB_ADMIN}
```


4.Переходим к постепенному созданию полноценного проекта. Начнем с авторизации пользвоателя (регистрация + вход)
Создаем форму регистрации:
```
from datetime import date
from enum import Enum
from typing import Optional, List
from sqlmodel import SQLModel, Field, Relationship


class TaskStatus(str, Enum):
    pending = "pending"
    in_progress = "in_progress"
    completed = "completed"


class ProjectStatus(str, Enum):
    planned = "planned"
    active = "active"
    finished = "finished"


class UserDefault(SQLModel):
    name: str
    date_of_birth: date
    about: Optional[str] = ""
    phone_number: str = Field(..., min_length=11, max_length=11)
    email: str
    password: str


class User(UserDefault, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    skills: List["UserSkill"] = Relationship(back_populates="user")
    positions: List["UserPosition"] = Relationship(back_populates="user")
    tasks: List["Task"] = Relationship(back_populates="user")
    participation: List["Participation"] = Relationship(back_populates="user")


class Skill(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    title: str


class UserSkillDefault(SQLModel):
    skill_id: int = Field(foreign_key="skill.id")
    user_id: int = Field(foreign_key="user.id")
    level: int = Field(ge=1, le=5)


class UserSkill(UserSkillDefault, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    user: Optional[User] = Relationship(back_populates="skills")
    skill: Optional[Skill] = Relationship()


class Position(SQLModel, table=True):
    id: Optional[int] = Field(default=None, primary_key=True)
    title: str
```


5.После создания моделей применяем миграции и смотри на базу данных в PostgreSQL:
![Alt текст](images/6.png)
![Alt текст](images/7.png)


6.Переходим к написанию CRUD-запросов:
```
from fastapi import APIRouter, HTTPException
from fastapi import Depends
from sqlalchemy.orm import joinedload
from db.connection import get_session
from endpoints.user_endpoints import auth_handler
from models.models import *
from models.read_models import *
from typing import List, TypedDict


team_platform_router = APIRouter(dependencies=[Depends(auth_handler.get_authenticated_user)])


@team_platform_router.post("/skill", tags=['Skill'], description='Создание нового навыка')
def skill_create(skill: Skill, session=Depends(get_session)) -> TypedDict('Response', {"status": int, "data": Skill}):
    skill = Skill.model_validate(skill)
    session.add(skill)
    session.commit()
    session.refresh(skill)
    return {"status": 200, "data": skill}


@team_platform_router.delete("/skill/delete{skill_id}", tags=['Skill'], description='Удаление навыка')
def skill_delete(skill_id: int, session=Depends(get_session)):
    skill = session.get(Skill, skill_id)
    name = skill.title
    if not skill:
        raise HTTPException(status_code=404, detail="Навык не обнаружен")
    session.delete(skill)
    session.commit()
    return {f"Навык {name} удален"}


@team_platform_router.patch("/skill/patch/{skill_id}", tags=['Skill'], description='Редактирование навыка')
def skill_update(skill_id: int, skill: Skill, session=Depends(get_session)) -> Skill:
    given_skill = session.get(Skill, skill_id)
    if not given_skill:
        raise HTTPException(status_code=404, detail="Навык не обнаружен")
    skill_data = skill.model_dump(exclude_unset=True)
    for key, value in skill_data.items():
        setattr(given_skill, key, value)
    session.add(given_skill)
    session.commit()
    session.refresh(given_skill)
    return given_skill


@team_platform_router.get("/skills_list", tags=['Skill'], description='Список всех навыков')
def skills_list(session=Depends(get_session)) -> List[Skill]:
    return session.query(Skill).all()
```
После создания запросов получаем следующее в разделе /docs от FastAPI:
![Alt текст](images/8.png)

Теперь протестируем готовые запросы:

- Skill:

- ![Alt текст](images/13.png)

- User Skill

![Alt текст](images/21.png)

- Position

![Alt текст](images/20.png)

![Alt текст](images/19.png)

- User Position

![Alt текст](images/18.png)

- Project

![Alt текст](images/17.png)

![Alt текст](images/16.png)

![Alt текст](images/11.png)

- Participation

![Alt текст](images/12.png)

- Task

![Alt текст](images/14.png)

![Alt текст](images/15.png)

- Более трудные запросы, а именно получение списка участников конкретного проекта, получение списка задач для пользователя

![Alt текст](images/9.png)

![Alt текст](images/10.png)


7.Следующая задача - реализовать аторизацию пользоателя по JWT-токенам. Создаем auth.py для реализации обработчика, а именно хэширование пароля, проверка пароля на корректность, кодировка и декодировка токена, получения пользователя по id и проверка на атентификацию:
```
import datetime
import os
from fastapi import Security, HTTPException
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from passlib.context import CryptContext
import jwt
from starlette import status
from dotenv import load_dotenv
from repos.user_repos import find_user


load_dotenv()


class AuthHandler:
    security = HTTPBearer()
    pwd_context = CryptContext(schemes=['bcrypt'])
    secret = os.getenv('JWT_SECRET')

    def hash_password(self, password):
        return self.pwd_context.hash(password)

    def verify_password(self, pwd, hashed_pwd):
        return self.pwd_context.verify(pwd, hashed_pwd)

    def encode_token(self, email):
        payload = {
            'expiration': (datetime.datetime.now() + datetime.timedelta(hours=12)).isoformat(),
            'created_at': datetime.datetime.now().isoformat(),
            'email': email
        }
        return jwt.encode(payload, self.secret, algorithm='HS256')

    def decode_token(self, token):
        try:
            payload = jwt.decode(token, self.secret, algorithms=['HS256'])
            print(payload['email'])
            return payload['email']
        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail='Срок действия токена истек')
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail='Неверный токен')

    def get_user_id_from_token(self, auth: HTTPAuthorizationCredentials = Security(security)):
        return self.decode_token(auth.credentials)

    def get_authenticated_user(self, auth: HTTPAuthorizationCredentials = Security(security)):
        credentials_exception = HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail='Не удалось аутентифицировать пользователя'
        )
        email = self.decode_token(auth.credentials)
        print(email)
        if email is None:
            raise credentials_exception
        user = find_user(email)
        if user is None:
            raise credentials_exception
        return user
```

Создаем JWT_SECRET (это секретный ключ, который используется для подписывания и валидации JWT токенов) и записываем его в файл .env
```
JWT_SECRET=my!MpAsSgHA!super!secret!
```

Также нам нужно обрабатывать юзера и сделать несколько проверок: вывод всех пользователей, поиск пользователя по email, создание пользователя и его обновление:
```
from sqlmodel import select
from db.connection import get_session
from models.models import User


def select_all_users():
    with next(get_session()) as session:
        statement = select(User)
        res = session.exec(statement).all()
        return res


def find_user(email):
    if not isinstance(email, str):
        raise ValueError(f"Expected a string for email, but got {type(email)}")
    print(f"Searching for user with email: {email}")
    with next(get_session()) as session:
        statement = select(User).where(User.email == email)
        return session.exec(statement).first()


def create_user(user_data):
    with next(get_session()) as session:
        user = User.model_validate(user_data)
        session.add(user)
        session.commit()
        session.refresh(user)
    return user


def update_user(user_id, user_data: dict):
    with next(get_session()) as session:
        user = session.get(User, user_id)
        for key, value in user_data.items():
            if hasattr(user, key):
                setattr(user, key, value)
        session.add(user)
        session.commit()
        session.refresh(user)
    return user
```


8.После первичной подготовки приступаем к реализации самих запросов в FastAPI:
```
@user_router.post("/user_registration", tags=['User'], description='Регистрация пользователя')
def register(user_data: UserCreate):
    existing_user = find_user(user_data.email)
    if existing_user:
        raise HTTPException(status_code=400, detail="Пользователь с такой почтой уже существует")
    hashed_password = auth_handler.hash_password(user_data.password)
    user_data.password = hashed_password
    user = create_user(user_data)
    return {"status": 200, "data": user}


@user_router.post("/user_ogin", tags=['User'], description='Вход')
def login(user_data: UserLogin):
    user = find_user(user_data.email)
    if not user or not auth_handler.verify_password(user_data.password, user.password):
        raise HTTPException(status_code=401, detail="Неверные данные для входа")
    token = auth_handler.encode_token(user.email)
    return {"token": token}


@user_router.get('/user_me', tags=['User'], description='Мои данные')
def get_current_user(user: User = Depends(auth_handler.get_authenticated_user)):
    return user
```

В результате для пользователя можно выполнять следующие запросы:

- Регистрация нового пользователя
- Вход в аккаунт
- Вывод информации о текущем пользователе
- Смена данных пользователя
- Смена пароля

Создадим нового пользователя:

![Alt текст](images/22.png)

![Alt текст](images/23.png)

Попробуем создать еще одного, но с такой же почтой:

![Alt текст](images/24.png)

Войдем в аккаунт и получим свой токен для авторизации и открытия доступа к другим запросам:

![Alt текст](images/25.png)

Далее авторизируемся по токену:

![Alt текст](images/27.png)

Как можно заметить, нам доступны все запросы:

![Alt текст](images/28.png)

Выведем свои данные после авторизации

![Alt текст](images/26.png)

Попробуем изменить пароль:

![Alt текст](images/29.png)

![Alt текст](images/30.png)


9.Сделаем все поля и запросы закрытыми для неаторизированного пользователя. Для этого пропишем следующую строчку в team_platform_endpoints:
```
team_platform_router = APIRouter(dependencies=[Depends(auth_handler.get_authenticated_user)])
```
Теперь на роутере стоит защита от незарегестрирвоанного пользователя


##Вывод: 
В рамках данной лабораторной работы я познакомилась с FastAPI, а также реализовала свое приложение для управлением платформы для поиска команд с подключением alembic для инициализации запросов к бд, dotenv для конфигурации необходимых данных и познакомилась с реализацией регистрации пользователей по JWT токенам.