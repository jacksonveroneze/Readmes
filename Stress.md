# Playbook de Stress Test e Sizing (API)

## Objetivo
Encontrar a **capacidade segura por pod** e calcular o **número de pods** necessário para atender o **RPS de pico** com **margem**, validando com teste de aceitação.

---

## 1) Defina metas (critérios)
### SLO (pass/fail no k6)
- Erro (`http_req_failed`) **≤ 1%**
- Latência (`http_req_duration`) **p95 ≤ 300ms**
- `dropped_iterations == 0`

### Guardrail (para sizing)
- CPU por pod **≤ 80% sustentado**
- Sem fila crescente (requests in progress / threadpool queue não podem crescer continuamente)

---

## 2) Defina cenário
- Endpoint(s) do pico (mix ou único endpoint)
- IDs variáveis (evitar cache irreal)
- Definir `RPS_pico` e **margem** `m` (ex.: 30%)

---

## 3) Congele ambiente + observabilidade
- Ambiente igual produção (Traefik/DB/cache/limits CPU+RAM)
- Tudo healthy
- Dashboards mínimos:
  - CPU/mem por pod
  - p95/p99 e erro
  - dropped_iterations
  - fila (requests in progress, threadpool queue)
  - conexões/latência DB e (se aplicável) PgBouncer

---

## 4) Baseline (sanidade)
- Rodar **50–100 RPS** por **1–3 min**
- Deve passar SLO/guardrail e métricas coerentes

---

## 5) Capacidade unitária (1 pod)
- Escale para **1 pod**
- Rode **step load** (degraus de ~60s) até falhar
- **RPS_pod_safe** = maior degrau que passa **SLO + guardrail**
- Se ficou “entre degraus”, repetir com step menor (ex.: 25–50 RPS)

---

## 6) Cálculo de pods
- `RPS_alvo = RPS_pico * (1 + m)`
- `pods = ceil(RPS_alvo / RPS_pod_safe)`
- Opcional HA: `pods_ha = pods + 1`

---

## 7) Validação (teste de aceitação com N pods)
- Subir para `pods`
- Rodar **RPS_alvo** por **10–20 min** (ramp suave)
- Passa se:
  - SLO ok
  - guardrail ok
  - DB/PgBouncer/Traefik estáveis (sem fila crescente)

---

## 8) Conclusão (o que registrar)
- `RPS_pod_safe`
- `pods` no pico e fora do pico
- CPU/mem por pod recomendado (requests/limits)
- Ajustes necessários (ex.: pool/PgBouncer/ramp/pre-warm) caso tenha ocorrido timeout/fila
- Alertas principais em produção: p95, erro, drops, CPU>80%, fila/threadpool, PgBouncer waiting/CPU, conexões no DB
