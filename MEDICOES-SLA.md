# MEDIÇÕES DO SLA — LattesScore

> **Estado:** o repositório recebido já contém medições reais do cadastro de usuários. O segundo serviço (importação de currículo Lattes) está instrumentado com um cenário k6 reproduzível, mas **ainda precisa ser executado** contra uma instância ativa com CouchDB. Não se devem tratar os dados ausentes como resultados.

## Painel visual

Abra [`docs-sla.html`](docs-sla.html) no navegador para comparar as séries de latência, vazão e concorrência. O painel lê [`docs-sla-data.json`](docs-sla-data.json), gerado a partir dos JSONs k6 existentes. Para publicar como GitHub Pages, habilite Pages para a branch/pasta que contém esses arquivos; o endereço público depende do proprietário e do nome do repositório.

## Serviço 1 — Cadastro de usuário

- **Endpoint:** `POST /api/usuarios/registar`
- **Operações:** inserção (INSERT) no MySQL.
- **Arquivos envolvidos:**
  - [`UsuarioController.java`](src/main/java/com/lattesscore/backend/controller/UsuarioController.java)
  - [`UsuarioService.java`](src/main/java/com/lattesscore/backend/service/UsuarioService.java)
  - [`UsuarioRepository.java`](src/main/java/com/lattesscore/backend/repository/UsuarioRepository.java)
  - [`Usuario.java`](src/main/java/com/lattesscore/backend/entity/Usuario.java)
  - [`application.properties`](src/main/resources/application.properties)
- **Código-fonte dos testes:** [`teste-cadastro.js`](k6/teste-cadastro.js), [`teste-cadastro-100.js`](k6/teste-cadastro-100.js), [`teste-cadastro-200.js`](k6/teste-cadastro-200.js).
- **Resultados brutos:** [`resultado-carga.json`](resultado-carga.json), [`resultados-cadastro.json`](resultados-cadastro.json), [`resultados-cadastro-100.json`](resultados-cadastro-100.json), [`resultados-cadastro-200.json`](resultados-cadastro-200.json).
- **Data das medições:** 09/11/2026 (timestamp dos arquivos JSON).
- **Configuração do ambiente:** aplicação Spring Boot na porta 8080; banco MySQL configurado em `localhost:3306/lattesscore`; carga k6 direcionada a `localhost:8080`. **CPU, RAM, versões de MySQL/k6, sistema operacional, topologia/container e recursos alocados não estão documentados nos artefatos recebidos**, portanto devem ser preenchidos antes de uma comparação científica/reprodutível.

### Resultados disponíveis

Valores calculados dos pontos `http_req_duration` e `http_reqs` dos JSONs originais. Latência e p95 em ms; vazão média = número de requisições dividido pelo intervalo entre o primeiro e o último ponto de duração. VU máx. vem do `vus_max` do k6.

| Execução | VUs máx. | Requisições | Latência média | p95 | Vazão média | Respostas |
|---|---:|---:|---:|---:|---:|---|
| `resultado-carga.json` | 20 | 1.039 | 15,75 ms | 29,89 ms | 10,45 req/s | 1.039 × HTTP 200 |
| `resultados-cadastro.json` | 20 | 1.033 | 20,94 ms | 39,06 ms | 10,42 req/s | 1.033 × HTTP 200 |
| `resultados-cadastro-100.json` | 100 | 5.213 | 14,25 ms | 23,97 ms | 52,30 req/s | 5.213 × HTTP 200 |
| `resultados-cadastro-200.json` | 200 | 10.512 | 6,96 ms | 11,16 ms | 105,85 req/s | 10.512 × HTTP 200 |

**Atenção à interpretação:** as médias não crescem monotonicamente com VUs. Isso não demonstra que 200 VUs sejam mais eficientes ou que esse seja o limite sustentável: os ensaios podem ter sido executados em condições diferentes, os cenários usam `sleep(1)` e os resultados disponíveis não documentam recursos da máquina nem saturação. O pico de VUs configurado é uma concorrência aplicada, não o limite máximo comprovado do sistema. Os JSONs contêm séries temporais; o painel mostra evolução em janelas de 5 segundos.

### Hipóteses de gargalo — cadastro

1. **Pool de conexões JDBC:** sob concorrência, threads podem aguardar conexão com MySQL; validar métricas de conexões ativas/ociosas e tempo de espera.
2. **MySQL / I/O / transações:** cada chamada persiste uma linha com `email` único e chave `IDENTITY`; índice único, escrita em disco, locks e flush podem elevar a latência.
3. **Contenção no índice de email ou limite de conexões:** observar slow query log, locks, CPU e IOPS do MySQL.
4. **Aplicação JVM:** threads HTTP, heap e pausas de GC podem limitar throughput antes do banco.

São hipóteses técnicas derivadas do caminho de execução, não gargalos comprovados pelas medições atuais.

## Serviço 2 — Importação de currículo Lattes

- **Endpoint:** `POST /api/lattes/importar` (multipart/form-data, campos `idLattes` e `file`).
- **Operações:** inserção de documento no CouchDB.
- **Arquivos envolvidos:**
  - [`LattesController.java`](src/main/java/com/lattesscore/backend/controller/LattesController.java)
  - [`AvaliacaoService.java`](src/main/java/com/lattesscore/backend/service/AvaliacaoService.java)
  - [`CouchDbService.java`](src/main/java/com/lattesscore/backend/couch/CouchDbService.java)
  - [`DocumentoLattes.java`](src/main/java/com/lattesscore/backend/couch/DocumentoLattes.java)
  - [`CouchDbResponse.java`](src/main/java/com/lattesscore/backend/couch/CouchDbResponse.java)
  - [`application.properties`](src/main/resources/application.properties)
- **Código-fonte de medição:** [`teste-importacao-lattes.js`](k6/teste-importacao-lattes.js) e [`curriculo.xml`](k6/fixtures/curriculo.xml).
- **Data:** cenário preparado em 09/11/2026; **medição pendente**.
- **Configuração:** Spring Boot na porta 8080 e CouchDB configurado em `127.0.0.1:5984`, banco `lattesscore_documentos`. O CouchDB precisa estar ativo e autenticado. Recursos de máquina/container não foram informados.

### Execução para obter as medições

Na raiz do projeto, com aplicação, MySQL e CouchDB ativos:

```bash
k6 run -e BASE_URL=http://localhost:8080   k6/teste-importacao-lattes.js   --summary-export=resultados-importacao-lattes.json
```

O cenário aplica rampas de 5, 10 e 20 VUs, registra `http_req_duration`, `http_reqs`, `vus`/`vus_max` e verifica HTTP 200. O threshold p95 de 1s é um critério inicial de teste, não um SLA já acordado. Após executar, adicionar `resultados-importacao-lattes.json` ao repositório e incluir a série no painel para comparação direta.

### Hipóteses de gargalo — importação

1. **Leitura do arquivo em memória:** `file.getBytes()` materializa o XML inteiro; arquivos grandes e várias importações simultâneas podem pressionar heap e GC.
2. **Cliente HTTP por requisição:** `new RestTemplate()` é criado em cada chamada; investigar custo de criação e ausência de pool/reuso de conexões.
3. **Serialização e payload:** o XML é inserido como conteúdo de um documento JSON, ampliando payload e custo de serialização/transporte.
4. **CouchDB / disco:** cada requisição grava um documento com ID aleatório; latência de persistência, compactação, replicação e I/O podem limitar vazão.
5. **Timeouts e falhas externas:** não há configuração explícita de timeout no `RestTemplate`; indisponibilidade/lentidão do CouchDB pode ocupar threads do servidor.

Validar com métricas do CouchDB, latência HTTP entre serviços, heap/GC, CPU, disco e tamanho dos XMLs. A medição k6 deve usar o mesmo tamanho de arquivo e ambiente documentado em todas as comparações.

## Reprodutibilidade e próximos passos

1. Registrar SO, CPU, RAM, versões Java/Spring Boot/MySQL/CouchDB/k6, containers e limites de CPU/memória.
2. Executar os dois serviços no mesmo ambiente e com dados de teste controlados.
3. Repetir cada perfil pelo menos três vezes; guardar JSON bruto e não sobrescrever execuções.
4. Comparar média, p95/p99, req/s, VUs, taxa de erro e recursos do servidor. O requisito pede latência média; p95 foi incluído como indicador complementar.
5. Não interpretar VUs máximos do script como limite real: aumentar carga progressivamente até violação de latência/erro ou saturação de recursos.

## Referências de implementação

Os resultados do cadastro correspondem ao código atual. O script de importação foi adicionado para viabilizar a segunda medição, não para simular ou fabricar resultados.
