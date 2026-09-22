# Guia de labeling — Ground truth do PoC de OCR

Regras exatas para preencher o ground truth de cada documento (`poc-000X.json`, seguindo [`ground-truth.schema.json`](ground-truth.schema.json)).

## Regra mais importante — ler antes de tudo o resto

> O ground truth representa **o que está escrito no documento**. Não representa o que o sistema, o labeler, ou o bom senso "acham" que deveria estar lá.

Não inferir nenhum valor a partir de:
- histórico de despesas do utilizador/técnico;
- viatura associada ao técnico;
- fornecedor "habitual" ou conhecido;
- despesas anteriores semelhantes;
- dados da empresa ou de outro sistema.

Se não está legível no documento, o campo correspondente é `absent` ou `unreadable` (ver abaixo) — nunca um valor "adivinhado", mesmo que pareça óbvio.

---

## Distinguir três situações, não usar `null` para tudo

Cada campo de conteúdo (`date`, `document_number`, `total`, `currency`, `supplier`, `supplier_tax_id`, `vehicle_plate`, `fuel_type`) segue a estrutura:

```json
{ "status": "present", "value": "2026-09-15" }
```

com `status` sempre um de três valores:

| `status` | Significa | `value` |
|---|---|---|
| `present` | O dado existe no documento e foi lido com confiança pelo labeler | o valor normalizado (ver regras abaixo) |
| `absent` | O documento **não contém** este dado (ex. um talão de portagem não tem "fornecedor" impresso de forma identificável) | `null` |
| `unreadable` | O dado provavelmente existe, mas está ilegível nesta cópia (desfocado, cortado, apagado) | `null` |

Esta distinção importa porque muda a interpretação do resultado de um fornecedor de OCR:
- Um fornecedor que devolve vazio num campo `absent` está **correto**.
- Um fornecedor que devolve vazio num campo `unreadable` é um resultado **neutro/esperado** (nem o humano conseguiu ler).
- Um fornecedor que devolve vazio num campo `present` é uma **falha real** (Missing Field Rate).
- Um fornecedor que devolve um valor num campo `absent` é uma **invenção** (False Extraction Rate), mesmo que o valor "pareça" plausível.

Em caso de dúvida entre `absent` e `unreadable`: se há qualquer traço visual do dado (ex. consegue perceber-se que havia números ali, mas não quais) é `unreadable`; se não há nenhum indício de que o campo alguma vez existiu nesse documento, é `absent`.

---

## Regras campo a campo

### Data (`date`)

Documento mostra `15/09/2026`, `15-09-2026`, `15 set 2026`, etc. → guardar sempre `2026-09-15` (ISO 8601, `YYYY-MM-DD`).

Se houver mais do que uma data no documento (ex. data de emissão e data de vencimento), usar a **data de emissão/transação** — é essa que corresponde ao campo `date` da despesa. Se não for claro qual é qual, marcar `unreadable` e explicar em `notes`.

### Número de documento (`document_number`)

Preservar o identificador **exatamente como impresso**, incluindo prefixos (`FT 4471`, `FR 2026/118`, `REC-00231`). Não normalizar espaços/traços de forma diferente do original — a única limpeza aceite é remover espaços em excesso no início/fim.

### Valor total (`total`)

`62,30 €`, `€ 62,30`, `TOTAL 62,30`, `62.30 EUR` → guardar sempre como número, formato `62.30` (ponto decimal, sem símbolo de moeda, sem separador de milhares).

Usar sempre o **valor total final a pagar** (já com impostos/gorjeta incluídos, se aplicável), não um subtotal. Se houver vários totais (ex. subtotal + total, ou total antes/depois de desconto), registar o total final e, se ambíguo, usar `notes` para explicar qual foi escolhido e porquê.

### Moeda (`currency`)

Código ISO 4217 de 3 letras: `EUR`, `USD`, `GBP`, etc. Se o documento não indicar moeda explicitamente mas o contexto deixa claro (ex. talão português comum, símbolo `€`), assumir `EUR` com `status: "present"` — isto não é "inferir um valor ausente", é ler o símbolo `€` como a própria informação de moeda.

### NIF do fornecedor (`supplier_tax_id`)

Guardar **apenas os dígitos**, depois de confirmar visualmente que correspondem mesmo a um NIF (rótulo "NIF", "Contribuinte", "NIPC" próximo dos números). **Não corrigir, não validar por checksum, não completar dígitos em falta** — o ground truth regista o que está impresso, mesmo que pareça ter um dígito errado. Validação de checksum é responsabilidade da camada de Validação do backend (fora do ground truth).

### Fornecedor (`supplier`)

Regra de normalização: usar o nome comercial tal como aparece no cabeçalho/destaque do documento, sem incluir a forma jurídica só se ela não estiver destacada (ex. documento mostra em grande "Posto ABC" e em letra pequena "ABC Combustíveis, Lda." no rodapé fiscal → registar `"Posto ABC"`, o nome pelo qual o documento se apresenta). Se só existir a designação fiscal completa, usar essa. Não abreviar nem expandir o nome por conta própria.

### Matrícula (`vehicle_plate`)

**Só preencher quando a matrícula estiver realmente impressa/visível no documento.** Isto é raro — a maioria dos recibos de combustível ou portagem não imprime a matrícula. Não inferir a matrícula a partir do técnico responsável pelo upload, da viatura habitualmente associada a ele, ou de qualquer outro sistema. Quando não estiver no documento: `status: "absent"`.

### Tipo de combustível (`fuel_type`)

Preencher só quando o **texto do talão** indicar explicitamente o tipo (ex. "Gasóleo simples", "Gasolina 95", "AdBlue" impresso na linha do produto). Se o talão só disser "Combustível" genericamente sem especificar, é `unreadable` só se houver indício de que a informação existe mas está cortada/ilegível — caso contrário é `absent` (a informação simplesmente não está lá).

### `expected_movement_type` / `expected_fuel_type`

Estes dois campos **não seguem** a estrutura `present/absent/unreadable` — são a categorização humana do documento (a que tipo de despesa este documento corresponde, usando a lista de `movement_type`/`fuel_type` já existente no sistema), usada apenas na fase de avaliação de classificação, não na avaliação de OCR. Preencher sempre com o valor mais adequado da lista existente no Símbolo² (ex. `"Combustível"`, `"Portagens"`, `"Reparação Auto"`).

### `notes`

Texto livre, opcional. Usar para justificar qualquer decisão ambígua tomada acima (ex. "dois totais no documento, usado o valor final após desconto"; "NIF com um dígito pouco nítido, mantido como impresso"), ou para registar impacto de anonimização no documento (ver `README.md`, secção "Anonimização e privacidade").

---

## Processo de dupla revisão

1. **Pessoa A** faz o labeling completo do documento (todos os campos).
2. **Pessoa B**, sem ver as respostas da Pessoa A primeiro, revê independentemente só os campos **críticos**: `total`, `date`, `document_number` (ver tabela de criticidade em `README.md`).
3. Comparar as duas leituras desses três campos:
   - Coincidem → confirmar `ground_truth_complete = true` no manifest.
   - Divergem → as duas pessoas olham novamente para o documento juntas e resolvem por consenso visual (nunca por votação ou por "parece mais razoável") — o valor final tem de corresponder ao que está de facto escrito no documento.
4. Os restantes campos (importantes/auxiliares) não exigem segunda revisão sistemática — só se a Pessoa A assinalar dúvida em `notes`.

Isto mantém o processo leve (só 3 campos revistos a dobrar, não o documento inteiro) sem abdicar de qualidade nos campos que mais pesam nas métricas finais.
