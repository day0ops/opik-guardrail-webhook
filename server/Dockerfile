FROM python:3.13-slim-trixie AS build

COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# Symlink the distroless path for python (/usr/bin/python) to the build
# image's path (/usr/local/bin/python) so the venv built here has the right
# interpreter path baked in for the runtime image.
RUN ln -s /usr/local/bin/python /usr/bin/python && \
    /usr/bin/python -m venv /venv

WORKDIR /app
COPY requirements.txt .
RUN uv pip install --python /venv/bin/python --no-cache -r requirements.txt
RUN /venv/bin/python -c "import nltk; nltk.download('vader_lexicon', download_dir='/usr/local/nltk_data')"

FROM gcr.io/distroless/python3-debian13:nonroot

WORKDIR /app
COPY --from=build /venv /venv
# --chown so the nonroot user can write here: main.py's nltk.download() call
# at startup resolves its download_dir to the first *writable* entry in
# nltk.data.path, falling back past this directory (silently, to ~/nltk_data)
# and re-downloading over the network on every pod start if it isn't.
COPY --from=build --chown=65532:65532 /usr/local/nltk_data /usr/local/nltk_data
COPY main.py .
COPY webhook_api.py .

ENV NLTK_DATA=/usr/local/nltk_data

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD ["/venv/bin/python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/docs')"]

ENTRYPOINT ["/venv/bin/python", "-m", "uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
