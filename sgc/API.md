# API do SGC

API REST construída com **Django REST Framework** e autenticação via **SimpleJWT**.

---

## Autenticação

A API suporta três métodos de autenticação:

| Método | Header |
|--------|--------|
| JWT (recomendado) | `Authorization: Bearer <access_token>` |
| Token | `Authorization: Token <token>` |
| Basic | `Authorization: Basic <base64(usuario:senha)>` |

O login aceita **username** ou **e-mail** (via backend customizado em `core/authentication.py`).

---

## Fluxo JWT

### 1. Obter tokens

```
POST /api/token/
Content-Type: application/json
```

```json
{
  "username": "seu_usuario",
  "password": "sua_senha"
}
```

**Resposta 200:**
```json
{
  "access": "eyJ0eXAiOiJKV1QiLCJhbGci...",
  "refresh": "eyJ0eXAiOiJKV1QiLCJhbGci..."
}
```

> O `access` token expira em **2 minutos**. O `refresh` token é válido por **1 dia**.

### 2. Usar o token nas requisições

```
GET /api/projetos/1
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGci...
```

### 3. Renovar o access token

```
POST /api/token/refresh/
Content-Type: application/json
```

```json
{
  "refresh": "eyJ0eXAiOiJKV1QiLCJhbGci..."
}
```

**Resposta 200:**
```json
{
  "access": "eyJ0eXAiOiJKV1QiLCJhbGci..."
}
```

### 4. Verificar validade de um token

```
POST /api/token/verify/
Content-Type: application/json
```

```json
{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGci..."
}
```

Retorna `200 OK` se válido ou `401 Unauthorized` se expirado/inválido.

---

## Endpoints

### Tabela geral

| Método | URL | Autenticação | Descrição |
|--------|-----|:------------:|-----------|
| `POST` | `/api/token/` | Não | Obter access + refresh token |
| `POST` | `/api/token/refresh/` | Não | Renovar access token |
| `POST` | `/api/token/verify/` | Não | Verificar validade do token |
| `GET` | `/api/projetos/` | Não | Listar todos os projetos |
| `GET` | `/api/projetos/<pk>` | **Sim** | Detalhes de um projeto |

---

### `GET /api/projetos/`

Lista todos os projetos cadastrados. Acesso público, sem autenticação.

**Exemplo com curl:**
```bash
curl http://localhost:8000/api/projetos/
```

**Resposta 200:**
```json
[
  { "pk": 1, "titulo": "Projeto IoT" },
  { "pk": 2, "titulo": "Projeto IA na Saúde" }
]
```

---

### `GET /api/projetos/<pk>`

Retorna os detalhes completos de um projeto. **Requer autenticação.**

**Exemplo com curl:**
```bash
curl http://localhost:8000/api/projetos/1 \
  -H "Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGci..."
```

**Resposta 200:**
```json
{
  "pk": 1,
  "titulo": "Projeto IoT",
  "descricao": "Descrição detalhada do projeto.",
  "inicio": "2023-01-15",
  "fim": "2023-12-31",
  "aprovado": true,
  "coordenador": {
    "nome": "Prof. João Silva",
    "email": "joao.silva@ifpe.edu.br"
  }
}
```

**Resposta 401 (sem token ou token inválido):**
```json
{
  "detail": "Authentication credentials were not provided."
}
```

---

## Modelos expostos

### Projeto

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `pk` | inteiro | Identificador único |
| `titulo` | string | Título do projeto (máx. 50 chars) |
| `descricao` | string | Descrição detalhada (máx. 500 chars) |
| `inicio` | data (`YYYY-MM-DD`) | Data de início |
| `fim` | data (`YYYY-MM-DD`) | Data de término (opcional) |
| `aprovado` | booleano | Se o projeto está institucionalizado |
| `coordenador` | objeto | Dados do professor coordenador |

### Professor (aninhado em Projeto)

| Campo | Tipo | Descrição |
|-------|------|-----------|
| `nome` | string | Nome do professor |
| `email` | string | E-mail institucional |

---

## Exemplo completo de fluxo

```bash
# 1. Autenticar e obter tokens
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "admin123"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['access'])")

# 2. Listar projetos (público)
curl http://localhost:8000/api/projetos/

# 3. Ver detalhes de um projeto (autenticado)
curl http://localhost:8000/api/projetos/1 \
  -H "Authorization: Bearer $TOKEN"
```
