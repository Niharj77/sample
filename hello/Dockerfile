FROM python:3.12-alpine

WORKDIR /app
COPY app.py .

ENV PORT=8080
ENV APP_VERSION=v1

EXPOSE 8080

CMD ["python", "app.py"]
