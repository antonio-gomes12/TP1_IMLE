# TP1 — From Raw Detections to Real Intelligence

**LIACD · Licenciatura em IACD · 2025/2026**  
António Gomes · Nº 53845

Pipeline de análise de dados de retalho que transforma eventos brutos de visão computacional em inteligência accionável para o gestor de loja.

---

## Descrição

O sistema de visão computacional de uma loja monitoriza **23 zonas** e produz eventos `entry`, `exit` e `linger` para cada pessoa detectada, sem identificador de pessoa. O pipeline reconstrói trajectórias individuais, calcula métricas de comportamento e gera um briefing semanal automático em linguagem natural.
```
events.csv → stitcher.py → journeys.csv
↓
analytics.py → metrics.json
↓
insights.py → insights.json
↓
report.py → weekly_report.md
```
---

## Estrutura do Projecto

```
TP1-IMLE/
├── data/
│   ├── events.csv                  # Dataset original (250.015 eventos, 7 dias)
│   ├── events_validation.csv       # Dataset com anomalias injectadas
│   └── anomalies_manifest.json     # Registo das anomalias injectadas
├── output/
│   ├── journeys.csv                # Trajectórias reconstruídas
│   ├── metrics.json                # Métricas pré-calculadas
│   ├── insights.json               # Insights gerados pelo LLM
│   ├── insights_a.json             # Resultado estratégia A (zero-shot)
│   ├── insights_b.json             # Resultado estratégia B (few-shot)
│   └── weekly_report.md            # Briefing semanal para o gestor
├── prompts/
│   ├── strategy_a_zero_shot.txt    # Prompt estratégia A
│   └── strategy_b_few_shot.txt     # Prompt estratégia B
├── src/
│   ├── stitcher.py                 # Fase 1: Reconstrução de trajectórias
│   ├── analytics.py                # Fase 2: Pipeline analítico
│   ├── insights.py                 # Fase 3: LLM Insight Engine
│   └── report.py                   # Fase 4: Report semanal
├── evaluate.py                     # Harness de avaliação end-to-end
├── inject_anomalies.py             # Injecção de anomalias para teste
├── zones.json                      # Grafo de adjacência das 23 zonas
└── requirements.txt
```

---

## Pré-requisitos

### Python
- Python 3.13

### Python packages

pip install -r requirements.txt

**`requirements.txt`:**

pandas
numpy==2.2.5
requests==2.32.3

### Ollama (LLM local)

O pipeline usa o modelo **`mistral:7b`** via Ollama, executado localmente — sem API key, sem internet.

1. Instalar o Ollama: [https://ollama.com/download](https://ollama.com/download)
2. Instalar o modelo (só uma vez, ~4 GB):
ollama pull mistral:7b
3. A app Ollama tem de estar aberta quando o `insights.py` corre.

---

## Execução do Pipeline

Corre os módulos pela seguinte ordem:

Fase 1 — Reconstrução de trajectórias
python src/stitcher.py --input data/events.csv --output output/journeys.csv
Fase 2 — Pipeline analítico
python src/analytics.py --input output/journeys.csv --output output/metrics.json
Fase 3 — LLM Insight Engine (requer Ollama a correr)
python src/insights.py --input output/metrics.json --output output/insights.json
Fase 4 — Report semanal
python src/report.py --input output/insights.json --output output/weekly_report.md

---

## Avaliação

### Criar dataset de validação próprio (com anomalias conhecidas)

python inject_anomalies.py

Gera `data/events_validation.csv` com 5 anomalias injectadas no último dia:

| ID | Tipo | Zona / Hora | Descrição |
|----|------|-------------|-----------|
| A1 | Pico de tráfego | Z_S3, 14h | Eventos ×3 — promoção não planeada |
| A2 | Zona fantasma | Z_N4, 16h | 0 visitantes — corredor bloqueado |
| A3 | Dwell spike | Z_S1 | Dwell time ×5 — fila anormal |
| A4 | Pico tardio | Z_C2, 21h | Actividade anormal à hora de fecho |
| A5 | Entries em Z_CK | Z_CK | Entradas em zona de saída (impossível) |

### Correr o harness de avaliação

python evaluate.py --data data/events_validation.csv --output evaluation_report.json

O harness corre o pipeline completo e calcula 6 métricas automaticamente:

| Métrica | Descrição | Threshold |
|---------|-----------|-----------|
| Consistência | % trajectórias sem sobreposição temporal | 100% |
| Cobertura | % eventos atribuídos a alguma trajectória | 90% |
| Completude | % trajectórias Z_E → Z_E/Z_CK | 50% |
| Detecção de anomalias | % anomalias identificadas nos insights | 50% |
| Precisão numérica | % números nos insights verificáveis nos dados | 70% |
| Ausência de alucinação | % valores do report verificáveis no metrics.json | 70% |

**Resultados obtidos no dataset de treino:**

consistencia          : 100.00%  ✓
cobertura             :  51.56%  ✗
completude            :  99.71%  ✓
detecao_anomalias     :   6.45%  ✗
precisao_numerica     :  85.00%  ✓
ausencia_alucinacao   :  94.44%  ✓
Score global          :  72.86%

---

## Decisão Arquitectural

> **Toda a computação sobre os dados é feita em Python. A LLM é responsável exclusivamente pela síntese e linguagem natural.**

O `metrics.json` serve como registo imutável das métricas calculadas, permitindo verificar a posteriori se o report contém afirmações factuais correctas. Esta separação garante fiabilidade numérica, reprodutibilidade (temperature=0, seed=42) e auditabilidade.

---

## Modelo LLM

| Parâmetro | Valor |
|-----------|-------|
| Modelo | `mistral:7b` |
| Interface | Ollama (local) |
| Temperature | 0 (reprodutibilidade) |
| Seed | 42 |
| Contexto | 8.192 tokens |

### Estratégias de Prompting

**Estratégia A — Zero-shot:** instrução directa com schema e dados.  
**Estratégia B — Few-shot:** igual ao A com exemplos de bons e maus insights antes da geração.

| Métrica | A | B |
|---------|---|---|
| Especificidade | 80% | 100% |
| Menção de zona | 0% | 40% |
| Completude | 100% | 100% |
| **Score** | **0,620** | **0,820** |
| Tempo | 29,9s | 31,1s |

A estratégia B vence consistentemente em todos os runs (+0,200 no score).

---

## Limitações Conhecidas

- **Cobertura (51,7%):** visits de zonas internas sem trajectória activa ou adormecida disponível são descartados. Reflecte trajectórias parciais do início/fim do dia e falhas de sensor.
- **Detecção de anomalias (6,45%):** o mistral:7b condensa múltiplas anomalias num único insight, ignorando as de menor magnitude. Limitação do modelo de 7B parâmetros.
- **Português do Brasil:** o modelo gera ocasionalmente português do Brasil apesar da instrução explícita no prompt.
