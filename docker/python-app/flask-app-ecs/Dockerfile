# base image
from python:3.14

workdir /app

copy . .

run pip install -r requirements.txt

expose 80

cmd ["python","run.py"]
