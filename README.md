# APS 3 — Dossiê de Arquitetura Viva e Modelagem Comportamental as-Code

| | |
|---|---|
| **Aluno** | Nome Sobrenome *(substituir)* |
| **Unidade Curricular** | Modelagem de Software — Engenharia de Software, 4º período (UniSenai) |
| **Projeto** | AutoFlow Monitor — Monitoramento IoT na Usinagem de Componentes Automotivos |
| **Funcionalidade crítica** | **Cadastro de ativos e sensores** (RF02 / UC04) |
| **Ator principal** | Gestor |

## 1. A funcionalidade

O Gestor cadastra um **ativo monitorado** (CNC ou bancada, com responsável e criticidade), seus **sensores** e os **pontos de medição** que ligam cada sensor ao ativo, com posição, grandeza, unidade e validade da verificação. Sem esse cadastro, a telemetria não tem a quem ser associada, e por isso a funcionalidade é crítica para todo o restante do monitoramento (RF04, RN01, RN02).

**Pré-condição:** Gestor autenticado com permissão `ATIVO_CADASTRAR` (RF01, RNF02).

**Regras aplicadas**

- **RF02:** não pode existir ponto de medição sem vínculo com ativo e unidade.
- **Estados independentes** (escopo 2.1): o ativo nasce com estado operacional `PARADO` e condição `INDETERMINADA`; o estado do sensor é separado de ambos.
- **Remontagem exige nova checagem:** o sensor volta a `PendenteVerificacao`.
- **RNF03:** toda criação gera trilha de auditoria, sem sobrescrever registros.
- **RNF05:** os consumidores dos eventos são idempotentes.

**Pós-condição (sucesso):** ativo, sensores e pontos persistidos em uma única transação. Os sensores ficam em `EM_MONITORAMENTO`, no subestado `SemComunicacao` até a primeira leitura válida. `AtivoCadastradoEvent` é publicado e consumido pela Auditoria e pelo Monitoramento.

## 2. Contrato de interfaces e DTOs

Regra de consistência do Artefato 1: toda mensagem enviada a uma linha de vida deve existir como operação pública na classe ou interface correspondente. O diagrama a seguir é esse contrato.

```mermaid
classDiagram
    class CadastroPage { +submeterCadastro(form FormularioCadastroDTO) void }
    class AtivoController { +cadastrarAtivo(req CadastroAtivoRequestDTO) RespostaHttp }
    class AuthService { +autorizar(token String, permissao String) AutorizacaoDTO }
    class CadastroAtivoService { +registrar(cmd CadastroAtivoCommand) AtivoCadastradoDTO }
    class ResponsavelRepository { +buscarPorId(id UUID) Responsavel }
    class ValidadorPontoMedicao { +validar(dto PontoMedicaoDTO) ResultadoValidacaoDTO }
    class AtivoRepository {
        +existePorTag(tag String) boolean
        +salvar(ativo Ativo) Ativo
    }
    class SensorRepository {
        +existePorNumeroSerie(numeroSerie String) boolean
        +salvar(sensor Sensor) Sensor
        +transicionarEstado(sensorId UUID, de EstadoSensor, para EstadoSensor) boolean
    }
    class PontoMedicaoRepository { +salvar(ponto PontoMedicao) PontoMedicao }
    class EventPublisher { +publicar(evento AtivoCadastradoEvent) void }
    class AuditoriaService { +registrar(dto RegistroAuditoriaDTO) void }
    class ConfiguracaoMonitoramentoService { +atualizarConfiguracao(dto ConfiguracaoAtivoDTO) void }
    CadastroPage ..> AtivoController
    AtivoController ..> AuthService
    AtivoController ..> CadastroAtivoService
    CadastroAtivoService ..> ResponsavelRepository
    CadastroAtivoService ..> ValidadorPontoMedicao
    CadastroAtivoService ..> AtivoRepository
    CadastroAtivoService ..> SensorRepository
    CadastroAtivoService ..> PontoMedicaoRepository
    CadastroAtivoService ..> EventPublisher
    EventPublisher ..> AuditoriaService : evento assincrono
    EventPublisher ..> ConfiguracaoMonitoramentoService : evento assincrono

```

| DTO | Campos principais |
|---|---|
| `CadastroAtivoRequestDTO` | `tag`, `nome`, `tipo` (CNC \| BANCADA), `criticidade`, `responsavelId`, `sensores: SensorDTO[]` |
| `SensorDTO` | `numeroSerie`, `modelo`, `pontosMedicao: PontoMedicaoDTO[]` |
| `PontoMedicaoDTO` | `posicao`, `grandeza` (TEMPERATURA \| VIBRACAO_RMS), `unidade`, `verificacaoValidaAte` |
| `AutorizacaoDTO` | `permitido`, `perfil`, `usuarioId` |
| `ResultadoValidacaoDTO` | `valido`, `erros: String[]` |
| `AtivoCadastradoDTO` | `ativoId`, `estadoOperacional` (PARADO), `condicao` (INDETERMINADA), `sensores: [{sensorId, estado, pontoIds}]`, `criadoEm` |
| `ErroDTO` | `codigo`, `mensagem`, `detalhes[]` |
| `AtivoCadastradoEvent` | `eventId`, `ativoId`, `usuarioId`, `ocorridoEm`, `sensores[]` |

## 3. Artefato 1 — Diagrama de Sequência

`->>` indica chamada síncrona bloqueante e `-)` indica mensagem assíncrona. O diagrama usa os fragmentos `alt/else` (autorização, vínculos, validação), `par` (validação concorrente e entrega aos consumidores), `loop` (sensores e pontos) e `break` (rollback por conflito).

```mermaid
%% Artefato 1 - Diagrama de Sequencia - Cadastro de Ativos e Sensores (RF02 / UC04)
sequenceDiagram
    autonumber
    actor G as Gestor
    participant UI as CadastroPage
    participant API as AtivoController
    participant AUTH as AuthService
    participant SVC as CadastroAtivoService
    participant RESP as ResponsavelRepository
    participant VAL as ValidadorPontoMedicao
    participant ATV as AtivoRepository
    participant SEN as SensorRepository
    participant PM as PontoMedicaoRepository
    participant BUS as EventPublisher
    participant AUD as AuditoriaService
    participant MON as ConfiguracaoMonitoramentoService

    G->>UI: submeterCadastro(FormularioCadastroDTO)
    UI->>API: cadastrarAtivo(CadastroAtivoRequestDTO)
    API->>AUTH: autorizar(token, "ATIVO_CADASTRAR")
    AUTH-->>API: AutorizacaoDTO

    alt autorizacao.permitido == false
        API-->>UI: 403 ErroDTO(ACESSO_NEGADO)
        UI-->>G: exibe acesso negado
    else autorizacao.permitido == true
        API->>SVC: registrar(CadastroAtivoCommand)
        SVC->>RESP: buscarPorId(responsavelId)
        RESP-->>SVC: Responsavel
        SVC->>ATV: existePorTag(tag)
        ATV-->>SVC: boolean

        alt responsavel inexistente ou tag ja cadastrada
            SVC-->>API: ErroDominio(RESPONSAVEL_INVALIDO ou TAG_DUPLICADA)
            API-->>UI: 409 ErroDTO
            UI-->>G: exibe conflito de cadastro
        else vinculos validos
            par validacao dos sensores
                loop para cada sensor da requisicao
                    SVC->>SEN: existePorNumeroSerie(numeroSerie)
                    SEN-->>SVC: boolean
                end
            and validacao dos pontos de medicao
                loop para cada ponto de medicao de cada sensor
                    SVC->>VAL: validar(PontoMedicaoDTO)
                    VAL-->>SVC: ResultadoValidacaoDTO
                end
            end

            alt erros.isEmpty() == false
                SVC-->>API: ErroDominio(VALIDACAO_FALHOU, ResultadoValidacaoDTO)
                API-->>UI: 422 ErroDTO
                UI-->>G: exibe erros de validacao
            else erros.isEmpty() == true
                Note over SVC,PM: BEGIN transacao (ativo, sensores e pontos sao atomicos)
                SVC->>ATV: salvar(Ativo)
                ATV-->>SVC: Ativo
                loop para cada sensor da requisicao
                    SVC->>SEN: salvar(Sensor)
                    SEN-->>SVC: Sensor
                    loop para cada ponto de medicao do sensor
                        SVC->>PM: salvar(PontoMedicao)
                        PM-->>SVC: PontoMedicao
                    end
                    SVC->>SEN: transicionarEstado(sensorId, PENDENTE_VERIFICACAO, EM_MONITORAMENTO)
                    SEN-->>SVC: boolean
                    break transicao recusada (boolean == false) ou violacao de UNIQUE
                        SVC-->>API: ErroDominio(ESTADO_INVALIDO) com ROLLBACK
                        API-->>UI: 409 ErroDTO
                        UI-->>G: exibe conflito de concorrencia
                    end
                end
                Note over SVC,PM: COMMIT transacao
                SVC-)BUS: publicar(AtivoCadastradoEvent)
                SVC-->>API: AtivoCadastradoDTO
                API-->>UI: 201 AtivoCadastradoDTO
                UI-->>G: exibe confirmacao do cadastro

                par entrega assincrona aos consumidores
                    BUS-)AUD: registrar(RegistroAuditoriaDTO)
                and
                    BUS-)MON: atualizarConfiguracao(ConfiguracaoAtivoDTO)
                end
            end
        end
    end

```

## 4. Artefato 2 — Diagrama de Atividades com Raias

Quatro raias (Gestor, Web Dashboard, API de Cadastro e Serviços Assíncronos). Há quatro decisões com guardas mutuamente exclusivas e dois pares Fork/Join: validação de sensores e pontos em paralelo, e resposta ao Gestor em paralelo com Auditoria e Monitoramento. O Mermaid não tem nó de barra nativo, então as barras são nós escuros rotulados FORK e JOIN.

```mermaid
%% Artefato 2 - Diagrama de Atividades com Raias - Cadastro de Ativos e Sensores
%% Barras FORK/JOIN representadas por nos escuros (Mermaid nao possui no nativo de barra)
flowchart TB
    classDef barra fill:#111,stroke:#111,color:#fff,font-weight:bold
    classDef inicio fill:#111,stroke:#111,color:#fff
    classDef fim fill:#fff,stroke:#111,stroke-width:4px,color:#111

    subgraph L1["Raia 1 - Gestor"]
        INI(("Inicio")):::inicio
        A1["Preencher dados do ativo, sensores e pontos de medicao"]
    end

    subgraph L2["Raia 2 - Web Dashboard (React 18)"]
        B1["Validar campos obrigatorios do formulario"]
        D1{"Formulario completo?"}
        B2["Destacar campos pendentes"]
        B3["Enviar POST /ativos (HTTPS)"]
        B4["Exibir erro de negocio ou validacao"]
        B6["Exibir acesso negado"]
        B5["Exibir confirmacao 201"]
    end

    subgraph L3["Raia 3 - API de Cadastro (Node.js, back-end)"]
        C1["Autenticar e autorizar perfil"]
        D2{"Perfil autorizado?"}
        C2["Retornar 403 Acesso Negado"]
        C3["Verificar responsavel e unicidade da tag"]
        D3{"Vinculos validos?"}
        F1["=== FORK validacoes ==="]:::barra
        C5["Validar numero de serie unico dos sensores"]
        C6["Validar pontos: ativo, unidade, grandeza e validade da verificacao"]
        J1["=== JOIN validacoes ==="]:::barra
        D4{"Validacao sem erros?"}
        C7["Retornar 422 com lista de erros"]
        C8["Persistir Ativo PARADO e INDETERMINADA, Sensores e Pontos em transacao"]
        C9["Transicionar sensores PENDENTE_VERIFICACAO para EM_MONITORAMENTO"]
        D5{"Transicao aceita?"}
        C4["Reverter transacao e retornar 409 Conflito"]
        C10["Publicar AtivoCadastradoEvent no RabbitMQ"]
        F2["=== FORK pos-cadastro ==="]:::barra
        C11["Responder 201 AtivoCadastradoDTO"]
        J2["=== JOIN pos-cadastro ==="]:::barra
        FIM1(("Fim negado")):::fim
        FIM2(("Fim cadastro concluido")):::fim
    end

    subgraph L4["Raia 4 - Servicos Assincronos (consumidores RabbitMQ)"]
        E1["Auditoria: registrar RegistroAuditoriaDTO"]
        E2["Monitoramento: atualizar configuracao de ativos e pontos"]
    end

    INI --> A1 --> B1 --> D1
    D1 -->|"[campos incompletos]"| B2 --> A1
    D1 -->|"[campos completos]"| B3 --> C1 --> D2
    D2 -->|"[perfil != GESTOR]"| C2 --> B6 --> FIM1
    D2 -->|"[perfil == GESTOR]"| C3 --> D3
    D3 -->|"[responsavel inexistente ou tag duplicada]"| C4
    D3 -->|"[responsavel existente e tag inedita]"| F1
    F1 --> C5 --> J1
    F1 --> C6 --> J1
    J1 --> D4
    D4 -->|"[erros.notEmpty()]"| C7 --> B4
    D4 -->|"[erros.isEmpty()]"| C8 --> C9 --> D5
    D5 -->|"[transicao recusada]"| C4
    C4 --> B4
    B4 --> A1
    D5 -->|"[transicao aceita]"| C10 --> F2
    F2 --> C11 --> B5 --> J2
    F2 --> E1 --> J2
    F2 --> E2 --> J2
    J2 --> FIM2

```

## 5. Artefato 3 — Máquina de Estados (entidade `Sensor`)

O ciclo de vida cadastral (`PendenteVerificacao`, `EmMonitoramento`, `Desativado`) engloba os três estados de saúde do sensor definidos no escopo (`Disponivel`, `Suspeito`, `SemComunicacao`) como subestados. `Desativado` é o estado terminal: não há reativação, e um novo cadastro preserva o histórico anterior.

**Proteção contra estados inválidos:** eventos não mapeados no estado atual são rejeitados e não alteram nada. Por exemplo, `leituraValidaRecebida` em `PendenteVerificacao` é rejeitada. No banco, a transição é um *compare-and-set* (`UPDATE ... WHERE estado = :de`), o que impede corrida entre requisições concorrentes.

```mermaid
%% Artefato 3 - Maquina de Estados da entidade Sensor (ciclo de vida cadastral + estados de saude)
stateDiagram-v2
    [*] --> PendenteVerificacao : sensorCadastrado [perfil == GESTOR and numeroSerieUnico == true] / registrarAuditoria(SENSOR_CADASTRADO)

    PendenteVerificacao : PendenteVerificacao
    PendenteVerificacao : entry / registrarInicioDaVerificacao()
    PendenteVerificacao : do / aguardarVinculoDoPontoDeMedicao()
    PendenteVerificacao : exit / registrarFimDaVerificacao()

    PendenteVerificacao --> EmMonitoramento : pontoMedicaoVinculado [verificacaoValida == true and unidadeInformada == true] / publicarEvento(SensorHabilitado)
    PendenteVerificacao --> Desativado : cadastroCancelado [perfil == GESTOR] / registrarAuditoria(CADASTRO_CANCELADO)

    state EmMonitoramento {
        [*] --> SemComunicacao
        SemComunicacao --> Disponivel : leituraValidaRecebida [qualidade == VALIDA] / atualizarUltimaLeitura()
        Disponivel --> SemComunicacao : timeoutLeitura [segundosSemLeitura ≥ 30] / sinalizarIndisponibilidade()
        Disponivel --> Suspeito : leituraSuspeita [faixaFisicaViolada == true or valorRepetido5min == true] / registrarVerificacao()
        Suspeito --> Disponivel : investigacaoConcluida [conclusao == DESCARTADA] / limparSuspeita()
        Suspeito --> SemComunicacao : timeoutLeitura [segundosSemLeitura ≥ 30] / sinalizarIndisponibilidade()
    }

    SemComunicacao : SemComunicacao
    SemComunicacao : entry / iniciarTemporizador(30s)
    Disponivel : Disponivel
    Disponivel : do / monitorarJanelaDeLeituras()
    Suspeito : Suspeito
    Suspeito : entry / abrirVerificacaoDeLeitura()

    EmMonitoramento --> PendenteVerificacao : remontagemRegistrada [perfil == GESTOR] / invalidarVerificacao()
    EmMonitoramento --> Desativado : desativacaoSolicitada [perfil == GESTOR] / registrarAuditoria(SENSOR_DESATIVADO)

    Desativado : Desativado
    Desativado : entry / bloquearNovasLeituras()
    Desativado --> [*]

```

## 6. Artefato 4 — C4 Nível 2 (Contêineres)

Escrito em Mermaid C4. As tecnologias e os protocolos estão explícitos nos conectores.

```mermaid
%% Artefato 4 - C4 Nivel 2 (Conteineres) - Cadastro de Ativos e Sensores
C4Container
    title Diagrama de Conteineres (C4 Nivel 2) - Cadastro de Ativos e Sensores - AutoFlow Monitor

    Person(gestor, "Gestor", "Cadastra ativos, sensores, pontos de medicao e responsaveis")
    Person(consulta, "Operador, Tecnico e Analista", "Consultam ativos e saude dos sensores no dashboard")
    System_Ext(aquisicao, "Infraestrutura de Aquisicao Industrial", "Gateway que coleta CNC e sensores e envia telemetria")

    System_Boundary(autoflow, "AutoFlow Monitor") {
        Container(web, "Web Dashboard", "React 18, TypeScript", "SPA com formularios de cadastro e dashboard")
        Container(api, "API de Cadastro", "Node.js 20, NestJS", "Autentica via JWT, autoriza por perfil, valida e persiste o cadastro. Publica eventos de dominio")
        ContainerQueue(broker, "Broker de Mensagens", "RabbitMQ 3.13", "Exchange topic cadastro.eventos com filas duraveis por consumidor")
        Container(aud, "Servico de Auditoria", "Node.js 20", "Consome eventos e grava trilha de auditoria sem sobrescrever registros")
        Container(mon, "Servico de Monitoramento", "Node.js 20", "Mantem configuracao de ativos e pontos e processa telemetria")
        ContainerDb(db, "Banco de Dados", "PostgreSQL 16", "Schemas cadastro, auditoria e monitoramento")
    }

    Rel(gestor, web, "Preenche cadastro de ativos e sensores", "HTTPS")
    Rel(consulta, web, "Consulta ativos e sensores", "HTTPS")
    Rel(web, api, "Chama operacoes de cadastro e consulta", "REST/JSON sobre HTTPS TLS 1.3")
    Rel(api, db, "Le e grava cadastro em transacao", "SQL sobre TLS, porta 5432")
    Rel(api, broker, "Publica AtivoCadastradoEvent", "AMQP 0-9-1 sobre TLS")
    Rel(broker, aud, "Entrega eventos de cadastro", "AMQP 0-9-1")
    Rel(broker, mon, "Entrega eventos de cadastro", "AMQP 0-9-1")
    Rel(aud, db, "Grava trilha de auditoria", "SQL sobre TLS")
    Rel(mon, db, "Atualiza configuracao de monitoramento", "SQL sobre TLS")
    Rel(aquisicao, mon, "Envia telemetria dos pontos de medicao", "MQTT 3.1.1 sobre TLS")

```

## 7. Artefato 5 — ADR 0001

[`docs/architecture/adr-0001.md`](docs/architecture/adr-0001.md): propagação assíncrona do cadastro por eventos de domínio (RabbitMQ). A decisão é *Aprovada*, com alternativas, consequências e riscos aceitos documentados.

## 8. Matriz de consistência entre os modelos

| Passo na Sequência | Nó nas Atividades | Transição na Máquina de Estados | Regra |
|---|---|---|---|
| `autorizar(token, "ATIVO_CADASTRAR")` | C1, D2 | guarda `perfil == GESTOR` | RF01, RNF02 |
| `buscarPorId`, `existePorTag` | C3, D3 | — | RF02 |
| `existePorNumeroSerie` | C5 (ramo do Fork 1) | guarda `numeroSerieUnico == true` | identificação única |
| `validar(PontoMedicaoDTO)` | C6 (ramo do Fork 1) | guarda `verificacaoValida and unidadeInformada` | RF02 |
| `salvar(Ativo/Sensor/PontoMedicao)` | C8 | `[*] → PendenteVerificacao` | RNF03 |
| `transicionarEstado(PENDENTE → EM_MONITORAMENTO)` | C9, D5 | `pontoMedicaoVinculado` | proteção do estado |
| `break` com rollback | C4 (409) | transição recusada, sem mudança de estado | consistência |
| `publicar(AtivoCadastradoEvent)` | C10, Fork 2 | ação `publicarEvento(SensorHabilitado)` | ADR 0001 |
| `registrar(RegistroAuditoriaDTO)` | E1 | ação `registrarAuditoria()` | RNF03 |
| `atualizarConfiguracao(...)` | E2 | — | RN01, RN02 |

## 9. Como renderizar e validar

- Os blocos `mermaid` acima renderizam direto no GitHub.
- Para validar localmente: `npx @mermaid-js/mermaid-cli -i docs/diagrams/sequencia.mmd -o sequencia.svg` (repita para cada `.mmd`).
- Os cinco arquivos-fonte ficam em `docs/`. Faça commits incrementais por artefato para evidenciar a evolução (política de auditoria da APS).
