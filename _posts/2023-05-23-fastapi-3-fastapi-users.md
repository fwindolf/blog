---
title: "Adventures into FastAPI backend (1/x): User Management with FastAPI Users"
date: 2023-05-09
---


# Using SQLModel instead of SQLAlchemy

After quite some time in the world of SQLAlchemy at work, I decided to try out the ORM in the FastAPI world. SQLModel combines SQLAlchemy and Pydantic into an ORM that behaves just like a class, but with the support of the SQLAlchemy ecosystem. 

##