# Diagnóstico de Memory Leak — Guia de Comandos

## 0. Pré-requisitos

```bash
# Instala as ferramentas se ainda não tiver
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-gcdump
dotnet tool install -g dotnet-dump

# Confirma instalação
dotnet tool list -g

# Descobre o PID da API
dotnet-counters ps
```

---

## 1. Confirma o sintoma — `dotnet-counters`

```bash
dotnet-counters monitor --process-id <pid> --refresh-interval 1 --counters System.Runtime
```

**Observar (nomes OpenTelemetry do .NET 10):**

```
dotnet.gc.collections[gc.heap.generation=gen2]      → nº de coleções completas
dotnet.gc.heap.total_allocated                      → total alocado desde o start
dotnet.gc.last_collection.heap.size[gen2]           → heap vivo após a última coleta
dotnet.gc.last_collection.memory.committed_size     → memória reservada pelo GC
dotnet.gc.last_collection.heap.fragmentation.size   → buracos entre objetos vivos
dotnet.process.memory.working_set                   → memória total do processo
```

### Capture um baseline antes de qualquer carga

Sem baseline você não distingue **crescimento** de **leak**.

```bash
# Deixe rodando 2-3 min com a app ociosa, anote os valores
dotnet-counters monitor --process-id <pid>
```

### Matriz de decisão

| Evidência | Leitura |
|---|---|
| `gen2` sobe e **não cai** após ociosidade | Retenção por raiz viva → **LEAK** |
| `gen0` alto, `total_allocated` alto, `gen2` estável | Pressão transitória → normal sob carga |
| `heap.size` ≈ `total_allocated` (retenção ~100%) | Praticamente nada morreu → **LEAK** |
| `heap.size[gen0]` = 0 com heap alto | Tudo já promoveu, sem churn → **LEAK** |
| `committed_size` ≈ `heap.size` ≈ `working_set` | Consumo é heap gerenciado puro |
| `working_set` ≫ `heap.size` | Consumo fora do heap (buffers, sockets, interop) |
| `fragmentation` alto sem crescimento de heap | Fragmentação, não leak |

> **Teste decisivo e barato**: pare a carga, aguarde ~60s de ociosidade e olhe de novo.
> Heap voltou ao baseline → era pressão. Heap continua alto → retenção real.

---

## 2. Descobre *o que* está vazando — `dotnet-gcdump`

Leve o suficiente para produção. Usa EventPipe, não pausa significativamente.

```bash
# Baseline (logo após o startup)
dotnet-gcdump collect --process-id <pid> --output baseline.gcdump

# Após a carga
dotnet-gcdump collect --process-id <pid> --output afterleak.gcdump
```

### Relatório por tipo

```bash
dotnet-gcdump report afterleak.gcdump
```

**Saída (ordenada por tamanho):**

```
Size (Bytes)    Count   Type
============    =====   ====
 250,000,000    5,000   EventSubscriber        ← suspeito
 100,000,000   10,000   Customer               ← suspeito
  50,000,000    5,000   System.Byte[]
     500,000      500   System.Timers.Timer    ← suspeito
```

### Comparação antes/depois (elimina falso positivo)

```bash
dotnet-gcdump report baseline.gcdump  > baseline.txt
dotnet-gcdump report afterleak.gcdump > afterleak.txt

# Linux/macOS
diff baseline.txt afterleak.txt

# Windows PowerShell
Compare-Object (Get-Content baseline.txt) (Get-Content afterleak.txt)
```

> Objetos legítimos de longa vida aparecem nos **dois** dumps com contagem parecida.
> O que cresce desproporcionalmente é o candidato.

> ⚠️ O `gcdump` força uma **GC de geração 2 completa** para enumerar o heap vivo. Isso pausa a aplicação e distorce a latência naquele instante. Anote o timestamp de cada coleta para excluir esses pontos da leitura de p95.

**Limitação**: o gcdump mostra o grafo de objetos vivos, não o histórico. Ele responde *o quê*, não *por quê*. Para a cadeia de referência, precisa do `dotnet-dump`.

---

## 3. Captura o dump — `dotnet-dump collect`

```bash
# Dump completo
dotnet-dump collect --process-id <pid>

# Em diretório específico
dotnet-dump collect --process-id <pid> --output ~/diagnostics/

# Apenas heap gerenciado (menor, suficiente para leak)
dotnet-dump collect --process-id <pid> --type Heap
```

**Tipos disponíveis:**

| Tipo | Conteúdo | Quando usar |
|---|---|---|
| `Full` | Memória, threads, handles | Diagnóstico profundo (padrão) |
| `Heap` | Apenas heap gerenciado | Memory leak — suficiente na maioria dos casos |
| `Mini` | Threads e stacks | Crash, não memória |
| `Triage` | Mínimo | Triagem inicial |

**Saída esperada:**

```
Writing full to /home/user/diagnostics/core_20260715_000351
Complete
```

---

## 4. Analisa o dump — `dotnet-dump analyze`

```bash
dotnet-dump analyze core_20260715_000351
```

### 4.1 Identifica os tipos suspeitos

```
> dumpheap -stat
```

**O que observar:**

```
              MT     Count     TotalSize Class Name
72767dcde8c8   715,800    17,179,200 ...Api.Services.Memory.EventLeakService+EventSubscriber
72767d57b718         2    33,554,480 ...Api.Models.Customer[]
727678f5dd58 2,886,141   107,346,458 System.String
72767d57afa0 2,861,000   137,328,000 ...Api.Models.Customer                    ← suspeito
72767baaada8 3,585,001   229,440,064 System.Threading.TimerCallback
72767dcf3508 3,585,000   315,480,000 System.Timers.Timer                       ← suspeito
72767baaabe8 3,585,002   344,160,192 System.Threading.TimerQueueTimer          ← suspeito
727679a757e8 3,577,236 1,874,727,072 System.Byte[]                             ← maior consumidor
```

> Leitura: `System.Byte[]` domina em bytes (1,87 GB), mas é **consequência** — arrays não se referenciam sozinhos. Os candidatos reais são os tipos que os seguram. O trio `Timer` / `TimerCallback` / `TimerQueueTimer` com ~3,58 milhões de instâncias cada é a assinatura mais forte: contagens idênticas indicam um timer criado e nunca parado por operação.

| Coluna | Significado |
|---|---|
| `MT` | *Method Table* — identificador único do tipo |
| `Count` | Quantidade de objetos desse tipo |
| `TotalSize` | Tamanho total ocupado pelo tipo |
| `Class Name` | Nome completo do tipo |

Filtros úteis:

```
> dumpheap -stat -gen 2            # só objetos de longa vida
> dumpheap -type Customer -stat    # filtra por substring do nome
> dumpheap -mt 72767d57afa0        # filtra pela Method Table (coluna MT do -stat)
> dumpheap -min 85000              # só objetos que foram para LOH
```

---

### 4.2 Pega o endereço de uma instância

```
> dumpheap -type JacksonVeroneze.NET.DotnetDiagnosticsLab.Api.Models.Customer -short
```

**Saída:**

```
723751e6f638
723751e6f8a0
723751e6fb08
723751e6fd70
723751e6ffd8
... (4997 mais)
```

> `-short` devolve só os endereços — conveniente para copiar direto para o `gcroot`.
> Qualquer endereço da lista serve: se o padrão é o mesmo, a raiz é a mesma.

---

### 4.3 🎯 Rastreia a raiz — `gcroot`

**Este é o comando central do diagnóstico.** Ele mostra a cadeia completa de referências que mantém o objeto vivo.

```
> gcroot 723751e6ffd8
```

**Saída:**

```
Caching GC roots, this may take a while.
Subsequent runs of this command will be faster.
Thread 10cfef:
    723576ffcae0 727678384d0c System.Diagnostics.Tracing.CounterGroup.PollForValues()
        r14:
          -> 723668002020     System.Object[]
          -> 72368e174600     System.Collections.Generic.List<...Api.Models.Customer>
          -> 723743400048     ...Api.Models.Customer[]
          -> 723751e6ffd8     ...Api.Models.Customer

Found 1 unique roots.
```

**Como ler a cadeia — de cima para baixo:**

```
Thread 10cfef → registrador r14        ← raiz reportada (ver nota abaixo)
   └─> System.Object[]                 ← array interno do List<T>
        └─> List<Customer>             ← a coleção que acumula
             └─> Customer[]            ← buffer interno do List
                  └─> Customer         ← o objeto que vaza
```

> ⚠️ **Raiz em Thread não significa objeto temporário aqui.** O `gcroot` reportou o caminho mais curto que encontrou: um registrador (`r14`) da thread do `CounterGroup.PollForValues()` — a própria thread do `dotnet-counters`. Isso é um artefato do momento da captura, não a raiz semântica.
>
> O que importa é o **elo seguinte**: o `List<Customer>` que aparece na cadeia. Ele é o acumulador real. Para achar quem o segura, rode `gcroot` no endereço do próprio List:
>
> ```
> > gcroot 72368e174600
> ```
>
> Se a saída apontar `HandleTable` / `strong handle`, confirma-se a raiz estática.

O topo da cadeia é **quem** mantém tudo vivo. O caminho é **por quê**.

**Tipos de raiz e o que significam:**

| Raiz | Origem | Interpretação |
|---|---|---|
| `HandleTable` / `strong handle` | Campo `static`, singleton do DI, GCHandle | Vive pelo tempo do processo — leak provável |
| `Thread <id>` / `rbp-XX` | Variável local na stack | Temporário — some quando o método retorna |
| `Timer queue` | Timer ativo na fila do runtime | Raiz **fora do heap gerenciado** |
| `Ephemeral Segment` | Objeto em gen0/gen1 referenciando | Verifique quem referencia o referenciador |
| `Finalizer queue` | Aguardando finalização | Transitório, mas pode acumular |

> ⚠️ **Raiz dupla**: um `Timer` ativo aparece com **duas** raízes — o objeto que o segura *e* a fila de timers do runtime. Mesmo removendo a coleção estática, o objeto sobrevive porque o timer está rodando. Só `Stop()` + `Dispose()` libera.

> Repita o `gcroot` em 2-3 endereços do mesmo tipo para confirmar que a raiz é a mesma. Se divergir, há mais de uma causa.

---

### 4.4 Inspeciona os elos da cadeia — `dumpobj`

Com a cadeia em mãos, examine cada nó para confirmar a hipótese.

```
> dumpobj 723751e6ffd8
```

**Saída:**

```
Name:        ...Api.Models.Customer
MethodTable: 000072767d57afa0
Size:        48(0x30) bytes
Fields:
              MT    Field   Offset          Type VT     Attr            Value Name
0000727679007940  400004f       18   System.Guid  1 instance 0000723751e6fff0 <Id>k__BackingField
0000727678f5dd58  4000050        8 System.String  0 instance 0000723751e6fda0 <Name>k__BackingField
0000727679a757e8  4000051       10 System.Byte[]  0 instance 0000723751e6fdc8 <Data>k__BackingField
```

**Como ler:**

| Coluna | Significado |
|---|---|
| `Field` | Token do campo nos metadados |
| `Offset` | Deslocamento em bytes dentro do objeto |
| `VT` | `1` = value type (inline no objeto); `0` = referência |
| `Value` | Endereço do objeto referenciado — use no próximo `dumpobj` |

> O `Customer` tem 48 bytes, mas dois dos três campos são **referências**: `Name` (`System.String`) e `Data` (`System.Byte[]`). Os 48 bytes não incluem o conteúdo apontado — é por isso que `Size` engana e o `objsize` da próxima etapa é necessário.
>
> Para inspecionar o array retido, siga o `Value` do campo `Data`:
>
> ```
> > dumpobj 723751e6fdc8
> ```

**Campos que confirmam cada padrão:**

| Padrão | Campo a inspecionar | Sinal |
|---|---|---|
| Evento não desinscrito | `_invocationCount` no delegate | Contagem alta e crescente |
| Cache sem expiração | `_count` / `Count` na coleção | Cresce sem limite |
| Closure capturando `this` | Campos do `<>c__DisplayClass` | Contém o objeto grande |
| CTS não disposed | `_registrations` no CTS pai | Lista de registrations acumulada |

---

### 4.5 Mede o custo real — `objsize`

`dumpheap` mostra o tamanho do objeto em si. `objsize` mostra o tamanho **transitivo** — incluindo tudo que ele referencia.

```
> objsize 723751e6ffd8
```

**Saída:**

```
Objects which 723751e6ffd8 (...Api.Models.Customer) transitively keep alive:
         Address               MT           Size
    723751e6ffd8     72767d57afa0             48
    723751e6fda0     727678f5dd58             38
    723751e6fdc8     727679a757e8            524
Statistics:
          MT Count TotalSize Class Name
727678f5dd58     1        38 System.String
72767d57afa0     1        48 ...Api.Models.Customer
727679a757e8     1       524 System.Byte[]
Total 3 objects, 610 bytes
```

> O `dumpheap` reportou 48 bytes para o `Customer`. O custo real é **610 bytes** — 12,7× maior. A diferença está no `Byte[]` de 524 bytes e na `String` de 38.
>
> **Projeção:** 2.861.000 instâncias × 610 bytes ≈ **1,74 GB**. Isso explica a maior parte do `System.Byte[]` visto no `dumpheap -stat` (1,87 GB) — os arrays não são um leak independente, são carga transportada pelos `Customer`.

Essa multiplicação é a razão de usar `objsize`: ordenar por `Count` ou por `Size` no `dumpheap` leva à conclusão errada sobre onde está a memória.

---

### 4.6 Confirma raízes fora do heap

**GC Handles** — referências que o GC não pode coletar:

```
> gchandles
```

```
Handle           Type       Object          Size   Data Type
723668001018     Strong     72368e174600      88   System.Collections.Generic.List<...Customer>
723668001020     Strong     723668002020   10,024   System.Object[]
723668001028     Pinned     723668003000    4,096   System.Object[]

Statistics:
     Strong Handles:  1,234
     Pinned Handles:     45
```

| Tipo | Mantém vivo? | Observação |
|---|---|---|
| `Strong` | Sim | Referência forte normal |
| `Pinned` | Sim | Também impede movimento — causa fragmentação |
| `WeakShort` | Não | Referência fraca |
| `WeakLong` | Não | Sobrevive à finalização |

**Tasks pendentes**:

```
> dumpasync
```

| State | Significado |
|---|---|
| `-2` | Task criada mas não iniciada |
| `-1` | Task executando |
| `≥ 0` | Task em await no ponto N |

---

## Fluxo de decisão

```
dotnet-counters
gen2 sobe e não cai após ociosidade?
        │
        ├── Não → pressão transitória, não é leak
        │
        └── Sim → LEAK confirmado
                    │
                    ▼
            dotnet-gcdump collect + report
            QUAIS tipos cresceram? (diff baseline × afterleak)
                    │
                    ▼
            dotnet-dump collect --type Heap
                    │
                    ▼
            dumpheap -stat → identifica tipo suspeito por Count/TotalSize
                    │
                    ▼
            dumpheap -type <Tipo> -short → pega um endereço
                    │
                    ▼
            🎯 gcroot <address> → CADEIA DE REFERÊNCIA (por que está vivo)
                    │
                    ▼
            dumpobj <cada elo> → confirma campos (_invocationCount, Count)
                    │
                    ▼
            objsize -detail → mede o custo transitivo real
                    │
                    ▼
            gchandles → raízes fora do heap
```

---

## Padrões de leak e assinatura no `gcroot`

| Padrão | Assinatura na cadeia | Confirmação |
|---|---|---|
| **Coleção estática** | `strong handle` → `Service` → `List`/`Dictionary` → objeto | `Count` da coleção cresce sem limite |
| **Evento não desinscrito** | `strong handle` → `Publisher` → `EventHandler` → `Object[]` → subscriber | `_invocationCount` alto |
| **Cache sem expiração** | `strong handle` → `CacheService` → `ConcurrentDictionary` → `Node[]` → item | Sem `SizeLimit` nem expiração configurada |
| **Closure capturando `this`** | `Timer queue` → `Timer` → delegate → `<>c__DisplayClass` → objeto grande | Campo do DisplayClass contém o array |
| **Timer não parado** | Raiz **dupla**: holder + `Timer queue` | `gcroot` retorna 2 unique roots |
| **CTS não disposed** | `strong handle` → CTS pai → `_registrations` → linked CTS | `dumpasync` com tasks pendentes |

---

## Leak × Fragmentação

| Característica | Memory Leak | Fragmentação |
|---|---|---|
| **Causa** | Referências mantidas indevidamente | Buracos entre objetos vivos |
| **Heap cresce?** | Sim, continuamente | Pode não crescer |
| **OOM ocorre por** | Esgotamento total | Falta de bloco contíguo |
| **GC ajuda?** | Não (objetos estão "vivos") | Parcialmente (compactação) |
| **Ferramenta** | `gcroot` (quem referencia?) | `dumpheap -type Free`, `fragmentation.size` |

---

## Diagnóstico em containers

```bash
# Executar dentro do container
docker exec -it <container_id> dotnet-gcdump collect -p 1
docker exec -it <container_id> dotnet-dump collect -p 1

# Copiar para fora
docker cp <container_id>:/app/core_20260715_000351 ./dump.dmp
```

**Dockerfile com ferramentas embutidas:**

```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS base

RUN dotnet tool install --tool-path /tools dotnet-dump
RUN dotnet tool install --tool-path /tools dotnet-gcdump
RUN dotnet tool install --tool-path /tools dotnet-counters

ENV PATH="/tools:${PATH}"
```

> Alternativa sem modificar a imagem: `dotnet-monitor` como sidecar, expondo coleta via HTTP.

---

## Ordem de segurança em produção

```
dotnet-counters   → impacto zero, sempre primeiro
       ↓
dotnet-gcdump     → baixo impacto, mas força GC gen2
       ↓
dotnet-dump       → pausa o processo, arquivo grande, dados sensíveis
       ↓
dotMemory/PerfView → análise offline
```

---


## Referências

- [dotnet-dump](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-dump)
- [dotnet-gcdump](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-gcdump)
- [dotnet-counters](https://learn.microsoft.com/dotnet/core/diagnostics/dotnet-counters)
- [Garbage Collection](https://learn.microsoft.com/dotnet/standard/garbage-collection)
- [Diagnóstico no .NET](https://learn.microsoft.com/dotnet/core/diagnostics)
- Tiago Tartari — [Profiling de Memória no .NET com dotnet-dump e dotnet-gcdump](https://tiagotartari.net/profiling-de-memoria-no-net-com-dotnet-dump-e-dotnet-gcdump.html)
