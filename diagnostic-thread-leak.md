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

### 4.3 Inspeciona o objeto Thread no heap

O `ThreadObj` do `clrthreads` é um ponteiro nativo interno do CLR — não use diretamente com `dumpobj`.
Para inspecionar o objeto gerenciado e ver o nome da thread:

```
# Lista objetos Thread no heap pelo Method Table
> dumpheap -mt <MT da linha System.Threading.Thread>

# Inspeciona um endereço da lista
> dumpobj <Address>
```

**Campos relevantes na saída:**

```
_name          leaked-thread-a1b2c3   ← nome definido via Thread.Name (nulo se não setado antes do dump)
_isDead        0                      ← 0 = viva, 1 = encerrada
_isThreadPool  0                      ← 0 = new Thread(), 1 = ThreadPool worker
_managedThreadId  138                 ← cruza com coluna ID do clrthreads
```

> ⚠️ O `_name` aparece nulo se a thread foi nomeada após o `Thread.Start()` ou se o dump foi capturado antes da atribuição. Sempre nomeie a thread **antes** do `Start()`.

Para listar apenas threads vivas:

```
> dumpheap -mt <MT> -live
```

---

### 4.4 Conta threads por tipo no heap

```
> dumpheap -stat
> dumpheap -type System.Threading.Thread
```

> Se `System.Threading.Thread` aparecer com contagem alta → confirma criação explícita via `new Thread()`.

---

### 4.5 Verifica estado do ThreadPool

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

### 4.6 Sai do analisador

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
            clrthreads → identifica grupo suspeito por State e quantidade
                    │
                    ▼
            setthread + clrstack → confirma call stack e arquivo/linha
                    │
                    ▼
            dumpheap -mt + dumpobj → inspeciona _name, _isDead, _isThreadPool
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
dotTrace attach <pid> --save-to=~/diagnostics/threadleak.dtp
```

---

### 5.2 Opção B — Coleta via `dotnet-trace` e abre no dotTrace

Use quando não for possível anexar o dotTrace diretamente ao processo.

```bash
dotnet-trace collect --process-id <pid> \
  --providers "Microsoft-Windows-DotNETRuntime:0x1CCBD:5,Microsoft-DotNETCore-SampleProfiler:0xF00000000000:5" \
  --duration 00:00:30 \
  --output ~/diagnostics/threadleak.nettrace
```

Abre o `.nettrace` no dotTrace: `File` → `Open` → seleciona o arquivo `.nettrace`

> ⚠️ Arquivos `.nettrace` abertos no dotTrace podem mostrar `[Unresolved]` no call stack — prefira a Opção A sempre que possível.

---

### 5.3 O que observar no dotTrace após abrir

**Timeline (painel central):**

- Dezenas de threads `.NET` com tempo idêntico em estado `Sleep`
- Padrão visual: muitas linhas horizontais paralelas = threads bloqueadas simultaneamente

```
ID       Name      ms
2710002  .NET    3,157ms   ▓▓▓▓▓▓▓▓▓▓
2710003  .NET    3,157ms   ▓▓▓▓▓▓▓▓▓▓   ← mesmo tempo = mesmo comportamento
2710004  .NET    3,157ms   ▓▓▓▓▓▓▓▓▓▓
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
```

**Filtros úteis no dotTrace:**

- `Thread State` → filtra por `Waiting` para isolar threads bloqueadas
- `Visible Threads` → seleciona apenas `.NET` para excluir threads nativas do runtime
- `Flame Graph` no painel de Call Stack para visualização hierárquica

---

### 5.4 Tipos de threads no dotTrace

| Tipo | Origem | Call Stack | Quando investigar |
|---|---|---|---|
| `.NET` | `new Thread()`, tasks, qualquer thread gerenciada sem nome específico | Resolvido — mostra métodos C# | Sempre — é onde está seu código |
| `ThreadPool Worker` | Threads do ThreadPool — `Task.Run`, `QueueUserWorkItem` | Resolvido | `threadpool-queue-length` alto, starvation, `.Result` / `.Wait()` bloqueando |
| `Main` | Thread principal do processo — `Program.cs` | Resolvido | Se a thread principal estiver bloqueada ou com alta latência |
| `CLR Worker` | Threads internas do CLR — GC, type loader | Parcialmente resolvido | Pausas longas de GC ou alto tempo de coleta |
| `JIT Thread` | Compilação JIT em background (.NET 6+) | Parcialmente resolvido | Alto consumo de CPU no startup ou primeiro acesso a métodos |
| `Finalizer` | Thread única do GC responsável por finalizers | Resolvido | `finalization-pending-count` alto ou objetos com `~Destructor` acumulando |
| `Kestrel` | Worker threads do servidor HTTP | Resolvido | Latência de requests ou saturação de conexões |
| `Console` | Thread de I/O do console | Nativo | Raramente — apenas se I/O de log estiver bloqueando |
| `Native` | Runtime interno, libs nativas, interop | `[Unresolved]` | Apenas se suspeitar de problema em interop ou no próprio runtime |
| `Timer` | Callbacks de `System.Threading.Timer` | Resolvido | `dotnet.timer.count` alto — timers sem dispose acumulando |

---

**Threads que indicam problema quando aparecem em quantidade:**

| Tipo | Contagem normal | Sinal de problema |
|---|---|---|
| `.NET` genérico | 20–40 | Dezenas a centenas = thread leak via `new Thread()` |
| `ThreadPool Worker` | igual ao número de cores | Crescendo indefinidamente = starvation por bloqueio síncrono |
| `CLR Worker` | 2–4 | Acima de 8 = pressão intensa no GC |
| `Finalizer` | 1 (sempre única) | Se lenta = finalizer queue saturada com muitos objetos pendentes |
| `Timer` | Variável | Crescimento contínuo = `System.Threading.Timer` sem dispose |

> A `Finalizer` thread é sempre única no processo — não confunda lentidão dela com múltiplas instâncias.
> Threads leaked via `new Thread()` aparecem como `.NET` genérico — sem nome se `Thread.Name` não foi setado antes do `Start()`.
> `ThreadPool Worker` e `.NET` genérico são distintos no dotTrace — importante separar para não confundir starvation com thread leak.

---

## Referência rápida — comandos dentro do `dotnet-dump analyze`

| Comando | O que faz |
|---|---|
| `clrthreads` | Lista todas as threads gerenciadas com estado |
| `setthread <dbg-id>` | Seleciona thread para inspeção |
| `clrstack` | Call stack da thread selecionada |
| `threadpool` | Estado do ThreadPool |
| `dumpheap -stat` | Contagem de objetos no heap por tipo |
| `dumpheap -type System.Threading.Thread` | Filtra objetos `Thread` no heap |
| `dumpheap -mt <MT> -live` | Lista apenas instâncias vivas de um tipo |
| `dumpobj <Address>` | Inspeciona campos de um objeto gerenciado |
| `help` | Lista todos os comandos disponíveis |
| `exit` | Sai do analisador |
