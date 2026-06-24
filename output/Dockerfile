FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt ./
RUN pip install --no-cache-dir \
    Flask==3.0.3 \
    redis==5.0.8 \
    gunicorn==22.0.0

COPY . .

RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser && \
    chown -R appuser:appgroup /app
USER appuser

EXPOSE 80

HEALTHCHECK --interval=30s --timeout=5s --start-period=15s --retries=3 \
  CMD python3 -c "import urllib.request; urllib.request.urlopen('http://localhost:80/')" || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:80", "--workers", "4", "--timeout", "60", "app:app"]
