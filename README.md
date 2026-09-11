# PzaaS Online — Serviço de Estoque `lorenzo_estoque`

Contrato de integração da API · v1

## 1. Identificação

| Campo | Valor |
|---|---|
| Serviço | Disponibilidade / Estoque |
| Identificação da dupla | `lorenzo_estoque` |
| Versão da API | `v1` |
| Ambiente | PzaaS Online — n8n compartilhado da turma |
| URL base | `https://pzaas.online/webhook` |
| Prefixo do serviço | `v1/estoque-242251` |
| Chave Redis | `lorenzo_estoque` |
| Banco | Redis compartilhado da turma |
| Formato persistido | String JSON com ingrediente e quantidade |

O Serviço de Estoque controla a disponibilidade dos ingredientes usados pela pizzaria. Ele consulta o estoque, verifica e consome ingredientes para um pedido e permite corrigir ou remover itens administrativamente.

## 2. URL base e endpoints

O n8n da turma registra os Webhooks globalmente. Por isso, o path exclusivo deste serviço é `v1/estoque-242251`.

### URLs de produção

```text
GET   https://pzaas.online/webhook/v1/estoque-242251
POST  https://pzaas.online/webhook/v1/estoque-242251
PATCH https://pzaas.online/webhook/v1/estoque-242251
GET   https://pzaas.online/webhook/health_242251
```

| Método | Endpoint | Função |
|---|---|---|
| GET | `/v1/estoque-242251` | Consulta o estoque disponível |
| POST | `/v1/estoque-242251` | Verifica e consome ingredientes de um pedido |
| PATCH | `/v1/estoque-242251` | Corrige quantidades ou remove ingredientes |
| GET | `/health_242251` | Verifica se o serviço está no ar |

> O JSON atual confirma que o GET, POST e PATCH usam o mesmo path `v1/estoque-242251`, diferenciados pelo método HTTP. O health usa o path `health_242251` [file:92].

### URLs de teste

Durante uma execução de teste no n8n, substitua `/webhook/` por `/webhook-test/`:

```text
GET   https://pzaas.online/webhook-test/v1/estoque-242251
POST  https://pzaas.online/webhook-test/v1/estoque-242251
PATCH https://pzaas.online/webhook-test/v1/estoque-242251
GET   https://pzaas.online/webhook-test/health_242251
```

A URL `webhook-test` somente funciona enquanto o workflow estiver aguardando uma execução de teste. Para a apresentação, use as URLs de produção com o workflow ativo.

## 3. Headers obrigatórios

### Healthcheck

O healthcheck não exige `x-api-key`:

```http
GET /health_242251
```

### Consulta, consumo e correção

Use:

```http
x-api-key: turma2026
```

Para POST e PATCH:

```http
Content-Type: application/json
```

Para operações associadas a um pedido existente, envie também:

```http
x-pedido-id: PED-yyyyMMddHHmmss-execucao
```

O `x-pedido-id` é gerado pelo Gateway ou Orquestrador e propagado entre os serviços que já possuem um pedido. Ele permite rastrear a solicitação ao longo do sistema distribuído [file:2][file:73].

## 4. Armazenamento

O serviço usa a chave Redis:

```text
lorenzo_estoque
```

O valor persistido é uma string JSON no formato:

```json
{
  "molho de tomate": 50,
  "mussarela": 50,
  "tomate": 50,
  "orégano": 50,
  "azeitona": 50,
  "calabresa": 50,
  "cebola roxa": 50
}
```

O node Redis de consulta faz `GET` nessa chave, o Code node interpreta a string JSON e a resposta pública acrescenta unidade e categoria aos ingredientes.

## 5. Healthcheck

### GET `/health_242251`

Verifica se o workflow do Serviço de Estoque está publicado e respondendo. Ele não acessa o Redis e não exige autenticação.

**Requisição:**

```bash
curl https://pzaas.online/webhook/health_242251
```

**Resposta 200:**

```json
{
  "health": true,
  "servico": "estoque",
  "status": "online"
}
```

Se o workflow estiver indisponível, a plataforma pode responder com `503 Service Unavailable`.

## 6. Consultar estoque

### GET `/v1/estoque-242251`

Retorna o estoque atual com quantidade, unidade e categoria.

**Header:**

```http
x-api-key: turma2026
```

**Exemplo:**

```bash
curl -H "x-api-key: turma2026" \
  https://pzaas.online/webhook/v1/estoque-242251
```

**Resposta 200:**

```json
{
  "molho de tomate": {
    "quantidade": 50,
    "unidade": "kg",
    "categoria": "mercearia"
  },
  "mussarela": {
    "quantidade": 50,
    "unidade": "kg",
    "categoria": "laticínios"
  },
  "tomate": {
    "quantidade": 50,
    "unidade": "unidades",
    "categoria": "hortifruti"
  },
  "orégano": {
    "quantidade": 50,
    "unidade": "gramas",
    "categoria": "temperos"
  },
  "azeitona": {
    "quantidade": 50,
    "unidade": "gramas",
    "categoria": "conservas"
  },
  "calabresa": {
    "quantidade": 50,
    "unidade": "kg",
    "categoria": "frios"
  },
  "cebola roxa": {
    "quantidade": 50,
    "unidade": "unidades",
    "categoria": "hortifruti"
  }
}
```

Metadados definidos pelo serviço:

| Ingrediente | Unidade | Categoria |
|---|---|---|
| `molho de tomate` | `kg` | `mercearia` |
| `mussarela` | `kg` | `laticínios` |
| `tomate` | `unidades` | `hortifruti` |
| `orégano` | `gramas` | `temperos` |
| `azeitona` | `gramas` | `conservas` |
| `calabresa` | `kg` | `frios` |
| `cebola roxa` | `unidades` | `hortifruti` |

Ingredientes sem metadados recebem `unidade: "und"` e `categoria: "outros"`.

## 7. Consumo de estoque

### POST `/v1/estoque-242251`

Verifica se os ingredientes necessários estão disponíveis e realiza a baixa no Redis quando houver estoque suficiente.

O body atual do fluxo usa o sabor da pizza e a quantidade:

```json
{
  "sabor": "mussarela",
  "quantidade": 1
}
```

O campo `pizza` também pode ser aceito pelo código como alternativa a `sabor`.

### Receitas implementadas

| Sabor | Ingredientes consumidos por pizza |
|---|---|
| `mussarela` | molho de tomate, mussarela, tomate, orégano, azeitona |
| `calabresa acebolada` | molho de tomate, calabresa, cebola roxa, orégano, azeitona |

A quantidade deve ser maior ou igual a `1`. Se não for enviada, o fluxo usa `1`.

### Exemplo de chamada

```bash
curl -X POST \
  https://pzaas.online/webhook/v1/estoque-242251 \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -H "x-pedido-id: PED-20260910123000-001" \
  -d '{"sabor":"mussarela","quantidade":1}'
```

### Resposta 200

```json
{
  "status": "sucesso",
  "mensagem": "Estoque atualizado e ingredientes separados."
}
```

### Resposta 400

Quando o sabor for inválido, a quantidade for inválida ou faltar algum ingrediente:

```json
{
  "erro": "Estoque insuficiente ou pedido inválido",
  "detalhes": [
    "mussarela",
    "tomate"
  ]
}
```

A baixa só ocorre no branch de sucesso. Quando o estoque é insuficiente, o Redis não deve ser atualizado.

## 8. Corrigir ou remover itens

### PATCH `/v1/estoque-242251`

Permite substituir quantidades ou remover ingredientes. O PATCH não soma: ele define o valor final informado.

### Corrigir quantidade

```json
{
  "tomate": 50,
  "mussarela": 40
}
```

### Remover ingrediente

Envie `null` como valor:

```json
{
  "azeitona": null
}
```

Isso remove somente o ingrediente do JSON; a chave Redis `lorenzo_estoque` continua existindo.

### Exemplo de chamada

```bash
curl -X PATCH \
  https://pzaas.online/webhook/v1/estoque-242251 \
  -H "Content-Type: application/json" \
  -H "x-api-key: turma2026" \
  -d '{"tomate":50,"azeitona":null}'
```

### Resposta 200 esperada

```json
{
  "sucesso": true,
  "acao": "estoque_corrigido",
  "mensagem": "Estoque atualizado com sucesso"
}
```

### Respostas de erro

- `400` se o body estiver vazio ou houver quantidade inválida.
- `404` se o ingrediente não existir no estoque.
- `500` se não for possível interpretar o JSON do Redis.

## 9. Integração com o Logger

O projeto exige o envio de logs estruturados ao Serviço 09, preferencialmente de forma assíncrona [file:2]. O contrato do Gateway recomenda os campos `servico`, `versao`, `pedidoId`, `nivel`, `mensagem`, `payload` e `timestamp` [file:73].

### Endpoint do Logger

```text
POST https://pzaas.online/webhook/logger241425:v1/logs
```

### Headers do Logger

```http
Content-Type: application/json
x-api-key: turma2026
x-pedido-id: <identificador-não-vazio>
```

O README do Logger informa que `x-pedido-id` é obrigatório e não pode ser vazio em `/v1/logs` [file:72].

### Log de consulta

```json
{
  "servico": "estoque",
  "versao": "v1",
  "pedidoId": "consulta-sem-pedido",
  "nivel": "info",
  "mensagem": "Consulta de estoque realizada",
  "payload": {
    "operacao": "consultar_estoque"
  },
  "timestamp": "2026-09-10T03:00:00.000Z"
}
```

### Log de consumo aprovado

```json
{
  "servico": "estoque",
  "versao": "v1",
  "pedidoId": "PED-20260910123000-001",
  "nivel": "info",
  "mensagem": "Baixa de estoque realizada",
  "payload": {
    "operacao": "consumir",
    "sabor": "mussarela",
    "quantidade": 1
  },
  "timestamp": "2026-09-10T03:00:00.000Z"
}
```

### Log de consumo rejeitado

```json
{
  "servico": "estoque",
  "versao": "v1",
  "pedidoId": "PED-20260910123000-001",
  "nivel": "warn",
  "mensagem": "Estoque insuficiente ou pedido inválido",
  "payload": {
    "operacao": "consumir",
    "faltantes": [
      "tomate",
      "mussarela"
    ]
  },
  "timestamp": "2026-09-10T03:00:00.000Z"
}
```

### Log de correção manual

```json
{
  "servico": "estoque",
  "versao": "v1",
  "pedidoId": "operacao-manual",
  "nivel": "info",
  "mensagem": "Estoque corrigido manualmente",
  "payload": {
    "operacao": "corrigir_estoque",
    "itensCorrigidos": [
      {
        "ingrediente": "tomate",
        "quantidade": 50
      }
    ],
    "itensRemovidos": [
      "azeitona"
    ]
  },
  "timestamp": "2026-09-10T03:00:00.000Z"
}
```

Os nodes de log devem estar em paralelo à resposta principal, depois da operação correspondente, para que uma falha no Logger não impeça a resposta do serviço. O fluxo recomendado é:

```text
Operação principal
├── Respond to Webhook
└── HTTP Request para o Logger
```

## 10. Códigos de erro

| Código | Situação |
|---:|---|
| 200 | Operação concluída |
| 400 | Sabor/quantidade inválida ou estoque insuficiente |
| 401 | `x-api-key` ausente ou inválida |
| 404 | Ingrediente não encontrado durante correção |
| 500 | Erro interno ou JSON inválido no Redis |
| 503 | Serviço indisponível |

O formato global do projeto utiliza respostas HTTP/JSON e prevê `503` para indisponibilidade após tentativas de resiliência [file:2].

## 11. Testes no Postman

### Healthcheck

```text
GET https://pzaas.online/webhook/health_242251
```

Sem body e sem `x-api-key`.

### Consulta

```text
GET https://pzaas.online/webhook/v1/estoque-242251
```

Header:

```text
x-api-key: turma2026
```

### Consumo aprovado

```text
POST https://pzaas.online/webhook/v1/estoque-242251
```

Headers:

```text
Content-Type: application/json
x-api-key: turma2026
x-pedido-id: PED-ESTOQUE-TESTE-001
```

Body:

```json
{
  "sabor": "mussarela",
  "quantidade": 1
}
```

### Consumo rejeitado

```json
{
  "sabor": "sabor inexistente",
  "quantidade": 1
}
```

O serviço deve responder com erro de pedido inválido e registrar o evento no Logger.

### Correção/remover

```text
PATCH https://pzaas.online/webhook/v1/estoque-242251
```

Headers:

```text
Content-Type: application/json
x-api-key: turma2026
```

Body:

```json
{
  "tomate": 50,
  "azeitona": null
}
```

## 12. Integração com outros serviços

O Gateway/Orquestrador que consumir o Serviço de Estoque deve utilizar a URL base publicada:

```text
https://pzaas.online/webhook
```

E combinar com o endpoint correspondente:

```text
POST https://pzaas.online/webhook/v1/estoque-242251
```

Para um pedido já criado, deve propagar:

```http
x-api-key: turma2026
x-pedido-id: PED-...
Content-Type: application/json
```

O corpo enviado ao consumo deve conter `sabor` e `quantidade`, conforme descrito na seção 7.

## 13. Observações de implementação

- A chave Redis usada em todos os nodes é `lorenzo_estoque`.
- O Redis é compartilhado; os nodes devem usar uma credencial que esteja autorizada na instância n8n.
- A consulta interpreta tanto o campo Redis padrão quanto `propertyName`, `estoque`, `data` ou `value` para suportar as diferentes configurações do node.
- O Logger aceita JSON livre, mas os exemplos desta documentação seguem o contrato recomendado pelo Gateway.
- A documentação deve ser publicada em uma URL acessível e seu link deve ser colocado no workflow e na planilha da turma, conforme as instruções do projeto [file:2].

## Chaos Monkey — falha controlada 503

O serviço de estoque possui um **Chaos Monkey** para simular uma indisponibilidade controlada. Quando a falha `503` está ativa, o endpoint de consumo interrompe a operação antes de consultar ou alterar o estoque e retorna `503 Service Unavailable`.

A falha é controlada por uma chave separada no Redis:

```text
Chave do estoque: lorenzoestoque
Chave do Chaos Monkey: chaos:lorenzoestoque
```

A chave `chaos:lorenzoestoque` não altera os ingredientes; ela somente define se o serviço de consumo deve ficar indisponível.

### Endpoint de configuração

```http
POST /webhook/v1/estoque-242251/chaos-monkey
```

#### Headers obrigatórios

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `x-api-key` | `turma2026` |

### Ativar falha 503

Envie:

```json
{
  "tipo_falha": 503
}
```

Quando `tipo_falha` é `503`, o workflow executa:

```text
Redis Set
Key: chaos:lorenzoestoque
Value: 503
```

Resposta esperada:

```json
{
  "status": "CHAOS_ATIVADO",
  "tipo_falha": 503
}
```

### Desativar falha

Envie para o mesmo endpoint:

```json
{
  "tipo_falha": 0
}
```

O workflow remove a chave de falha:

```text
Redis Delete
Key: chaos:lorenzoestoque
```

Resposta esperada:

```json
{
  "status": "CHAOS_DESATIVADO",
  "tipo_falha": 0
}
```

### Validação no consumo

Antes de acessar o estoque real, o endpoint de consumo consulta a chave de Chaos:

```text
Webhook Consumir (POST)
→ Valida x-api-key
→ VERIFICA CHAOS ESTOQUE
→ CHAOS 503 ATIVO?
   ├── True  → RETORNA 503 CHAOS
   └── False → Puxa Estoque Atual
              → Analisa Receita e Estoque
              → Tem Estoque?
```

O node `VERIFICA CHAOS ESTOQUE` utiliza:

```text
Operation: Get
Key: chaos:lorenzoestoque
Property Name: chaos_status
```

O node `CHAOS 503 ATIVO?` compara:

```javascript
{{ String($json.chaos_status ?? '').trim() }}
```

com:

```text
503
```

Quando o Chaos Monkey está ativo, o consumo responde:

```http
HTTP/1.1 503 Service Unavailable
```

```json
{
  "erro": "SERVICO_INDISPONIVEL",
  "servico": "estoque",
  "identificacao": "lorenzoestoque",
  "motivo": "CHAOS_MONKEY",
  "mensagem": "Serviço de estoque temporariamente indisponível por falha controlada."
}
```

Enquanto a falha estiver ativa, os nodes `Puxa Estoque Atual`, `Analisa Receita e Estoque` e `Baixa no Redis` não são executados. Portanto, nenhum ingrediente é consumido durante a indisponibilidade simulada.

### Teste no Postman

Ative a falha:

```http
POST [https://pzaas.online/webhook/v1/estoque-242251/chaos-monkey](https://pzaas.online/webhook/v1/estoque-242251/chaos-monkey)
Content-Type: application/json
x-api-key: turma2026
```

```json
{
  "tipo_falha": 503
}
```

Em seguida, tente consumir:

```http
POST [https://pzaas.online/webhook/v1/estoque-242251](https://pzaas.online/webhook/v1/estoque-242251)
Content-Type: application/json
x-api-key: turma2026
x-pedido-id: pedido-chaos-001
```

```json
{
  "sabor": "mussarela",
  "quantidade": 1
}
```

Resultado esperado: `503 Service Unavailable`.

Para normalizar o serviço, envie:

```json
{
  "tipo_falha": 0
}
```
