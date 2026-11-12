# API DevOps de Tarefas

API em Python (FastAPI) usada como projeto da disciplina **Integracao DevOps**. O objetivo e demonstrar uma esteira completa de CI/CD: a cada alteracao aprovada na `main`, a aplicacao e testada, empacotada em uma imagem Docker, publicada no Amazon ECR e implantada automaticamente em uma instancia Amazon EC2.

## Tecnologias

- Python 3.13 e FastAPI (API) com Uvicorn
- Pytest, pytest-cov e Ruff (testes e qualidade)
- Docker (containerizacao)
- GitHub Actions (CI/CD)
- Amazon ECR (registro privado de imagens)
- Amazon EC2 com AWS Systems Manager (SSM) para executar o deploy

## Execucao local

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt
uvicorn app.main:app --reload
pytest
```

A API fica em `http://127.0.0.1:8000` e a documentacao interativa em `/docs`.

## Docker

```bash
docker build -t devops-python-api:local .
docker run -d --name devops-python-api -p 8000:8000 devops-python-api:local
curl http://127.0.0.1:8000/health
docker rm -f devops-python-api
```

Tambem e possivel usar `docker compose up --build`.

## Integracao continua (`.github/workflows/ci.yml`)

Executa em Pull Requests para a `main` (e tambem em pushes na `main`):

1. Baixa o codigo, configura o Python e instala as dependencias.
2. Roda `ruff check` e `ruff format --check`.
3. Roda os testes com `pytest` e gera cobertura.
4. Constroi a imagem Docker e testa o endpoint `/health` do container.

O GitHub informa na pagina do Pull Request se a execucao passou ou falhou.

## Entrega continua (`.github/workflows/deploy.yml`)

Executa a cada push na `main` (ou seja, apos o merge de um Pull Request) com tres jobs encadeados:

1. **test**: instala as dependencias e executa o `pytest`.
2. **build-and-push** (so roda se `test` passar): autentica na AWS, faz login no ECR, constroi a imagem e envia para o ECR com a tag igual ao SHA do commit.
3. **deploy** (so roda se `build-and-push` passar): usa o AWS Systems Manager para executar na EC2 o login no ECR, o `docker pull` da imagem do commit, a troca do container anterior pelo novo e a verificacao de que ele esta em execucao e responde em `/health`.

Se os testes falharem, nenhuma imagem e publicada e nenhum deploy acontece. Ninguem precisa acessar a EC2 manualmente.

### Credenciais

As credenciais da AWS ficam em **GitHub Secrets** (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` e `AWS_SESSION_TOKEN`) e nunca sao versionadas. A EC2 le imagens do ECR por meio de uma **IAM Role** associada a instancia.

## Arquitetura de implantacao

```
Desenvolvedor -> branch -> Pull Request (CI) -> merge na main
                                                   |
                                          GitHub Actions (deploy.yml)
                                   test -> build -> push (tag = SHA do commit)
                                                   |
                                          Amazon ECR (repositorio privado)
                                                   |
                          AWS SSM (Run Command) -> EC2 (Docker) <- IAM Role (le ECR)
                                                   |
                                      Container publicado na porta 80
```

## Acesso a API publicada

- URL: `http://IP-PUBLICO-DA-EC2/` (preenchida apos o primeiro deploy)
- Documentacao: `http://IP-PUBLICO-DA-EC2/docs`

## Como verificar a versao implantada

```bash
curl http://IP-PUBLICO-DA-EC2/health
```

A resposta traz o campo `version`, que muda quando o codigo e atualizado e implantado.

## Evidencias da execucao das pipelines

Os links dos Pull Requests e das execucoes do GitHub Actions estao nas abas **Pull requests** e **Actions** deste repositorio.
