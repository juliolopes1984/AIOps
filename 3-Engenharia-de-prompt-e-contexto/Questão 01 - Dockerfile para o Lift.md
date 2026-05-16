## Prompt:
```
# Role
Você é um SRE sênior especialista em containerização para Kubernetes

# Task
Escreva o Dockerfile para a API Lift, que foi desenvolvida em Python/Flask, roda na porta 8080, possui dependências no arquivo requirements.txt presente no diretório raiz do projeto, a API tem nome app.py e está no diretório raiz do projeto, o serviço deve ser exposto para todos os IPs, e roda utilizando o gunicorn, as variáveis DATABASE_URL e API_KEY precisam estar presentes no runtime. Siga as boas práticas de criação do Dockerfile

# Format
Retorne apenas o arquivo Dockerfile, sem explicações
```
---

## Modelo:
```
Gemini 3 Flash e Sonet 4.6
```
---

## Output Gemini 3 Flash:
```
FROM python:3.11-slim

#Build-time metadata
LABEL maintainer="SRE Team"
LABEL service="api-lift"

#Environment variables for Python and Runtime
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    DATABASE_URL="" \
    API_KEY="" \
    PORT=8080

#Create and set working directory
WORKDIR /app

#Install system dependencies (if needed by any packages in requirements.txt)
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

#Leverage Docker cache by copying only requirements first
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

#Copy application source code
COPY . .

#Create a non-privileged user for security (K8s best practice)
RUN useradd -m appuser && chown -R appuser /app
USER appuser

#Expose the port the app runs on
EXPOSE 8080

#Healthcheck to ensure the container is running correctly
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:8080/health || exit 1

#Execute gunicorn
#--workers: rule of thumb (2 x $num_cores) + 1
#--bind: 0.0.0.0 to listen on all interfaces
CMD ["gunicorn", "--workers", "4", "--bind", "0.0.0.0:8080", "app:app"]
```

## Output Sonet 4.6:
```
FROM python:3.12-slim

ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1

WORKDIR /app

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser

COPY requirements.txt .

RUN pip install --no-cache-dir --upgrade pip \
    && pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8080/health')" || exit 1

ARG DATABASE_URL
ARG API_KEY

ENV DATABASE_URL=${DATABASE_URL} \
    API_KEY=${API_KEY}

CMD ["gunicorn", "--bind", "0.0.0.0:8080", "--workers", "4", "--timeout", "120", "app:app"]
```
---

## Justificativa:
```
O framework RTF, utilizado nesta atividade é bem definida, onde a Role define o papel com o qual o desejo que a IA veja a questão (um SRE Sênior especialista em K8s) e a temperatura; a Task define o ambiente onde a API está sendo provisionada e características da API para guiar a criação do Dockerfile; o Format define como o output deve ser apresentado pela IA (apenas o arquivo Dockerfile sem explicações).
```
