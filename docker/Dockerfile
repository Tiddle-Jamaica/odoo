FROM python:3.12-bookworm

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1

RUN apt-get update && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        curl \
        git \
        gettext \
        libffi-dev \
        libfribidi-dev \
        libharfbuzz-dev \
        libjpeg-dev \
        libldap2-dev \
        libmagic1 \
        libopenjp2-7-dev \
        libpq-dev \
        libsasl2-dev \
        libssl-dev \
        libtiff-dev \
        libwebp-dev \
        libxcb1-dev \
        libxml2-dev \
        libxslt1-dev \
        nodejs \
        npm \
        postgresql-client \
        wkhtmltopdf \
        zlib1g-dev \
    && npm install -g less less-plugin-clean-css rtlcss \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /workspace/odoo

COPY requirements.txt /tmp/odoo-requirements.txt
RUN python -m pip install --upgrade pip wheel setuptools \
    && pip install -r /tmp/odoo-requirements.txt \
    && rm /tmp/odoo-requirements.txt

EXPOSE 8069 8072

CMD ["./odoo-bin", "-c", "docker/odoo-docker.conf", "-d", "odoo_dev", "--dev=xml,qweb,assets"]
