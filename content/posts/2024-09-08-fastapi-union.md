---
title: "FastAPI and OpenAPI with Union type and oneOf"
date: 2024-09-08T00:00:00+00:00
tags:
  - fastapi
  - openapi
---

```python
# .python-version
# 3.12.0
#
# requirements.txt
# fastapi[standard]==0.114.0
#
# main.py
from typing import Literal, Union
from pydantic import BaseModel, Field

from fastapi import FastAPI

app = FastAPI()


class Cat(BaseModel):
    pet_type: Literal["c"]
    meows: int


class Dog(BaseModel):
    pet_type: Literal["d"]
    barks: float


class Pet(BaseModel):
    pet: Union[Cat, Dog] = Field(..., discriminator="pet_type")
    age: int


@app.post("/cat")
def post_cat(pet: Cat):
    return {"pet": pet}


@app.post("/dog")
def post_dog(pet: Dog):
    return {"pet": pet}


@app.post("/pet")
def post_pet(pet: Pet):
    return {"pet": pet}
```
