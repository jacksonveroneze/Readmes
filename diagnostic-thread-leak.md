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

## 5. Coleta insumo para visualizar no dotTrace

O dotTrace resolve símbolos gerenciados e exibe timeline de threads, call stacks completos e tempo em cada estado — mais visual que o `dotnet-dump` para identificar thread leak.

### 5.1 Opção A — Coleta direto pelo dotTrace (recomendado)

É o caminho mais simples e já entrega símbolos resolvidos sem conversão.

**No Rider:**
`Run` → `Attach to Process` → seleciona o processo da API → modo **Sampling** → `Run`

Enquanto o dotTrace coleta, bate no endpoint de thread leak. Após 20-30 segundos clica em `Stop`.

**No dotTrace standalone:**
```bash
# Anexa ao processo em modo Sampling
dotTrace attach <pid> --save-to=~/diagnostics/threadleak.dtp
```

---

### 5.2 Opção B — Coleta via `dotnet-trace` e abre no dotTrace

Use quando não for possível anexar o dotTrace diretamente ao processo.

```bash
# Coleta com provider que inclui sampling gerenciado
dotnet-trace collect --process-id <pid> \
  --providers "Microsoft-Windows-DotNETRuntime:0x1CCBD:5,Microsoft-DotNETCore-SampleProfiler:0xF00000000000:5" \
  --duration 00:00:30 \
  --output ~/diagnostics/threadleak.nettrace
```

Abre o `.nettrace` no dotTrace:
`File` → `Open` → seleciona o arquivo `.nettrace`

> ⚠️ Arquivos `.nettrace` abertos no dotTrace podem mostrar `[Unresolved]` no call stack — prefira a Opção A sempre que possível.

---

### 5.3 O que observar no dotTrace após abrir

**Timeline (painel central):**
- Dezenas de threads `.NET` com tempo idêntico em estado `Sleep`
- Padrão visual: muitas linhas horizontais paralelas da mesma cor = threads bloqueadas simultaneamente

```
ID       Name      ms      %
2710002  .NET    3,157ms   ▓▓▓▓▓▓▓▓▓▓
2710003  .NET    3,157ms   ▓▓▓▓▓▓▓▓▓▓   ← mesmo tempo = mesmo comportamento
2710004  .NET    3,157ms   ▓▓▓▓▓▓▓▓▓▓
...
```

**Subsystems (painel esquerdo):**
```
Sleep       99.9%   ← threads bloqueadas, sem trabalho útil
GC Wait      0.01%
```

**Call Stack (painel direito) — seleciona qualquer thread suspeita:**
```
System.Threading.Thread.Sleep
  ThreadLeakService.RecurseAndSleep(Int32, Int32)
    ThreadLeakService.RecurseAndSleep(Int32, Int32)   ← recursão profunda
      ThreadLeakService.RecurseAndSleep(Int32, Int32)
        ...
```

**Filtros úteis no dotTrace:**
- `Thread State` → filtra por `Waiting` para isolar threads bloqueadas
- `Visible Threads` → seleciona apenas `.NET` para excluir threads nativas do runtime
- Clica em `Flame Graph` no painel de Call Stack para visualização hierárquica

---

### 5.4 Diferença entre threads `.NET` e `Native` no dotTrace

| Tipo | Origem | Call Stack | Quando investigar |
|---|---|---|---|
| `.NET` | `new Thread()`, ThreadPool, runtime gerenciado | Resolvido — mostra métodos C# | Sempre — é onde está seu código |
| `Native` | Runtime interno, Kestrel I/O, libs nativas | `[Unresolved]` ou símbolos nativos | Apenas se suspeitar de interop ou problema no próprio runtime |

> Threads leaked via `new Thread()` sempre aparecem como `.NET` — foque nelas.

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
