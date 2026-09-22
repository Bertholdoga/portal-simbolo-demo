# Arquitetura — Leitura Automática de Comprovativos (OCR/IA)
### Portal Símbolo² · Módulo de Despesas

Documento de arquitetura técnica. **Não implementa código** e **não altera o protótipo existente** (`prototipo/index.html`). Define a solução a construir nas fases seguintes.

---

## 1. Estado atual

O protótipo (`prototipo/index.html`) tem hoje uma **simulação 100% client-side**, sem rede nem backend:

- O upload é feito por `<input type="file">` (clique) ou drag-and-drop para a `dropzone`.
- Ao receber um ficheiro, `handleReceiptFile()` mostra o estado "A ler o comprovativo…" e chama `setTimeout(...)` para simular latência (1.1–1.6s).
- `extractReceiptData(file)` **não lê o conteúdo do ficheiro**. Escolhe um "cenário" de um array fixo (`receiptScenarios`) com base em palavras-chave no **nome do ficheiro** (ex. `combustivel.jpg` → tipo "Combustível"), ou aleatoriamente se não encontrar pistas. O único dado real extraído do ficheiro é `file.lastModified`, usado como sugestão de data quando plausível.
- `applyReceiptData()` preenche `d-data`, `d-tipo`, `d-doc`, `d-valor`, marca os campos com a classe `just-filled` e a etiqueta "Detetado", e mostra o aviso para o utilizador rever antes de submeter.
- Não existe leitura de imagem/PDF, não existe chamada a nenhum serviço externo, não existe armazenamento do ficheiro, e não existe validação de tipo de ficheiro além do atributo `accept="image/*,.pdf"` do input (que é apenas uma sugestão no browser, não uma validação real).

Isto foi construído deliberadamente assim porque o protótipo corre num sandbox de artifact sem acesso a rede em runtime — serviu para validar o fluxo de UX, não a extração de dados.

---

## 2. Arquitetura proposta

```
┌─────────────┐      1. upload multipart       ┌──────────────────────┐
│  Frontend   │ ──────────────────────────────▶ │   API Backend         │
│ (Nova       │                                  │  (orquestrador)       │
│  Despesa)   │                                  └──────────┬────────────┘
└─────┬───────┘                                             │
      │                                          2. valida ficheiro
      │                                          (tipo real, tamanho,
      │                                           antivírus)
      │                                                      ▼
      │                                          ┌──────────────────────┐
      │                                          │ Storage temporário     │
      │                                          │ (encriptado, TTL curto)│
      │                                          └──────────┬────────────┘
      │                                                      │ 3. envia para leitura
      │                                                      ▼
      │                                          ┌──────────────────────┐
      │                                          │ OCR / Vision AI        │
      │                                          │ (texto + layout bruto) │
      │                                          └──────────┬────────────┘
      │                                                      │ 4. texto/campos brutos
      │                                                      ▼
      │                                          ┌──────────────────────┐
      │                                          │ Classificação           │
      │                                          │ (mapeia para enum do   │
      │                                          │  Símbolo²: movement_   │
      │                                          │  type, fuel_type)      │
      │                                          └──────────┬────────────┘
      │                                                      │ 5. dados classificados
      │                                                      ▼
      │                                          ┌──────────────────────┐
      │                                          │ Normalização            │
      │                                          │ (datas, moeda,          │
      │                                          │  matrícula, nº doc.)    │
      │                                          └──────────┬────────────┘
      │                                                      │ 6. dados normalizados
      │                                                      ▼
      │                                          ┌──────────────────────┐
      │                                          │ Validação                │
      │                                          │ (enums permitidos,       │
      │                                          │  ranges, consistência)   │
      │                                          └──────────┬────────────┘
      │             7. JSON estruturado                     │
      │◀─────────────────────────────────────────────────────┘
      ▼
┌─────────────┐
│  Frontend   │  8. preenche formulário por confiança
│  (revisão   │  9. utilizador corrige/confirma
│   humana)   │  10. "Submeter despesa" → POST /api/expenses
└─────────────┘
```

Ponto-chave: **nenhuma etapa entre 1 e 7 escreve na tabela de despesas**. Só o passo 10, disparado por ação explícita do utilizador, cria a despesa real.

---

## 3. Responsabilidade de cada componente

| Componente | Responsabilidade | Não faz |
|---|---|---|
| **Frontend** | Upload, estados de UX, renderizar dados por nível de confiança, permitir edição total, bloquear submissão automática | Não interpreta nem valida os dados extraídos — só apresenta o que o backend devolve |
| **API Backend (orquestrador)** | Autenticação/autorização, validação do ficheiro, coordenar OCR → classificação → normalização → validação, persistir a despesa só no submit final | Não confia cegamente na resposta do OCR/IA |
| **Storage temporário** | Guardar o ficheiro original só durante o processamento, encriptado, com TTL curto | Não é o armazenamento definitivo do comprovativo aprovado |
| **OCR / Vision** | Ler texto e layout do documento; devolver campos brutos + confiança nativa quando disponível | Não decide o `movement_type` nem valida contra o sistema |
| **Classificação** | Mapear texto/contexto extraído para os enums internos do Símbolo² (`movement_type`, `fuel_type`) | Não lê a imagem diretamente — trabalha sobre o texto já extraído |
| **Normalização** | Formatar datas (ISO 8601), moeda, matrícula (`AA-00-AA`), nº de documento | Não decide se o valor é "correto", só o formato |
| **Validação** | Rejeitar/assinalar valores fora dos enums permitidos, fora de ranges plausíveis, ou inconsistentes | Não corrige silenciosamente — sinaliza com `warnings` |
| **Persistência (submit)** | Guardar a despesa definitiva só após confirmação humana | Nunca é chamada pelo pipeline de análise |

---

## 4. Contrato da API

Processamento **assíncrono** (ver justificação na secção 6/Processamento). Dois endpoints principais, mais um de limpeza:

### 4.1 Upload + pedido de análise

```
POST /api/expenses/receipts
Content-Type: multipart/form-data
Authorization: Bearer <token>

file: <ficheiro>          (obrigatório)
```

**201 Created**
```json
{
  "receipt_id": "6f2e1c3a-...",
  "status": "queued",
  "accepted_mime_type": "image/jpeg",
  "size_bytes": 842113,
  "created_at": "2026-09-21T10:15:30Z"
}
```

**400 / 413 / 415** para ficheiro inválido, demasiado grande, ou tipo não suportado (ver secção 8).

### 4.2 Consultar resultado da análise

```
GET /api/expenses/receipts/{receipt_id}/analysis
Authorization: Bearer <token>
```

**200 OK — ainda em processamento**
```json
{ "receipt_id": "6f2e1c3a-...", "status": "processing" }
```

**200 OK — concluído** → devolve o schema completo (secção 5).

**200 OK — falhou**
```json
{
  "receipt_id": "6f2e1c3a-...",
  "status": "failed",
  "error": { "code": "OCR_UNAVAILABLE", "message": "Serviço de leitura indisponível. Pode preencher os campos manualmente." }
}
```

O frontend faz *polling* curto (ex. a cada 1.5s, até ~20s) ou, preferencialmente, recebe o resultado por **Server-Sent Events**/WebSocket no mesmo `receipt_id` — evita polling desnecessário e dá feedback mais imediato.

### 4.3 Descartar comprovativo não submetido

```
DELETE /api/expenses/receipts/{receipt_id}
```

Chamado quando o utilizador fecha o modal sem submeter, ou substitui o ficheiro (`Substituir`). Força a remoção imediata do storage temporário em vez de esperar pelo TTL.

### 4.4 Submissão final (já existente conceptualmente, sem alterações de fundo)

```
POST /api/expenses
```

Corpo = campos do formulário **tal como o utilizador os confirmou** (não o JSON do OCR diretamente), mais `receipt_id` como referência para associar o ficheiro definitivo à despesa. O backend volta a validar tudo aqui, independentemente do que a análise sugeriu — este é o único endpoint com efeitos persistentes.

**Porque separar `/receipts` de `/expenses`:** a análise pode ser repetida, descartada ou nunca chegar a ser submetida (utilizador desiste, troca de ficheiro, corrige tudo à mão). Misturar os dois num único endpoint tornaria impossível ter um estado "analisado mas não confirmado" — que é exactamente o requisito central desta tarefa (a IA nunca submete sozinha).

---

## 5. Schema JSON

Versão refinada da proposta inicial — mantém a estrutura pedida e acrescenta o que falta para suportar confiança por campo, ambiguidade, auditoria e o ciclo de vida assíncrono:

```json
{
  "receipt_id": "6f2e1c3a-9b7d-4e2a-9f31-2d4b6e8a1c00",
  "status": "done",

  "engine": {
    "ocr_provider": "vendor-x",
    "model_version": "2026-08-01",
    "processed_at": "2026-09-21T10:15:33Z",
    "processing_time_ms": 2140
  },

  "document": {
    "date": "2026-09-21",
    "document_number": "FT 4471",
    "total": 62.30,
    "currency": "EUR",
    "supplier": "Exemplo Combustíveis, Lda.",
    "supplier_tax_id": "500000000",
    "page_count": 1
  },

  "expense": {
    "movement_type": "Combustível",
    "fuel_type": "Gasolina simples 95",
    "vehicle_plate": "AS-13-UE",
    "mileage": null
  },

  "confidence": {
    "date": 0.98,
    "document_number": 0.94,
    "total": 0.99,
    "currency": 0.99,
    "supplier": 0.72,
    "supplier_tax_id": 0.55,
    "movement_type": 0.91,
    "fuel_type": 0.88,
    "vehicle_plate": 0.61,
    "mileage": null
  },

  "ambiguities": [
    {
      "field": "total",
      "candidates": [62.30, 6.30],
      "reason": "dois valores monetários detetados no documento"
    }
  ],

  "warnings": [
    { "code": "LOW_CONFIDENCE_FIELD", "field": "vehicle_plate", "message": "Matrícula com confiança abaixo do limite de preenchimento automático." },
    { "code": "SUPPLIER_TAX_ID_UNVERIFIED", "field": "supplier_tax_id", "message": "NIF lido mas não validado por checksum." }
  ],

  "auto_filled_fields": ["date", "document_number", "total", "currency", "movement_type", "fuel_type"],
  "review_suggested_fields": ["supplier", "vehicle_plate"],
  "unresolved_fields": ["mileage"],

  "requires_review": true
}
```

**Porque estas mudanças em relação à proposta original:**

- `receipt_id` + `status` + `engine` — necessários para o modelo assíncrono e para auditoria/observabilidade (secção "Observabilidade"). Sem isto não é possível saber *quando* e *com que versão* de modelo um campo foi lido, o que dificulta investigar erros reportados semanas depois.
- `ambiguities` — separa explicitamente o caso "encontrei dois valores plausíveis" (ex. total da fatura vs. total de um item) do caso "não encontrei nada". Sem isto, um documento com duas datas ou dois totais forçaria uma escolha silenciosa, sem sinal de que houve ambiguidade.
- `warnings` com `code` — os códigos permitem tratamento programático no frontend (ex. `SUPPLIER_TAX_ID_UNVERIFIED` pode mostrar um ícone diferente de `LOW_CONFIDENCE_FIELD`), em vez de o frontend ter de fazer *parsing* de texto livre.
- `auto_filled_fields` / `review_suggested_fields` / `unresolved_fields` — o frontend não devia ter de recalcular isto a partir dos números de confiança; o backend já aplicou os thresholds (secção 6) e devolve a decisão pronta a usar. Os números de `confidence` continuam disponíveis para quem quiser mostrar mais detalhe (ex. tooltip).
- Omiti deliberadamente qualquer campo com o texto bruto/completo do OCR (`raw_text`) na resposta que chega ao frontend — ver secção "Privacidade": o texto integral do documento não deve percorrer mais camadas do que o estritamente necessário.

---

## 6. Sistema de confiança

Confiança **por campo**, não global — o documento já reflete isso. Proposta de thresholds:

| Faixa | Comportamento | Sinal visual |
|---|---|---|
| **≥ 0.85** (alta) | Preenche automaticamente | Etiqueta "Detetado" (como já existe no protótipo) |
| **0.60 – 0.84** (média) | Preenche, mas marca para confirmação | Contorno âmbar + etiqueta "Confirmar" |
| **< 0.60** (baixa) | **Não preenche.** Campo fica vazio/como estava | Não aparece marcado; opcionalmente, um texto pequeno "sugestão: X" ao lado do campo, nunca aplicado automaticamente |

Regras adicionais, para além do threshold por campo:

1. **Campos financeiros críticos nunca "saltam" a revisão.** Mesmo com confiança ≥ 0.85, `total`, `movement_type` e `fuel_type` continuam sujeitos ao botão explícito "Submeter despesa" — o threshold decide *se o campo é pré-preenchido*, nunca *se pode ser submetido sem revisão humana*. Isto já é garantido estruturalmente por `/expenses/receipts/analyze` e `/expenses` serem endpoints diferentes (secção 4).
2. `requires_review` (booleano global) é `true` sempre que **qualquer** destas condições se verificar:
   - algum campo caiu na faixa média ou baixa;
   - existe pelo menos uma entrada em `ambiguities`;
   - `page_count > 1` (documento com múltiplas páginas — maior risco de dados trocados);
   - `movement_type` ou `fuel_type` devolvidos pela classificação não correspondem a nenhum valor do enum do sistema (ver secção "Validação dos dados").
3. Quando `requires_review` é `true`, o botão "Submeter despesa" mantém-se ativo (o utilizador pode sempre submeter manualmente), mas a UI reforça visualmente que há itens por confirmar (ex. o aviso já existente no protótipo, mais específico por campo).

---

## 7. Fluxo UX

Mantém os 5 estados do protótipo, com precisão adicional:

**Estado 1 — Upload**
Utilizador arrasta ou seleciona ficheiro. Validação client-side imediata (tipo/tamanho) antes de sequer chamar a API, para feedback instantâneo em erros óbvios.

**Estado 2 — Processamento**
`POST /receipts` → resposta `queued`. UI mostra "A ler o comprovativo…" e começa a consultar `GET /receipts/{id}/analysis` (ou escuta o evento). Deve ter um limite de espera (ex. 20s) — ver "timeout" nos casos de erro.

**Estado 3 — Resultado**
Resposta `done` chega. Frontend aplica os thresholds da secção 6: preenche `auto_filled_fields` normalmente, `review_suggested_fields` com destaque âmbar, deixa `unresolved_fields` vazios.

**Estado 4 — Revisão**
Cada campo tem um de três estados visuais:
- **Reconhecido (confiança alta)** — como a etiqueta "Detetado" atual;
- **Precisa de revisão (confiança média)** — contorno/etiqueta âmbar, ex. "Confirmar valor";
- **Não encontrado** — sem marca especial, campo em branco pronto a preencher à mão.

Importante (acessibilidade): a distinção não pode depender só de cor — usar também o texto da etiqueta ("Detetado" vs "Confirmar"), como já acontece no protótipo.

Qualquer edição manual de um campo remove a etiqueta de confiança desse campo (comportamento já existente no protótipo com `clearDetected`) — o valor passa a ser "do utilizador", não "sugerido pela IA".

**Estado 5 — Confirmação**
Botão "Submeter despesa" continua sempre disponível e é sempre uma ação explícita. Se `requires_review` for `true`, considerar (fase de refinamento de UX, não bloqueante) um pequeno resumo tipo "3 campos foram detetados automaticamente, 1 precisa da sua confirmação" acima do botão.

**Estado adicional — Falha/indisponibilidade**
Se a análise falhar ou expirar, a UI não bloqueia o utilizador: mostra aviso curto ("Não foi possível ler o comprovativo automaticamente — preencha os campos manualmente") e o formulário fica igual ao comportamento sem OCR nenhum. O upload do ficheiro em si (como anexo da despesa) é independente do sucesso da leitura.

---

## 8. Tratamento de erros

| Cenário | Deteção | Comportamento |
|---|---|---|
| Imagem desfocada / ilegível | Confiança global muito baixa em todos os campos | Todos os campos ficam por preencher; aviso "Não conseguimos ler este comprovativo com confiança suficiente" |
| Documento cortado | OCR devolve campos parciais / `page_count` inconsistente | Preenche só o que tiver confiança suficiente; resto fica `unresolved` |
| Fotografia escura | Falha de pré-processamento no OCR | Erro `LOW_QUALITY_IMAGE`; sugestão na UI para tirar nova foto com melhor luz |
| PDF inválido / corrompido | Falha ao abrir o ficheiro no backend | `400` com `INVALID_FILE`; não chega a ir para o OCR |
| PDF protegido por password | Falha ao abrir | `422` com `PROTECTED_FILE`; pedir ao utilizador para remover a proteção ou enviar imagem |
| Ficheiro muito grande | Validado antes do upload completar (limite ex. 10MB) | `413 Payload Too Large`, mensagem clara do limite |
| Formato não suportado | Validação por assinatura real do ficheiro, não só extensão | `415 Unsupported Media Type` |
| Sem valor identificável | `total` ausente/baixa confiança | Campo `total` fica vazio; não bloqueia — utilizador preenche à mão |
| Múltiplos valores | Populaçãode `ambiguities` | Não escolhe sozinho; campo fica sem preenchimento automático, com nota de ambiguidade se a UI quiser mostrar |
| Múltiplas datas | Igual à anterior, aplicado a `date` | idem |
| Múltiplas páginas | `page_count > 1` | Processa todas, mas força `requires_review = true` |
| Sem número de documento | Campo ausente | `document_number` fica vazio, sem bloquear |
| Matrícula não identificada | Confiança baixa/ausente | Campo `vehicle_plate` não preenchido; utilizador introduz manualmente |
| Serviço OCR indisponível | Erro de infraestrutura do fornecedor | `status: "failed"`, `code: OCR_UNAVAILABLE`; UI cai para preenchimento 100% manual |
| Timeout | Sem resposta dentro do limite (ex. 20–30s) | Frontend para de aguardar, mostra aviso, permite continuar manualmente; job pode continuar em background e, se responder depois, é descartado (o utilizador já seguiu em frente) |
| Erro de rede | Falha na chamada HTTP | Mensagem genérica + botão "Tentar novamente" |
| Resposta inválida da IA/OCR | JSON não corresponde ao schema esperado | Backend rejeita a resposta do fornecedor antes de a devolver ao frontend; regista erro interno; frontend recebe `failed`/`OCR_INVALID_RESPONSE` |
| Resultado com baixa confiança geral | Média de confiança abaixo de um limite mínimo (ex. 0.4) | `requires_review = true` forçado; eventualmente, no futuro, poderia nem preencher nenhum campo — a decidir na fase de testes com dados reais |

Princípio geral: **qualquer falha do pipeline de leitura degrada graciosamente para o formulário manual**, nunca bloqueia o registo da despesa.

---

## 9. Segurança e privacidade

### Segurança do ficheiro

- **Validação do tipo real**, não só extensão/MIME declarado pelo browser — verificar assinatura binária (magic bytes) no backend.
- **Limite de tamanho** por ficheiro (ex. 10MB) aplicado antes de gravar em disco/memória.
- **Análise antivírus/malware** do ficheiro antes de qualquer processamento (especialmente PDFs, que podem conter exploits).
- **Renderização/parsing de PDF em ambiente isolado** (sandbox), nunca com bibliotecas desatualizadas expostas diretamente à internet.

### Armazenamento e ciclo de vida

- Storage temporário **encriptado em repouso**, acessível apenas pelo serviço de análise.
- **TTL curto** (ex. eliminação automática 24–72h após processamento, ou imediatamente após submissão/descarte da despesa).
- O ficheiro **definitivo** de uma despesa aprovada é um armazenamento **diferente** deste, com política de retenção própria (ver nota fiscal abaixo) — não confundir "cópia de trabalho para OCR" com "arquivo do comprovativo aprovado".
- Nunca expor URLs diretas e permanentes para os ficheiros — usar **URLs temporários assinados** (ex. validade de poucos minutos) quando o frontend precisar de pré-visualizar a imagem.

### Acesso e autenticação

- Todos os endpoints exigem **autenticação** (o mesmo mecanismo já usado no resto do Portal Símbolo²).
- **Autorização**: um técnico só pode ver/analisar/submeter os seus próprios comprovativos; perfis de contabilidade/aprovação têm acesso de leitura mais amplo, conforme regras já existentes no sistema de despesas.
- Rate limiting no endpoint de upload/análise, para conter abuso e custos.

### Logs e dados sensíveis

- Logs de aplicação **não guardam** o conteúdo do documento, valores financeiros completos, NIF ou matrícula em texto simples — apenas metadados técnicos (`receipt_id`, timestamps, status, tempos de processamento, códigos de erro).
- Nenhum log deve conter o ficheiro em si nem o texto bruto extraído.

### Privacidade / RGPD (aplicável a empresa europeia)

- **Minimização de dados**: enviar ao serviço de OCR/IA apenas o ficheiro necessário — nunca dados adicionais do utilizador (nome, email, histórico) que não sejam indispensáveis à leitura do documento.
- Se o fornecedor de OCR/IA for um serviço de terceiros, é necessário um **acordo de tratamento de dados (DPA)** com esse fornecedor, e confirmar onde os dados são processados/armazenados (idealmente UE, ou com garantias equivalentes — cláusulas contratuais-tipo).
- **Política de retenção** documentada e curta para a cópia "de trabalho" usada no OCR (distinta da retenção fiscal do comprovativo aprovado, que em Portugal está sujeita a prazos legais próprios de conservação de documentos contabilísticos — tipicamente vários anos, gerido fora deste pipeline de OCR).
- **Direito ao apagamento**: se o utilizador descartar a despesa antes de submeter, o ficheiro e os dados extraídos devem poder ser eliminados de imediato (endpoint `DELETE /receipts/{id}`).
- **Auditoria**: manter registo de quando um documento foi processado e por qual motor/versão (já previsto no schema, campo `engine`), sem guardar o conteúdo sensível em si nesse registo de auditoria.

---

## 10. Estratégias de OCR/IA

| Critério | A — OCR tradicional (especializado em documentos/despesas) | B — Modelo multimodal / Vision AI | C — Híbrido (OCR + IA de classificação) |
|---|---|---|---|
| **Precisão em campos estruturados** (data, total, NIF) | Alta — motores especializados em faturas/recibos já têm modelos treinados para estes campos, com confiança calibrada nativamente | Variável — bom em compreensão geral, mas confiança "declarada" por um LLM não é estatisticamente calibrada da mesma forma | Alta — combina o melhor dos dois |
| **Flexibilidade a formatos variados** (recibo manuscrito, fatura estrangeira, talão amachucado) | Média — depende do motor e do treino prévio | Alta — modelos multimodais lidam melhor com layouts pouco comuns | Alta |
| **Classificação semântica** (mapear para os 30 `movement_type` do Símbolo², incluir "Táxi/Uber" vs "Bilhete de autocarro") | Fraca isolado — OCR não entende contexto de negócio | Boa — um LLM consegue interpretar contexto ("bilhete de metro" → "Metro") | Boa — é exatamente para isto que se usa a camada de IA, sobre texto já limpo |
| **Complexidade de integração** | Média | Média | Maior (duas integrações), mas cada uma mais simples e testável isoladamente |
| **Custo por documento** | Geralmente mais baixo, preço por página | Pode ser mais caro (tokens de imagem + geração) | Custo combinado, mas pode otimizar-se (só chama o LLM para classificação sobre texto curto, não a imagem completa) |
| **Velocidade** | Rápida | Mais lenta (latência de geração) | Depende, mas paralelizável nalguns passos |
| **Segurança/controlo de output** | Alta — resposta estruturada, sem "criatividade" | Risco de alucinação de campos se mal restringido | Alta — a IA generativa só atua sobre texto já extraído e é forçada a escolher de uma lista fechada de valores (constrained decoding / function calling), nunca inventa `movement_type` novo |
| **Manutenção** | Simples, mas pouco adaptável a novas categorias sem retreino | Mais flexível a mudanças (basta ajustar o prompt/enum) | Boa — a lista de `movement_type`/`fuel_type` pode crescer sem re-treinar o motor de OCR |

---

## 11. Arquitetura recomendada

**Opção C (híbrida)** é a recomendação: um motor de OCR/Document AI especializado para a extração bruta (data, número de documento, total, moeda, fornecedor, NIF, com confiança nativa por campo), seguido de uma camada de classificação leve — regras determinísticas primeiro (ex. correspondência direta de palavras-chave já testada no protótipo), com um modelo de IA generativa como camada de reforço só para os casos ambíguos que as regras não resolvem (ex. decidir entre "Almoço", "Jantar" ou "Refeição" a partir do contexto, ou mapear "portagem A22" para "Portagens").

Porquê, especificamente para o Símbolo²:

1. **O sistema tem ~30 valores possíveis de `movement_type`**, muitos deles com nomes específicos da empresa (ex. "Substituição de cartão Edenred", "CTT - Despacho") que nenhum motor de OCR genérico vai "saber" classificar sozinho — isto exige uma camada semântica, o que favorece incluir IA generativa nalgum ponto.
2. **Mas os campos financeiros (`total`, `date`, `document_number`) precisam de confiança calibrada e fiável**, algo em que motores de Document AI especializados são tipicamente mais consistentes do que pedir a um LLM para "inventar" um número de confiança.
3. **A classificação semântica não vê a imagem**, só o texto já extraído — o que reduz a quantidade de dados sensíveis (imagem do documento) que circula por múltiplos serviços, alinhado com a secção de privacidade.
4. **`movement_type` e `fuel_type` devem ser restringidos por construção** (lista fechada passada ao modelo, ex. via function calling/structured output), nunca texto livre — isto elimina a maior parte do risco de alucinação da componente generativa.

Esta escolha não fixa ainda um fornecedor específico (conforme pedido) — só a forma da arquitetura.

---

## 12. Roadmap de implementação

| Fase | Objetivo | Critério de saída |
|---|---|---|
| **0. Preparação** | Avaliar 2–3 fornecedores de Document AI/OCR (POC com ~20–30 comprovativos reais anonimizados), confirmar conformidade RGPD/DPA | Fornecedor(es) selecionado(s) com métricas de precisão documentadas |
| **1. Contrato da API** | Fixar o schema JSON (secção 5) e os endpoints (secção 4) como OpenAPI/Swagger | Contrato revisto e aprovado pela equipa de frontend e backend |
| **2. Upload real** | Implementar `POST /receipts` com validação de ficheiro, storage temporário e `DELETE /receipts/{id}` — **sem OCR ainda** | Upload funcional, ficheiros a expirar corretamente, sem qualquer leitura automática |
| **3. Integração OCR/Document AI** | Ligar o storage ao motor de OCR escolhido, devolver campos brutos + confiança nativa | Endpoint de análise devolve `document.*` e `confidence.*` reais para um conjunto de teste |
| **4. Classificação** | Camada de regras + fallback de IA generativa para `movement_type`/`fuel_type`, restrita à lista de enums do sistema | Classificação correta em ≥ X% do conjunto de teste (definir X na Fase 0 com dados reais) |
| **5. Normalização** | Datas ISO, moeda, matrícula (`AA-00-AA`), limpeza de nº de documento | Testes unitários cobrindo os formatos de entrada esperados (`€ 62,30`, `62.30 EUR`, etc.) |
| **6. Validação** | Enums permitidos, ranges plausíveis, deteção de ambiguidade, cálculo de `requires_review` | Backend rejeita/assinala corretamente valores fora do sistema |
| **7. Integração com o formulário** | Substituir a simulação no `prototipo`/frontend real pelos dados reais da API, aplicar thresholds de confiança na UI | Fluxo completo Upload → Processamento → Revisão → Submissão a funcionar de ponta a ponta em ambiente de testes |
| **8. Confiança e revisão humana** | Afinar thresholds com dados reais de uso, medir taxa de correção manual | Thresholds calibrados; taxa de correção manual documentada como baseline |
| **9. Segurança e privacidade** | Auditoria de segurança (antivírus, TTL, logs, DPA assinado), revisão RGPD | Checklist de segurança aprovado antes de dados reais de produção passarem pelo pipeline |
| **10. Testes** | Testes unitários, integração, carga (volume esperado de comprovativos/mês), casos de erro da secção 8 | Todos os cenários de erro listados têm comportamento coberto por teste |
| **11. Produção e monitorização** | Rollout gradual (feature flag / grupo piloto de técnicos), dashboards de observabilidade | Métricas em produção dentro dos critérios de aceitação (secção 13) |

---

## 13. Critérios de aceitação

A funcionalidade está pronta para produção quando, cumulativamente:

- **Nunca** existe um caminho de código em que uma despesa é submetida sem uma ação explícita do utilizador (verificado por revisão de código + teste automatizado dedicado).
- `movement_type` e `fuel_type` devolvidos ao frontend **correspondem sempre** a um valor da lista permitida no sistema, ou são `null` — nunca um valor fora da lista chega à UI.
- Ficheiros de tipo/tamanho não suportado são rejeitados em 100% dos casos testados, antes de qualquer processamento.
- Comprovativos temporários são eliminados dentro da janela de retenção definida, verificado por teste automatizado (não apenas por inspeção manual).
- Todos os cenários de erro da secção 8 têm uma mensagem clara em português para o utilizador e não bloqueiam o preenchimento manual.
- Tempo de processamento (p95) dentro de um limite aceite pela equipa (a definir na Fase 0, com base nos tempos reais do fornecedor escolhido).
- Precisão de extração medida num conjunto de teste representativo atinge o limiar definido na Fase 0.
- Nenhum dado financeiro, pessoal ou conteúdo de documento aparece em logs de aplicação (verificado por revisão/scan de logs).
- DPA assinado (se aplicável) com o(s) fornecedor(es) de OCR/IA, e política de retenção documentada e publicada internamente.
- Revisão de segurança concluída (validação de ficheiros, antivírus, controlo de acesso, URLs temporários).

---

## 14. Riscos

**Técnicos**
- Indisponibilidade ou degradação do serviço de OCR/IA de terceiros — mitigado pelo *fallback* gracioso para preenchimento manual (já desenhado na secção 8).
- Custo por documento a crescer com o volume sem controlo — necessário monitorizar (secção Observabilidade) e definir alertas de orçamento.
- Latência elevada em PDFs longos/multi-página — pode exigir processamento assíncrono mais robusto (filas) desde o início, não só polling simples.
- Deriva do enum: `movement_type` cresce com o tempo (já aconteceu nesta iteração, +25 tipos) e a camada de classificação precisa de acompanhar essa lista sem re-treino manual constante.

**Operacionais**
- Utilizadores a confiarem cegamente nos campos preenchidos automaticamente e a não reverem, mesmo com a UI a pedir revisão — mitigar com fricção visual proporcional ao risco (secção 6/7) e, se necessário no futuro, formação interna.
- Dependência de um único fornecedor (vendor lock-in) — mitigar mantendo a camada de classificação/normalização independente do motor de OCR escolhido, para facilitar troca de fornecedor.

**Qualidade de dados**
- Fotografias de má qualidade (desfocadas, mal enquadradas) a gerar sistematicamente baixa confiança — pode ser necessário orientação na própria UI ("tire a foto com boa luz, documento completo no enquadramento").
- Documentos com múltiplos valores/datas a exigir sempre intervenção manual — aceitável dado o requisito de nunca inventar dados, mas pode gerar frustração se for muito frequente; medir a taxa real na Fase 0.

**Conformidade**
- Transferência de dados para fora da UE, se o fornecedor escolhido não garantir localização/tratamento adequado — a decidir explicitamente na Fase 0, não deixar implícito.
- Sobreposição entre a retenção fiscal obrigatória do comprovativo aprovado (regras contabilísticas portuguesas) e a política de retenção curta da cópia de trabalho do OCR — são coisas diferentes e é preciso documentá-las separadamente para não gerar confusão de compliance.

---

## 15. Próximo passo

Antes de qualquer código: **Fase 0 — avaliação de fornecedores.**

Concretamente, a primeira tarefa técnica deveria ser:

1. Reunir um pequeno conjunto de comprovativos reais (anonimizados/ou fictícios mas realistas) representativos dos tipos mais comuns do Símbolo² — combustível, portagens, refeição, material, alojamento.
2. Testar 2–3 fornecedores/motores de Document AI (Opção A/C da secção 10) contra esse conjunto, medindo precisão por campo (data, total, NIF, etc.) e a qualidade da confiança que cada um devolve nativamente.
3. Com esses resultados, **fechar o contrato definitivo da API** (secção 4/5, formalizado em OpenAPI) e escolher formalmente a arquitetura de classificação (regras vs. híbrido com IA).

Só depois disso faz sentido avançar para a Fase 2 (upload real) do roadmap — implementar contra um contrato ainda não validado por dados reais é o principal risco a evitar aqui.
