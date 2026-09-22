# Kit de PoC — OCR / Document AI para comprovativos do Símbolo²

Kit operacional para comparar fornecedores de OCR/Document AI usando exatamente o mesmo conjunto de documentos e o mesmo critério de avaliação.

**Este kit não contém nenhum comprovativo real.** Serve para organizar o processo — a recolha e o preenchimento do ground truth são trabalho humano, feito fora deste repositório (ver "Próximo passo operacional" no fim).

Não implementa OCR, não chama nenhum fornecedor, não envia nada para a internet. É documentação e estrutura de suporte, alinhada com [`docs/arquitetura-ocr-ia.md`](../arquitetura-ocr-ia.md) (arquitetura) e a análise de Fase 0 já feita (fornecedores candidatos, métricas, thresholds).

---

## Objetivo do PoC

Comparar objetivamente diferentes motores de OCR/Document AI (Fase 0: Google Document AI, Azure AI Document Intelligence, Mindee, e um modelo multimodal genérico como termo de comparação) usando o mesmo dataset e o mesmo ground truth — para decidir qual fornecedor (ou combinação) usar na integração real do módulo de despesas.

Mede **extração**, não classificação — `expected_movement_type` e `expected_fuel_type` ficam registados no ground truth para uma fase posterior, mas não fazem parte da nota de qualidade do OCR em si (ver [`docs/arquitetura-ocr-ia.md`](../arquitetura-ocr-ia.md), secção "Não confundir OCR com classificação").

## Fluxo

```
Documento (comprovativo real, recolhido manualmente)
  → anonimização
  → ground truth humano (labeling + dupla revisão)
  → [neste kit termina o trabalho manual — a partir daqui é execução do PoC]
  → envio ao fornecedor (fora do escopo desta fase)
  → resposta do OCR
  → normalização
  → comparação com o ground truth
  → métricas
  → decisão
```

Este kit prepara tudo até "ground truth humano" inclusive, mais a estrutura onde os resultados dos fornecedores vão ser guardados depois.

---

## Estrutura deste kit

```
docs/ocr-poc/
├── README.md                  este ficheiro
├── labeling-guide.md          regras exatas de preenchimento do ground truth
├── ground-truth.schema.json   schema formal (JSON Schema) de cada documento do ground truth
└── dataset-manifest.csv       template do índice do dataset (uma linha por documento)
```

Os **documentos** (imagens/PDFs dos comprovativos) e os **ficheiros de ground truth** (um `.json` por documento, seguindo `ground-truth.schema.json`) não ficam neste repositório de documentação — a localização de armazenamento deve ser decidida à parte, com controlo de acesso (ver secção "Anonimização e privacidade").

---

## Estrutura do dataset

**Quantidade**: ~50 documentos (dentro do intervalo de 40–60 já validado na Fase 0).

**Categorias e distribuição proposta**

| Categoria (`movement_type`) | Documentos | Racional |
|---|---:|---|
| Combustível | 7 | Categoria mais frequente; talão térmico é o caso mais difícil de OCR do dataset |
| Portagens | 6 | Muito frequente; talões pequenos, por vezes com matrícula impressa |
| Hotel / Alojamento | 4 | Faturas mais estruturadas, por vezes multipágina |
| Almoço | 4 | Frequente, formato de talão variável |
| Jantar | 3 | Igual a Almoço, categoria separada no sistema |
| Parque / Parquímetro | 3 | Talões muito pequenos, texto reduzido |
| Reparação Auto | 4 | Faturas de oficina, mais linhas de detalhe, bom teste de reconhecimento de tabela |
| Pneus | 3 | Similar a Reparação Auto, por vezes a mesma oficina |
| Inspeção viatura | 3 | Documento oficial, formato mais previsível |
| Táxi / Uber | 3 | Inclui recibos digitais (PDF/print de app) — bom teste de formato "não físico" |
| Mat. Escritório | 3 | Faturas de retalho, formato variável |
| Ferramentas | 3 | Idem |
| CTT - Despacho | 2 | Documento oficial, formato consistente |
| CTT - Selos | 2 | Idem, tipicamente valores pequenos |
| **Total** | **50** | |

Categorias com poucos exemplos (2–3) não permitem grande variação de dificuldade dentro delas — nesses casos, priorizar pelo menos 1 documento fácil e 1 difícil, preenchendo o resto em dificuldade média (ver critério abaixo).

**Distribuição por dificuldade** (alvo aproximado, não rígido): 15 fácil / 20 médio / 15 difícil.

| Nível | Definição objetiva |
|---|---|
| **Fácil** | Documento perfeitamente legível, bom enquadramento (documento inteiro visível, sem corte), boa iluminação uniforme, sem inclinação relevante, papel em bom estado |
| **Médio** | Pelo menos uma destas condições: ligeira inclinação (até ~15°), sombra parcial, texto pequeno mas legível, leve desfoque, papel ligeiramente amarrotado sem perda de informação |
| **Difícil** | Pelo menos uma destas condições: papel térmico visivelmente desbotado, documento parcialmente cortado/fora de enquadramento, reflexo de luz sobre o texto, amarrotado com perda de legibilidade em zonas do documento, iluminação claramente insuficiente, documento muito longo (talão extenso) |

Um documento pode acumular múltiplas `difficulty_tags` (ex. `["fotografia_telemovel", "pouca_luz", "papel_termico"]`) — o nível (`difficulty`) é a classificação agregada; as tags registam as condições específicas, para permitir cruzar resultados por causa de dificuldade, não só por nível.

**Formatos** — proporção indicativa dentro dos ~50:
- JPG (fotografia de telemóvel): ~70% — é o caso de uso real dominante do Símbolo²
- PNG (ex. print de recibo digital, app de táxi): ~10%
- PDF (fatura eletrónica — hotel, oficina, CTT): ~20%, dos quais 3–5 devem ser **multipágina**, se existirem casos reais assim

**Idioma** (`language`, ver manifest): maioria `pt`; incluir pelo menos 3–5 documentos `en` ou `es` (ex. aluguer de viatura ou hotel no estrangeiro), alinhado com o requisito de suporte multi-idioma já definido na arquitetura.

---

## Convenção de IDs e nomes de ficheiro

- Identificador único: `poc-0001`, `poc-0002`, … (sequencial, 4 dígitos, sem reaproveitar números se um documento for removido).
- **Nunca** usar no nome do ficheiro (nem em metadados do ficheiro):
  - nome do colaborador/técnico;
  - matrícula;
  - nome do fornecedor;
  - número do documento;
  - data específica do documento (evitar mesmo `2026-09-15.jpg` — usar só `poc-0001.jpg`).
- O ficheiro do documento e o seu ground truth partilham o `document_id`: `poc-0001.jpg` ↔ `poc-0001.json`.
- Ao exportar/guardar o ficheiro, confirmar que os **metadados EXIF** (nome de ficheiro original do telemóvel, geolocalização, etc.) foram removidos — isto é parte do procedimento de anonimização abaixo, não é automático.

---

## Processo humano

Resumo (regras completas em [`labeling-guide.md`](labeling-guide.md)):

1. **Pessoa A — Recolha e anonimização**: reúne o documento, atribui `document_id`, remove/anonimiza dados desnecessários, regista no manifest.
2. **Pessoa A — Labeling inicial**: preenche o ground truth (`poc-000X.json`) seguindo `labeling-guide.md`.
3. **Pessoa B — Revisão dos campos críticos**: revê independentemente `total`, `date`, `document_number` (ver critérios de criticidade abaixo) contra o documento original. Não revê tudo — só os campos de maior impacto, para manter o processo leve.
4. **Divergência entre A e B**: resolução manual conjunta, olhando outra vez para o documento; o valor final é o que os dois confirmam olhando para o documento — nunca "o que parece mais plausível" sem confirmar visualmente.
5. Só depois disto o documento é marcado `ground_truth_complete = true` no manifest e está pronto para entrar no PoC.

Critérios de criticidade dos campos (usados para decidir o que a Pessoa B revê sempre, e mais tarde para ponderar as métricas):

| Nível | Campos | Porquê |
|---|---|---|
| **Críticos** | `total`, `date`, `document_number` | Maior impacto financeiro/de auditoria; um erro aqui é o mais caro de corrigir depois na despesa real |
| **Importantes** | `supplier`, `supplier_tax_id`, `currency` | Relevantes para contabilidade e auditoria, mas um erro é menos crítico que nos campos acima. Nota: `currency` sobe para crítico nos documentos em moeda estrangeira (ex. hotel fora de Portugal) |
| **Auxiliares** | `vehicle_plate`, `fuel_type` | Úteis mas raramente vêm impressos no documento — a expectativa realista é que continuem maioritariamente `absent` no ground truth |

`expected_movement_type` e `expected_fuel_type` ficam **fora** desta tabela de criticidade — são rótulos para uma fase de avaliação de classificação futura, não campos de extração de OCR, e não devem ser usados para pontuar a qualidade de extração de um fornecedor (ver `docs/arquitetura-ocr-ia.md` e a distinção OCR vs. classificação já estabelecida).

---

## Anonimização e privacidade

**Dados potencialmente sensíveis a verificar em cada documento antes de entrar no dataset:**
- Nome do colaborador/técnico (se impresso, ex. em fatura de hotel em nome da pessoa)
- Matrícula da viatura
- Morada pessoal
- NIF pessoal (distinto do NIF do fornecedor, que é necessário ao teste)
- Telefone / email pessoal
- Dados de pagamento (nº de cartão, mesmo parcial/mascarado)
- Qualquer outro identificador pessoal direto ou indireto

**Regra central**: não remover dados que sejam necessários para avaliar o OCR. O NIF do **fornecedor** (não pessoal), o valor total, a data, o nome do estabelecimento — isto é precisamente o que o PoC precisa de testar, não deve ser removido.

Quando um dado sensível **não** é necessário ao teste (ex. nome do colaborador impresso numa fatura de hotel, morada pessoal):
1. Preferir **substituição por dado sintético visualmente equivalente** (mesmo comprimento/formato aproximado) a simplesmente apagar/rasurar — apagar cria uma "mancha" artificial que pode alterar o comportamento do OCR nesse ponto do documento (um bloco preto ou branco onde antes havia texto pode confundir o motor de deteção de layout de forma que um documento real nunca teria).
2. Se a substituição visual não for possível sem comprometer a legibilidade do resto do documento, documentar essa limitação no campo `notes` do ground truth (ex. `"notes": "Nome do hóspede rasurado; pode ter reduzido ligeiramente a área de texto legível."`) — para que uma eventual queda de precisão nesse documento específico possa ser explicada e não confundida com uma falha real do fornecedor.
3. **Nunca** anonimizar o NIF do fornecedor, o número de documento, a data ou o total — são o próprio objeto do teste.

**Controlo de acesso**: o conjunto de documentos e ground truth deve ficar num local com acesso restrito à equipa do PoC — não neste repositório de documentação pública do projeto.

**Retenção**: manter o dataset apenas durante a execução do PoC e a análise dos resultados; depois, guardar só o relatório agregado de métricas, eliminando os documentos e os ground truths individuais (alinhado com `docs/arquitetura-ocr-ia.md`, secção de privacidade).

---

## Estratégia de amostragem (evitar viés)

- Não usar só documentos "perfeitos" — a distribuição de dificuldade acima é obrigatória, não opcional.
- Não usar só recibos de combustível — respeitar a distribuição por categoria mesmo que combustível seja o mais fácil de reunir.
- Não usar só fornecedores/estabelecimentos conhecidos ou "bonitos" (ex. só postos de grande cadeia) — incluir também comerciantes pequenos/locais, que tendem a ter talões piores.
- Preservar uma proporção real de fotografias de telemóvel "tal como o técnico tira no dia a dia" — não reencaminhar fotos já corrigidas/editadas.

**Se dois fornecedores ficarem muito próximos nos resultados** (diferença de Field Accuracy ponderada abaixo de uma margem que não seja claramente decisiva, ex. <3–5 pontos percentuais): ampliar o dataset de forma direcionada — acrescentar mais documentos especificamente das categorias/dificuldades onde os dois candidatos divergiram, em vez de simplesmente duplicar o dataset inteiro. Isto foca o esforço adicional onde a informação é mais útil.

---

## Convenção para resultados futuros (ainda não criado)

Quando o PoC for executado (fora do escopo desta tarefa), os resultados de cada fornecedor devem seguir esta estrutura, para ligação directa ao `document_id`:

```
results/
├── google/
│   └── poc-0001.json     resposta bruta + campos mapeados para o ExtractionResult comum
├── azure/
│   └── poc-0001.json
├── mindee/
│   └── poc-0001.json
└── multimodal/
    └── poc-0001.json
```

Nenhuma destas pastas é criada nesta fase — fica só definida a convenção, para quando a integração de teste (fora do escopo desta tarefa) for feita.

---

## Cobertura de métricas futuras

O manifest e o schema de ground truth já preparados suportam todas as métricas definidas na Fase 0:

| Métrica | Suportada por |
|---|---|
| Exact Match / Normalized Match | Comparação entre `results/<fornecedor>/poc-000X.json` e `poc-000X` no ground truth, campo a campo |
| Field Accuracy (por categoria/dificuldade) | `category` e `difficulty` no manifest, cruzados com os resultados |
| Missing Field Rate | Campos com `status: "present"` no ground truth vs. campo devolvido vazio pelo fornecedor |
| False Extraction Rate | Campos com `status: "absent"`/`"unreadable"` no ground truth vs. fornecedor a devolver um valor mesmo assim (ou valor errado com confiança alta) |
| Confidence Calibration | Confiança declarada pelo fornecedor (em `results/`) vs. Normalized Match real, agrupado por faixa |
| Latência / falhas | A registar por execução em `results/`, não faz parte do ground truth (é uma métrica de fornecedor, não do documento) |
| Custo por documento | Idem — métrica de execução, não do dataset |

Nenhum campo adicional foi necessário no manifest ou no ground truth para suportar estas métricas além dos já propostos nas secções seguintes.

---

## Próximo passo operacional (fora do código — para o utilizador)

Este kit não avança sozinho — precisa de trabalho humano fora deste repositório:

1. **Reunir os ~50 comprovativos reais** (ou digitalizações/fotos deles), seguindo a distribuição de categorias e dificuldade acima. Pode começar por juntar o que já existe em arquivo de despesas anteriores.
2. **Anonimizar** cada um, segundo a secção "Anonimização e privacidade".
3. Guardar cada documento como `poc-000X.<ext>` num local com acesso controlado (a decidir — não neste repositório).
4. Preencher uma linha por documento em `dataset-manifest.csv`.
5. Preencher o ground truth de cada documento (`poc-000X.json`), seguindo `labeling-guide.md` e validando contra `ground-truth.schema.json`.
6. Aplicar a dupla revisão nos campos críticos.
7. Só quando um documento cumprir a checklist abaixo, marcar `ground_truth_complete = true`.

**Checklist de validação antes de um documento entrar no dataset:**

- [ ] Tem `document_id` atribuído e único
- [ ] Foi anonimizado (ou confirmado que não contém dados sensíveis desnecessários)
- [ ] Tem `category` atribuída (uma das categorias da tabela acima)
- [ ] Tem `difficulty` e pelo menos uma `difficulty_tag` coerente com o conteúdo
- [ ] Tem ground truth preenchido, validado contra `ground-truth.schema.json`
- [ ] Foi revisto por uma segunda pessoa nos campos críticos (`total`, `date`, `document_number`)
- [ ] Não contém dados pessoais desnecessários ao teste (nome de colaborador, morada pessoal, NIF pessoal, etc.)
- [ ] Está num formato suportado (JPG, PNG, PDF) e abre corretamente
- [ ] Está registado no `dataset-manifest.csv` com `status` atualizado

Só depois de ter um número razoável de documentos "prontos" (não precisa de ser os 50 de uma vez) é que faz sentido avançar para a fase seguinte — contactar os fornecedores candidatos e executar o PoC propriamente dito, que continua **fora do escopo desta tarefa**.
