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

---

## Código DRF inserido para a API funcionar

Esta seção explica o que foi adicionado ao projeto para expor a API REST, arquivo por arquivo.

---

### `sgc/sgc/settings.py` — Configuração do DRF e JWT

Dois blocos foram inseridos nas configurações do projeto.

**Registro do DRF e autenticação padrão:**

```python
# sgc/sgc/settings.py

INSTALLED_APPS = [
    ...
    'rest_framework',        # habilita o Django REST Framework
    'rest_framework.authtoken',  # habilita autenticação por Token estático
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_simplejwt.authentication.JWTAuthentication',  # JWT (principal)
        'rest_framework.authentication.TokenAuthentication',           # Token estático
        'rest_framework.authentication.BasicAuthentication',           # usuário/senha
    ),
}
```

`DEFAULT_AUTHENTICATION_CLASSES` define a ordem em que o DRF tenta autenticar cada requisição. O JWT é tentado primeiro; se não houver header `Authorization: Bearer`, tenta Token; depois Basic.

**Configuração dos tokens JWT (`SimpleJWT`):**

```python
import datetime

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': datetime.timedelta(minutes=2),   # token de acesso expira em 2 min
    'REFRESH_TOKEN_LIFETIME': datetime.timedelta(days=1),     # refresh válido por 1 dia
    'ROTATE_REFRESH_TOKENS': False,
    'BLACKLIST_AFTER_ROTATION': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,                                # assina com a SECRET_KEY do Django
    'AUTH_HEADER_TYPES': ('Bearer',),                         # exige "Authorization: Bearer <token>"
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',                               # nome da claim no payload do JWT
    'AUTH_TOKEN_CLASSES': ('rest_framework_simplejwt.tokens.AccessToken',),
}
```

---

### `sgc/sgc/urls.py` — Rotas de autenticação JWT e inclusão da API

As três rotas do SimpleJWT foram registradas no `urlpatterns` principal, apontando para views prontas da biblioteca:

```python
# sgc/sgc/urls.py

from rest_framework_simplejwt.views import (
    TokenObtainPairView,   # POST /api/token/ → gera access + refresh
    TokenRefreshView,      # POST /api/token/refresh/ → renova o access
    TokenVerifyView,       # POST /api/token/verify/ → valida um token
)

urlpatterns = [
    ...
    path('api/', include("api.urls", namespace="api")),          # rotas dos recursos
    path('api/token/', TokenObtainPairView.as_view()),           # autenticação
    path('api/token/refresh/', TokenRefreshView.as_view()),      # renovação
    path('api/token/verify/', TokenVerifyView.as_view()),        # verificação
]
```

`TokenObtainPairView`, `TokenRefreshView` e `TokenVerifyView` são views já implementadas pelo `djangorestframework-simplejwt` — não foi necessário escrever nenhuma lógica de autenticação.

---

### `sgc/api/serializers.py` — Serialização dos modelos

Os serializers convertem instâncias dos models Django em JSON (e validam JSON de entrada). Foram criadas três classes utilizadas pela API:

**`ProfessorSerializer`** — serializa o model `Professor` de `core/models.py`:

```python
class ProfessorSerializer(serializers.ModelSerializer):
    class Meta:
        model = Professor
        fields = ['nome', 'email']
```

Herda de `ModelSerializer`, que inspeciona o model automaticamente e gera os campos. Apenas `nome` e `email` são expostos (o campo `lattes` existe no model mas não é incluído).

**`ProjetoSerializerList`** — versão enxuta para a listagem:

```python
class ProjetoSerializerList(serializers.ModelSerializer):
    class Meta:
        model = Projeto
        fields = ['titulo', 'pk']
```

Retorna apenas título e chave primária, reduzindo o payload da listagem.

**`ProjetoSerializer`**: versão completa para o detalhe, com serializer aninhado:

```python
class ProjetoSerializer(serializers.ModelSerializer):
    coordenador = ProfessorSerializer(many=False, read_only=True)  # objeto aninhado

    class Meta:
        model = Projeto
        fields = ['pk', 'titulo', 'descricao', 'inicio', 'fim', 'aprovado', 'coordenador']
```

O campo `coordenador` é uma `ForeignKey` no model. Ao declarar `coordenador = ProfessorSerializer(...)`, o DRF serializa o objeto relacionado completo em vez de retornar apenas o ID. `read_only=True` impede que o campo seja usado em escritas.

---

### `sgc/api/views.py` — Views da API

As views definem o comportamento de cada endpoint. O DRF fornece classes genéricas que eliminam código repetitivo:

```python
# sgc/api/views.py

from rest_framework import generics
from rest_framework.permissions import IsAuthenticated
from .serializers import ProjetoSerializer, ProjetoSerializerList
from projeto.models import Projeto


class ProjetoListView(generics.ListAPIView):
    queryset = Projeto.objects.all()
    serializer_class = ProjetoSerializerList
```

`ListAPIView` responde a `GET` retornando uma lista de objetos. Sem `permission_classes`, usa o padrão do `settings.py` — que não define permissão padrão — então o endpoint é **público**.

```python
class ProjetoDetailView(generics.RetrieveAPIView):
    permission_classes = (IsAuthenticated,)
    queryset = Projeto.objects.all()
    serializer_class = ProjetoSerializer
```

`RetrieveAPIView` responde a `GET /api/projetos/<pk>` retornando um único objeto. `permission_classes = (IsAuthenticated,)` faz o DRF rejeitar requisições sem autenticação válida com `401 Unauthorized`.

---

### `sgc/api/urls.py`: Roteamento interno da app `api`

```python
# sgc/api/urls.py

from django.urls import path
from . import views

app_name = 'api'

urlpatterns = [
    path("projetos/", views.ProjetoListView.as_view(), name="api_projeto_list"),
    path("projetos/<pk>", views.ProjetoDetailView.as_view(), name="api_projeto_detail"),
]
```

`.as_view()` converte a classe em uma função de view compatível com o sistema de rotas do Django. O `app_name = 'api'` habilita o namespace, permitindo referenciar as rotas como `api:api_projeto_list` em outros pontos do projeto.

---
