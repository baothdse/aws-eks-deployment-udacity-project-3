FROM python:3.10-slim-buster

ARG DB_USERNAME=$DB_USERNAME
ARG DB_PASSWORD=$DB_PASSWORD

USER root

WORKDIR /app

COPY requirements.txt requirements.txt

# Dependencies are installed during build time in the container itself so we don't have OS mismatch
RUN pip install -r requirements.txt

COPY . /app

# Start the database and Flask application
CMD python app.py
