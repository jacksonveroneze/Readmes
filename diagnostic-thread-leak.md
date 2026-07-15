# Diagnóstico de Thread Count Elevado — Guia de Comandos

## 0. Pré-requisitos

```bash
# Instala as ferramentas se ainda não tiver
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-stack
dotnet tool install -g dotnet-counters

# Confirma instalação
dotnet tool list -g

# Descobre o PID da API
dotnet-counters ps
```

---

## 1. Confirma o sintoma — `dotnet-counters`

```bash
dotnet-counters monitor --process-id <pid>
```

**Observar:**
```
dotnet.thread_pool.thread.count       → TP workers
dotnet.process.memory.working_set     → memória total do processo
```

> Se `working_set` crescendo mas `thread_pool.thread.count` estável → threads fora do pool (`new Thread()`).

---

## 2. Inspeciona call stacks sem dump — `dotnet-stack`

```bash
dotnet-stack report --process-id <pid>
```

**Sinal de thread leak — mesmo call stack repetido N vezes:**

```
Thread (id: 2710002)
  ThreadLeakService.RecurseAndSleep
  System.Threading.Thread.Sleep

Thread (id: 2710003)
  ThreadLeakService.RecurseAndSleep   ← idêntico ao anterior
  System.Threading.Thread.Sleep
```

> Se o mesmo call stack aparecer em dezenas de threads → causa identificada, pode pular para a correção.  
> Se precisar de mais detalhes → continua para o dump.

---

## 3. Captura o dump — `dotnet-dump collect`

```bash
# Dump completo
dotnet-dump collect --process-id <pid>

# Dump em diretório específico
dotnet-dump collect --process-id <pid> --output ~/diagnostics/
```

**Saída esperada:**
```
Writing full to /home/user/diagnostics/core_20260715_000351
Complete
```

> ⚠️ O dump é uma cópia da memória do processo — pode gerar arquivo de centenas de MB a GBs dependendo do heap. Garanta espaço em disco antes.

---

## 4. Analisa o dump — `dotnet-dump analyze`

```bash
dotnet-dump analyze core_20260715_000351
```

### 4.1 Visão geral das threads

```
> clrthreads
```

**O que observar:**
```
ThreadCount:      45      ← total de threads gerenciadas
BackgroundThread: 40      ← número alto aqui é sinal de leak
DeadThread:       5

DBG  ID   State      Name
...
30   23   2021020    Ukn   ← sem nome, mesmo State repetido = suspeito
31   24   2021020    Ukn
32   25   2021020    Ukn   ← padrão de leak: N threads, mesmo state, sem nome
```

---

### 4.2 Inspeciona call stack das threads suspeitas

```
> setthread 30
> clrstack
```

**Confirma leak quando ver:**
```
System.Threading.Thread.Sleep
ThreadLeakService.RecurseAndSleep @ linha 77
ThreadLeakService.RecurseAndSleep @ linha 77
ThreadLeakService.RecurseAndSleep @ linha 77  ← stack profundo = memória consumida
```

> Repete para 2-3 threads do mesmo grupo para confirmar o padrão.

---

### 4.3 Conta threads por tipo no heap

```
> dumpheap -stat
> dumpheap -type Thread
```

> Se `System.Threading.Thread` aparecer com contagem alta → confirma criação explícita via `new Thread()`.

---

### 4.4 Verifica estado do ThreadPool

```
> threadpool
```

**Saída esperada quando TP está saudável:**
```
CPU utilization: 8%
Worker Thread: Total: 8, Running: 4, Idle: 4
Work Request Queue: 0
```

> Se o ThreadPool estiver saudável mas `clrthreads` mostrar muitas threads → confirmado `new Thread()` fora do pool.

---

### 4.5 Sai do analisador

```
> exit
```

---

## Fluxo de decisão

```
dotnet-counters
working_set subindo?
        │
        ▼
dotnet-stack report
Mesmo call stack repetido?
    ├── Sim → causa identificada
    └── Não → precisa do dump
                    │
                    ▼
            dotnet-dump collect
                    │
                    ▼
            clrthreads → identifica grupo suspeito
                    │
                    ▼
            setthread + clrstack → confirma call stack e arquivo/linha
                    │
                    ▼
            threadpool → descarta starvation de TP
```

---

## Referência rápida — comandos dentro do `dotnet-dump analyze`

| Comando | O que faz |
|---|---|
| `clrthreads` | Lista todas as threads gerenciadas com estado |
| `setthread <dbg-id>` | Seleciona thread para inspeção |
| `clrstack` | Call stack da thread selecionada |
| `threadpool` | Estado do ThreadPool |
| `dumpheap -stat` | Contagem de objetos no heap por tipo |
| `dumpheap -type Thread` | Filtra apenas objetos `Thread` no heap |
| `help` | Lista todos os comandos disponíveis |
| `exit` | Sai do analisador |
