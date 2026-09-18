# Knight's Quest — Checklist de Melhorias

Diagnóstico feito a partir da leitura integral do código (37 `.cpp`, 40 `.hpp`, ~9.300 linhas), do `CMakeLists.txt`, do estado do repositório e do `Relatório Jogo.pdf`.

**Ordem do documento:** da correção mais urgente (bug real, comportamento indefinido) para a mais secundária (adição de escopo, item acadêmico). As últimas seções são **acréscimos desejáveis**, não dívidas do código.

**Legenda de severidade**

| | Significado |
|---|---|
| 🔴 | Comportamento indefinido / corrupção de memória — corrigir antes de qualquer outra coisa |
| 🟠 | Bug de comportamento ou perda de dados do jogador |
| 🟡 | Fragilidade, inconsistência de padrão, risco latente |
| 🧹 | Código morto — remoção pura, sem mudança de comportamento |
| 🏗️ | Dívida estrutural — muda arquitetura, exige planejamento |
| 🔧 | Infraestrutura de projeto (build, testes, CI, repositório) |
| ⚡ | Performance |
| 🎮 | Adição de produto — o jogo funciona sem isso |
| 🎓 | Item do checklist do relatório da disciplina |

---

## Fase 0 — Pré-requisito (fazer primeiro, 10 minutos)

Sem isto, boa parte da Fase 1 é invisível. O compilador acusa sozinho vários dos bugs abaixo.

- [ ] Ligar warnings no [CMakeLists.txt](game/CMakeLists.txt): `-Wall -Wextra -Wpedantic` (e o equivalente MSVC `/W4`).
- [ ] Rodar um build limpo e **salvar a lista de warnings** — ela é a extensão desta Fase 1. Esperado aparecer, no mínimo: membros não inicializados (item 1.2), `-Wreorder` (item 1.6) e variável sombreada (item 4.7).
- [ ] Rodar uma vez com sanitizers (`-fsanitize=address,undefined`) e jogar uma partida completa, incluindo **fechar pelo X da janela** — é o caminho que dispara o item 1.1.
- [ ] Só depois considerar `-Werror` (com os warnings já zerados).

**Critério de pronto:** build sem warnings novos e partida completa sob ASan/UBSan sem relatório.

---

## Fase 1 — 🔴 Comportamento indefinido e corrupção de memória

Estes não são "melhorias": são defeitos que podem travar o jogo ou produzir comportamento aleatório.

- [ ] **1.1 — `Tela::~Tela()` faz `delete` em objetos que nunca foram `new`**
  [Tela.cpp:13-16](game/JOGO_2D/Sources/Tela.cpp#L13-L16) — `for (sf::Text& texto : textos) { delete& texto; }`, sendo `textos` um `std::vector<sf::Text>` (objetos **por valor**). Entrega ao alocador um endereço no meio do buffer do vector → corrupção de heap.
  **Correção:** apagar o laço inteiro (o vector destrói os elementos sozinho).
  ⚠️ **Atenção à ordem:** hoje o bug está mascarado porque `Sair` chama `exit(1)` ([Menu.cpp:101](game/JOGO_2D/Sources/Menu.cpp#L101)), que pula destrutores. Fechar pelo X já executa o destrutor. **Corrigir este item ANTES do item 3.1** — remover o `exit(1)` sem consertar isto transforma um bug mascarado em crash na saída.

- [ ] **1.2 — Membros lidos sem inicialização (leitura de valor indeterminado)**
  - [ ] `Jogador`: a lista de inicialização ([Jogador.cpp:22-52](game/JOGO_2D/Sources/Jogador.cpp#L22-L52)) omite `envenenado`, `lento`, `tomandoDano`, `forcaVeneno`, `forcaLentidao`, `forcaPulo`. E `atualizar()` lê `if (envenenado)` ([:76](game/JOGO_2D/Sources/Jogador.cpp#L76)) e `if (lento)` ([:97](game/JOGO_2D/Sources/Jogador.cpp#L97)).
    *Sintoma provável (a confirmar):* jogador nasce envenenado/lento e perde vida sozinho, de forma intermitente.
  - [ ] `Tela`: o construtor ([Tela.cpp:3-9](game/JOGO_2D/Sources/Tela.cpp#L3-L9)) não inicializa `entradaAtiva`, `entradaAtiva2`, `caixa1selecionada`, `caixa2selecionada`. `desenharTela()` lê `entradaAtiva` ([:33](game/JOGO_2D/Sources/Tela.cpp#L33)), e só `telaGameOver` recebe `setEntradaAtiva` — `telaInicial`, `tela1`, `tela3`, `telaPausa` e `telaMundos` **nunca**.
    *Sintoma provável:* caixa de texto fantasma em telas sem campo de entrada.
  - [ ] `Menu::n_jogadores` ([Menu.hpp:25](game/JOGO_2D/Headers/Menu.hpp#L25)) sem inicialização.
  **Correção estrutural:** adotar inicialização na declaração (`bool envenenado = false;`) em todos os headers de entidade, em vez de depender da lista de inicialização de cada construtor.

- [ ] **1.3 — Chefão vaza memória e cresce sem limite**
  [Chefao.cpp](game/JOGO_2D/Sources/Chefao.cpp): `new Projetil` ([:70](game/JOGO_2D/Sources/Chefao.cpp#L70)) e `new Portal` ([:156](game/JOGO_2D/Sources/Chefao.cpp#L156), [:382](game/JOGO_2D/Sources/Chefao.cpp#L382), [:389](game/JOGO_2D/Sources/Chefao.cpp#L389)). Ao terminar, o slot recebe `nullptr` ([:202](game/JOGO_2D/Sources/Chefao.cpp#L202), [:278](game/JOGO_2D/Sources/Chefao.cpp#L278)) mas **nunca há `delete` nem `erase`**; o destrutor ([:48-50](game/JOGO_2D/Sources/Chefao.cpp#L48-L50)) está vazio.
  **Efeito:** cada projétil/portal vaza, e os vetores crescem para sempre sendo varridos todo quadro.
  **Correção:** `std::vector<std::unique_ptr<...>>` + `erase` dos concluídos. (A solução definitiva é o pool do item 5.5.)

- [ ] **1.4 — Ponteiros crus com posse implícita em `Tela`**
  `botoes` é `std::vector<sf::RectangleShape*>` alimentado por `new` em [Menu.cpp:432-565](game/JOGO_2D/Sources/Menu.cpp#L432-L565) e [Principal.cpp:158-256](game/JOGO_2D/Sources/Principal.cpp#L158-L256), e destruído dentro de `~Tela`. Funciona, mas a posse não está expressa em nenhum tipo.
  **Correção:** `std::vector<std::unique_ptr<sf::RectangleShape>>`, ou melhor: `Tela` cria o botão a partir de parâmetros em vez de receber ponteiro pronto.

- [ ] **1.5 — `Tela::verificaEventoTela` assume `botoes.size() == textos.size()`**
  [Tela.cpp:60-66](game/JOGO_2D/Sources/Tela.cpp#L60-L66) itera `botoes` e indexa `textos[i]`. Hoje todas as telas casam, mas qualquer botão adicionado sem texto (ou vice-versa) causa acesso fora dos limites.
  **Correção:** um único `struct ItemMenu { RectangleShape area; Text rotulo; }` — elimina a possibilidade do desalinhamento.

- [ ] **1.6 — Listas de inicialização fora da ordem de declaração**
  [Inimigo.cpp:9-27](game/JOGO_2D/Sources/Inimigo.cpp#L9-L27) inicializa `iteracoes(0)` **antes** da base `Personagem()`; `Jogador` põe `regiaoAtaque()` antes de `atacando`. Não é UB, mas é fonte de bug quando um membro passa a depender de outro. `-Wreorder` acusa.

**Critério de pronto:** ASan/UBSan silenciosos numa partida completa (menu → fase → pausa → salvar → carregar → morrer → ranking → fechar pelo X).

---

## Fase 2 — 🟠 Bugs de comportamento e perda de dados

- [ ] **2.1 — Rebuild apaga o ranking do jogador**
  [CMakeLists.txt:36-39](game/CMakeLists.txt#L36-L39) copia `JOGO_2D/Saves/` sobre o diretório de build **após cada build**, sobrescrevendo o `ranking.txt` do jogador pela versão do repositório.
  **Correção:** dado de jogador não mora no build tree nem no versionamento. Gravar em diretório de usuário (ou em `saves/` fora do build) e remover a cópia.

- [ ] **2.2 — Carregar save pela pausa apaga o jogador 2**
  [EstadoPausa.cpp:168](game/JOGO_2D/Sources/EstadoPausa.cpp#L168): `fase->instanciaProcedural(m.getFaseAtual(), 1)` com **1 hardcoded**. Em partida de 2 jogadores, carregar pela pausa remove o P2 em silêncio.
  **Correção:** usar `numJogadores` do save (`DadosSave::numJogadores` já existe e é gravado).

- [ ] **2.3 — `aplicarSave` chamado duas vezes no fluxo de carregar**
  [EstadoPausa.cpp:164](game/JOGO_2D/Sources/EstadoPausa.cpp#L164) aplica na fase **antiga** (que será destruída na linha seguinte) e [:169](game/JOGO_2D/Sources/EstadoPausa.cpp#L169) na nova. A primeira chamada é inútil e confunde a leitura do fluxo.

- [ ] **2.4 — Obstáculos perigosos atualizam 2× por passo**
  [Gerenciador_Colisoes.cpp:41](game/JOGO_2D/Sources/Gerenciador_Colisoes.cpp#L41) chama `obstaculos.at(i)->atualizar()` **e** [Fase.cpp:294](game/JOGO_2D/Sources/Fase.cpp#L294) chama `o->atualizar()` para a lista inteira — ambos no mesmo passo ([EstadoJogo.cpp:109-110](game/JOGO_2D/Sources/EstadoJogo.cpp#L109-L110)). Serra/Espinho/Slime animam no dobro da velocidade projetada.
  **Correção:** atualizar em um único lugar (a `Fase` é a dona; o gerenciador de colisões só deve detectar e resolver).
  ⚠️ **Decisão de design necessária:** corrigir isso **muda o game feel**. Ou se ajustam as velocidades de animação junto, ou se mantém o comportamento atual de propósito — o que precisa estar escrito no código, não implícito na chamada duplicada.

- [ ] **2.5 — Cooldowns do chefão usam relógio de parede numa simulação de passo fixo**
  [Chefao.cpp:91](game/JOGO_2D/Sources/Chefao.cpp#L91) e [:211-215](game/JOGO_2D/Sources/Chefao.cpp#L211-L215) usam `std::chrono::steady_clock` para decidir ataque/slam/spawn, enquanto a simulação roda em passo fixo de 1/60 s ([EstadoJogo.cpp:19](game/JOGO_2D/Sources/EstadoJogo.cpp#L19)).
  **Efeito:** durante a pausa os cooldowns continuam correndo; ao despausar o chefão dispara tudo acumulado de uma vez.
  **Correção:** contar **passos de simulação**, não milissegundos de parede. Remover o `#include <thread>` órfão de [Chefao.hpp:7](game/JOGO_2D/Headers/Chefao.hpp#L7).

- [ ] **2.6 — Parsing do ranking quebra com nome que contém hífen**
  [Menu.cpp:591](game/JOGO_2D/Sources/Menu.cpp#L591) divide a linha no **primeiro `-`**; o formato gravado é `pontos-nome` ([Mundo.cpp:195](game/JOGO_2D/Sources/Mundo.cpp#L195)).
  **Correção:** separador que não pode aparecer no nome (tab), ou validar o nome na entrada, ou gravar com tag como o `GerenciadorSave` já faz.

- [ ] **2.7 — Ranking sem limite de entradas transborda a tela**
  [Menu.cpp:187](game/JOGO_2D/Sources/Menu.cpp#L187) posiciona em `50 + 40*i`; a partir de ~21 registros o texto sai dos 900 px.
  **Correção:** exibir o Top N (com rolagem ou paginação) e truncar o arquivo.

- [ ] **2.8 — Pontuação zero é descartada do ranking**
  [Mundo.cpp:194-197](game/JOGO_2D/Sources/Mundo.cpp#L194-L197) só grava `if (pontuacao != 0)`. Quem fez 0 ponto some sem aviso, e nome vazio é aceito (`"30-"`).

- [ ] **2.9 — `Jogador::curar` não tem teto**
  [Jogador.cpp:633-639](game/JOGO_2D/Sources/Jogador.cpp#L633-L639) — com a habilidade Vampiro o HP cresce sem limite. A barra clampa visualmente ([Personagem.cpp:164-166](game/JOGO_2D/Sources/Personagem.cpp#L164-L166)), o que **esconde** o problema em vez de resolvê-lo.
  **Correção:** limitar em `vidaMaxima`.

- [ ] **2.10 — `Fase::setFase` engole falha de carregamento**
  [Fase.cpp:197-201](game/JOGO_2D/Sources/Fase.cpp#L197-L201) — `if (!texturaFundo.loadFromFile(...))` com corpo vazio e comentário explicando que é melhor manter a textura anterior. Asset ausente passa sem nenhum registro.
  **Correção:** usar `Gerenciador_Recursos` (que já lança exceção) — resolve junto o item 7.4.

**Critério de pronto:** roteiro manual verificado — salvar/carregar em 1P e 2P preservando número de jogadores; pausar durante luta de chefão e despausar sem rajada; nome com hífen sobrevive ao ranking; rebuild não apaga ranking.

---

## Fase 3 — 🟡 Tratamento de erro e ciclo de vida

- [ ] **3.1 — Substituir os 10 `exit(1)` por erro tratado**
  [Menu.cpp:19](game/JOGO_2D/Sources/Menu.cpp#L19), [:38](game/JOGO_2D/Sources/Menu.cpp#L38), [:101](game/JOGO_2D/Sources/Menu.cpp#L101), [:270](game/JOGO_2D/Sources/Menu.cpp#L270) · [Personagem.cpp:45](game/JOGO_2D/Sources/Personagem.cpp#L45), [:49](game/JOGO_2D/Sources/Personagem.cpp#L49) · [Fase.cpp:142](game/JOGO_2D/Sources/Fase.cpp#L142) · [Gerenciador_Grafico.cpp:124](game/JOGO_2D/Sources/Gerenciador_Grafico.cpp#L124) · [Floresta.hpp:18](game/JOGO_2D/Headers/Floresta.hpp#L18) · [Ruinas.hpp:18](game/JOGO_2D/Headers/Ruinas.hpp#L18)
  Mata o processo sem destrutores, sem fechar a janela, **passando por cima** do `try/catch` do [main.cpp:15](game/JOGO_2D/Sources/main.cpp#L15).
  Note a inconsistência interna: `Gerenciador_Recursos` faz certo (`throw std::runtime_error`), mas `Personagem` carrega textura na mão e aborta.
  **Correção:** `throw`; no caso do `exit(1)` do botão Sair ([Menu.cpp:101](game/JOGO_2D/Sources/Menu.cpp#L101)), sair do laço normalmente.
  ⚠️ **Depende do item 1.1** estar corrigido antes.

- [ ] **3.2 — Toda carga de recurso passa pelo `Gerenciador_Recursos`**
  Ainda há `loadFromFile` direto em [Personagem.cpp:44-49](game/JOGO_2D/Sources/Personagem.cpp#L44-L49) (barra de vida, relida **por personagem instanciado**), [Menu.cpp:17](game/JOGO_2D/Sources/Menu.cpp#L17) e [:36](game/JOGO_2D/Sources/Menu.cpp#L36), [Fase.cpp:197](game/JOGO_2D/Sources/Fase.cpp#L197), `Floresta.hpp`/`Ruinas.hpp`.
  **Ganho:** um só caminho de erro, uma só cópia de cada textura na memória.

- [ ] **3.3 — `Menu` carrega a fonte duas vezes**
  [Menu.cpp:12-17](game/JOGO_2D/Sources/Menu.cpp#L12-L17) faz `new sf::Font` + `loadFromFile("Menu/antiquity-print.ttf")`, e [Principal.cpp:15](game/JOGO_2D/Sources/Principal.cpp#L15) pega a **mesma** fonte pelo `Gerenciador_Recursos`. Duas cópias do mesmo arquivo em memória.

- [ ] **3.4 — Idioma inválido nos singletons**
  ```cpp
  pGerenciador == nullptr ? pGerenciador = new Gerenciador_Colisoes() : pGerenciador;
  ```
  [Gerenciador_Colisoes.cpp:23](game/JOGO_2D/Sources/Gerenciador_Colisoes.cpp#L23), [Gerenciador_Eventos.cpp:24](game/JOGO_2D/Sources/Gerenciador_Eventos.cpp#L24), [Gerenciador_Grafico.cpp:34](game/JOGO_2D/Sources/Gerenciador_Grafico.cpp#L34) — expressão ternária com valor descartado no lugar de um `if`.
  **Correção:** `static` local de função (thread-safe e destruído no fim do programa), resolvendo junto o item 3.5.

- [ ] **3.5 — Nenhum singleton é destruído**
  Todos criados com `new` e nunca liberados. `~Gerenciador_Grafico` — que deleta a `RenderWindow` ([Gerenciador_Grafico.cpp:22-30](game/JOGO_2D/Sources/Gerenciador_Grafico.cpp#L22-L30)) — é código morto: a janela nunca é fechada de forma ordenada.

**Critério de pronto:** nenhum `exit(` fora de `main`; nenhum `loadFromFile` fora do `Gerenciador_Recursos`; asset renomeado à força produz mensagem de erro clara em vez de fechar a janela do nada.

---

## Fase 4 — 🧹 Remoção de código morto

Nenhum destes muda comportamento. Fazer antes da Fase 5: refatorar código morto é trabalho jogado fora.

- [ ] **4.1 — API de persistência antiga: 22 funções que ninguém chama**
  `salvar(int)` / `limparArquivo(int)` são **virtuais puras** em [Entidade.hpp:30-31](game/JOGO_2D/Headers/Entidade.hpp#L30-L31) e obrigam implementação em 11 classes (`Jogador`, `Chefao`, `Cogumelo`, `OlhoVoador`, `Slime`, `Serra`, `Espinho`, `Plataforma`, `Parede`, `Portal`, `Projetil`). Verificado por busca: **nenhuma chamada existe** — o único `salvar()` chamado é o da `Configuracao`, que é outra classe.
  São ~200 linhas de esquema antigo (um arquivo por tipo de entidade), algumas com campos comentados ([Jogador.cpp:474-494](game/JOGO_2D/Sources/Jogador.cpp#L474-L494)).
  **Correção:** remover os dois métodos da interface e as 22 implementações. Quem persiste é o `GerenciadorSave`.

- [ ] **4.2 — `Floresta.hpp` e `Ruinas.hpp` nunca são incluídos**
  [Floresta.hpp](game/JOGO_2D/Headers/Floresta.hpp), [Ruinas.hpp](game/JOGO_2D/Headers/Ruinas.hpp) — viraram apenas temas visuais aplicados por `Fase::setFase`, como o próprio comentário de [Principal.hpp:22-24](game/JOGO_2D/Headers/Principal.hpp#L22-L24) registra. Apagar.

- [ ] **4.3 — Caminho do carregador de fase por arquivo está inalcançável**
  `Fase::instanciaEntidades` ([Fase.cpp:129](game/JOGO_2D/Sources/Fase.cpp#L129)), `CarregadorFase::carregar` ([CarregadorFase.cpp:113](game/JOGO_2D/Sources/CarregadorFase.cpp#L113)), `CarregadorFase::aleatorizar` ([:18](game/JOGO_2D/Sources/CarregadorFase.cpp#L18)) e os 4 arquivos `Fases/*.txt`: só `instanciaProcedural` é chamado.
  **Decisão:** apagar, ou reativar como "modo clássico" no menu. Manter inalcançável é o pior dos dois.

- [ ] **4.4 — Telas mortas no `Menu`**
  `tela2` é populada com textos e botões ([Menu.cpp:322-337](game/JOGO_2D/Sources/Menu.cpp#L322-L337), [:491-515](game/JOGO_2D/Sources/Menu.cpp#L491-L515)) e **nunca desenhada**. O `case 5` (`tela4`, "Fase 1 / Fase 2", [:195-239](game/JOGO_2D/Sources/Menu.cpp#L195-L239)) é inalcançável desde a virada para roguelike — e é justamente o caso que lê `n_jogadores` possivelmente não inicializado (item 1.2).

- [ ] **4.5 — `Principal::inicializaMundos` e `Tela telaMundos`**
  [Principal.cpp:190-258](game/JOGO_2D/Sources/Principal.cpp#L190-L258) monta a tela de slots, mas a seleção de slot hoje é feita por `Menu::escolherSlotSave` e pelo `EstadoPausa`. Verificar se `telaMundos` ainda é exibida em algum caminho; se não, remover.

- [ ] **4.6 — `Personagem::getMoveu`**
  [Personagem.cpp:131-143](game/JOGO_2D/Sources/Personagem.cpp#L131-L143) — declarado, definido, zero chamadas. Junto dele, avaliar `moveu` e `posAnterior`.

- [ ] **4.7 — Sobras dentro de `Inimigo::atualizarAnimacao`**
  [Inimigo.cpp:119-120](game/JOGO_2D/Sources/Inimigo.cpp#L119-L120) — `static sf::Clock clock; sf::Time elapsed = clock.getElapsedTime();` nunca usado. [:146](game/JOGO_2D/Sources/Inimigo.cpp#L146) — `int lado;` local sombreando o membro `lado`.

- [ ] **4.8 — Sobras de Visual Studio no repositório**
  `game/JOGO_2D/JOGO_2D.APS`, `game/JOGO_2D/JOGO_2D.rc`, `game/JOGO_2D/resource.h`, `game/JOGO_2D/x64/`. Se o ícone do executável no Windows importa, manter apenas o `.rc` + `.ico` e referenciá-los no CMake; o resto sai.

**Critério de pronto:** build continua idêntico, jogo continua idêntico, e o projeto tem algumas centenas de linhas menos.

---

## Fase 5 — 🏗️ Dívida estrutural

Aqui começa o trabalho que exige planejamento. Ordem importa: o 5.1 é pré-requisito de quase tudo.

- [ ] **5.1 — Separar `atualizar()` de `desenhar()`** *(o bloqueio central)*
  Hoje a lógica desenha: [Jogador.cpp:195](game/JOGO_2D/Sources/Jogador.cpp#L195), [Cogumelo.cpp:235](game/JOGO_2D/Sources/Cogumelo.cpp#L235), [Chefao.cpp:344](game/JOGO_2D/Sources/Chefao.cpp#L344), [Projetil.cpp:193](game/JOGO_2D/Sources/Projetil.cpp#L193), [Espinho.cpp:76](game/JOGO_2D/Sources/Espinho.cpp#L76).
  **Consequências já visíveis no código:**
  - `Estado::desenhar()` está **vazio nos dois estados** ([EstadoJogo.cpp:171-175](game/JOGO_2D/Sources/EstadoJogo.cpp#L171-L175), [EstadoPausa.cpp:237-240](game/JOGO_2D/Sources/EstadoPausa.cpp#L237-L240)) — a interface promete três responsabilidades e entrega duas.
  - O split-screen precisou de [`Fase::redesenharEntidades`](game/JOGO_2D/Sources/Fase.cpp#L170) como segunda passada de contorno.
  - Impossível interpolar entre passos, congelar o jogo atrás da pausa ou desacoplar taxa de render da taxa de simulação.
  **Correção:** `atualizar()` só muda estado; `desenhar()` só lê estado e emite draw calls. `EstadoJogo::desenhar` passa a ter conteúdo de verdade.

- [ ] **5.2 — Unificar as duas máquinas de estado**
  Existe `Gerenciador_Estados` (pilha de `Estado`, correto) **e** `Menu::executar` com `std::stack<int>` + `switch` de 6 casos com números mágicos ([Menu.cpp:62-277](game/JOGO_2D/Sources/Menu.cpp#L62-L277)).
  Pior: as telas de configuração **sequestram o laço principal** com `while` próprio — [:800](game/JOGO_2D/Sources/Menu.cpp#L800), [:888](game/JOGO_2D/Sources/Menu.cpp#L888), [:1012](game/JOGO_2D/Sources/Menu.cpp#L1012), [:1174](game/JOGO_2D/Sources/Menu.cpp#L1174) — exatamente o padrão que a refatoração de `Principal` eliminou do gameplay.
  **Correção:** `EstadoMenu`, `EstadoRanking`, `EstadoConfiguracoes`, `EstadoControles`, `EstadoHabilidades`, `EstadoSelecaoSlot`. `Menu.cpp` (1.250 linhas = 17% do projeto) deve cair para ~300.

- [ ] **5.3 — Um único pipeline de eventos**
  Hoje há **cinco** lugares drenando `pollEvent`: [Gerenciador_Eventos.cpp:45](game/JOGO_2D/Sources/Gerenciador_Eventos.cpp#L45) (que só trata `Closed`), [Tela.cpp:71](game/JOGO_2D/Sources/Tela.cpp#L71), as telas de config do `Menu`, [Menu.cpp:739](game/JOGO_2D/Sources/Menu.cpp#L739) e [EstadoPausa.cpp:131](game/JOGO_2D/Sources/EstadoPausa.cpp#L131). O gameplay, em paralelo, consulta `Keyboard::isKeyPressed` direto ([Jogador.cpp:404-411](game/JOGO_2D/Sources/Jogador.cpp#L404-L411)).
  **Correção:** o gerenciador drena a fila **uma vez** por quadro e entrega os eventos ao topo da pilha de estados. Isto é o requisito 8.2 do relatório, cumprido de verdade.

- [ ] **5.4 — Camada de input por ação**
  `Jogador` lê `Keyboard` diretamente, mesmo já existindo `Configuracao` com remapeamento. Um `InputMap` (ação → estado atual/borda de subida) desacopla o jogador do teclado, abre caminho para gamepad e centraliza a detecção de borda que hoje está espalhada em três flags (`ataquePressionadoAnterior`, `puloPressionadoAnterior`, `escPressionadoAnteriormente`).

- [ ] **5.5 — Pool de projéteis**
  Resolve o item 1.3 na raiz e elimina o `new`/`delete` por tiro. Padrão GOF adicional (conta para o item 9.6).

- [ ] **5.6 — `Inimigo::escolherAlvo()` — IA triplicada**
  O bloco "escolher o jogador mais próximo dentro do alcance, senão vagar" está copiado em três lugares, ~40 linhas cada, todos com `i < 2` hardcoded: [Cogumelo.cpp:172-223](game/JOGO_2D/Sources/Cogumelo.cpp#L172-L223), [OlhoVoador.cpp:146-180](game/JOGO_2D/Sources/OlhoVoador.cpp#L146-L180), [Chefao.cpp:298-338](game/JOGO_2D/Sources/Chefao.cpp#L298-L338) (+ o helper `jogadorMaisProximo` em [Chefao.cpp:53-66](game/JOGO_2D/Sources/Chefao.cpp#L53-L66), que é uma quarta variação do mesmo cálculo).

- [ ] **5.7 — `enum class EstadoAnimacao` no lugar dos números mágicos**
  `animacao == 2` significa "morrendo" em oito pontos: [Personagem.cpp:90](game/JOGO_2D/Sources/Personagem.cpp#L90), [:180](game/JOGO_2D/Sources/Personagem.cpp#L180), [Jogador.cpp:74](game/JOGO_2D/Sources/Jogador.cpp#L74), [:146](game/JOGO_2D/Sources/Jogador.cpp#L146), [:387](game/JOGO_2D/Sources/Jogador.cpp#L387), [Inimigo.cpp:134](game/JOGO_2D/Sources/Inimigo.cpp#L134), [:228](game/JOGO_2D/Sources/Inimigo.cpp#L228). Os índices 0-7 só existem em comentário ([Jogador.cpp:544-580](game/JOGO_2D/Sources/Jogador.cpp#L544-L580)) — e **cada classe usa uma ordem diferente** (`Jogador` tem 8 estados, `Cogumelo` 5, `Chefao` 5 com semântica distinta).

- [ ] **5.8 — Unificar a convenção de índice de jogador**
  `Mundo` é 0-based e generalizado para N slots ([Mundo.hpp:26](game/JOGO_2D/Headers/Mundo.hpp#L26)), mas:
  - `Gerenciador_Colisoes` e `Gerenciador_Eventos` têm `pJogador`/`pJogador2` fixos ([Gerenciador_Colisoes.hpp:18-19](game/JOGO_2D/Headers/Gerenciador_Colisoes.hpp#L18-L19), [Gerenciador_Eventos.hpp:13-14](game/JOGO_2D/Headers/Gerenciador_Eventos.hpp#L13-L14));
  - `danar(1)`/`danar(2)` é 1-based e cada obstáculo decodifica com `getJogador(jogador - 1)` ([Espinho.cpp:84](game/JOGO_2D/Sources/Espinho.cpp#L84), [Cogumelo.cpp:52](game/JOGO_2D/Sources/Cogumelo.cpp#L52));
  - vários laços usam `for (int i = 0; i < 2; ++i)` cravado ([Chefao.cpp:57](game/JOGO_2D/Sources/Chefao.cpp#L57), [:133](game/JOGO_2D/Sources/Chefao.cpp#L133), [Principal.cpp:58](game/JOGO_2D/Sources/Principal.cpp#L58)).
  A refatoração do `Mundo` parou no meio do caminho.

- [ ] **5.9 — Resolver a sobreposição dos três conceitos de vida**
  `vida` (atual), `vidaMaxima` (máximo real) e o virtual `getVida()` que retorna **constante de classe** — [Jogador.cpp:274-277](game/JOGO_2D/Sources/Jogador.cpp#L274-L277), [Chefao.cpp:429](game/JOGO_2D/Sources/Chefao.cpp#L429), [Cogumelo.cpp:155-161](game/JOGO_2D/Sources/Cogumelo.cpp#L155-L161). Um método chamado `getVida()` que não devolve a vida atual é armadilha para quem mexer depois.
  **Correção:** `getVidaAtual()` / `getVidaMaxima()` como membros, sem virtual.

- [ ] **5.10 — Higiene de header e macros**
  - [ ] `using namespace` em 10 headers, incluindo `using namespace std` em [Principal.hpp:11](game/JOGO_2D/Headers/Principal.hpp#L11) e [Gerenciador_Grafico.hpp:10](game/JOGO_2D/Headers/Gerenciador_Grafico.hpp#L10) — vaza para toda unidade de compilação que inclui.
  - [ ] `#define TELA_X/TELA_Y` ([Gerenciador_Grafico.hpp:6-7](game/JOGO_2D/Headers/Gerenciador_Grafico.hpp#L6-L7)) → `constexpr float`.
  - [ ] `#define MAX_DIST 1000.0f;` **com ponto-e-vírgula dentro da macro** ([Projetil.hpp:4](game/JOGO_2D/Headers/Projetil.hpp#L4)) — não é usado hoje; é uma mina para quem usar amanhã.
  - [ ] `#define VIDA_MAX` e `SIZE` repetidos com valores diferentes em [Chefao.cpp:10-11](game/JOGO_2D/Sources/Chefao.cpp#L10-L11) (600) e [Cogumelo.cpp:6-7](game/JOGO_2D/Sources/Cogumelo.cpp#L6-L7) (60) → `constexpr` em namespace anônimo, como `Jogador` e `Personagem` já fazem corretamente.

- [ ] **5.11 — Unificar o RNG**
  `std::rand()` ([CarregadorFase.cpp:26-38](game/JOGO_2D/Sources/CarregadorFase.cpp#L26-L38), [Camera.cpp:121](game/JOGO_2D/Sources/Camera.cpp#L121), [Chefao.cpp:380](game/JOGO_2D/Sources/Chefao.cpp#L380), [:387](game/JOGO_2D/Sources/Chefao.cpp#L387), [Cogumelo.cpp:34](game/JOGO_2D/Sources/Cogumelo.cpp#L34)) convive com `mt19937`. E `Inimigo` constrói um `random_device` + `mt19937` **por inimigo instanciado** ([Inimigo.cpp:33-34](game/JOGO_2D/Sources/Inimigo.cpp#L33-L34)) e outro a cada 600 updates ([:162-168](game/JOGO_2D/Sources/Inimigo.cpp#L162-L168)) — caro, e anula a reprodutibilidade que `gerarProcedural` tenta garantir com seed determinística ([CarregadorFase.cpp:207](game/JOGO_2D/Sources/CarregadorFase.cpp#L207)).
  **Correção:** um serviço de RNG único, semeável — pré-requisito para o save por seed do item 9.3.

- [ ] **5.12 — Remover o fator de escala mágico da animação**
  [Animacao.cpp:30-37](game/JOGO_2D/Sources/Animacao.cpp#L30-L37): `getAnimationSpeed()` multiplica por `ESCALA_60HZ = 0.25f` porque as velocidades foram calibradas numa máquina de ~240 fps. As velocidades deveriam estar em segundos, não em "iterações de uma máquina específica".

**Critério de pronto:** `EstadoJogo::desenhar()` com conteúdo real; `Menu.cpp` abaixo de 400 linhas; nenhum `pollEvent` fora do gerenciador de eventos; zero `using namespace` em header.

---

## Fase 6 — 🔧 Infraestrutura de projeto

- [ ] **6.1 — Limpar o repositório**
  - [ ] `game/build/` está **versionado**, incluindo o binário compilado de 1,1 MB (`game/build/JOGO_2D`) e uma **cópia duplicada de todos os assets**. Há dois commits `chore: Update binary build of JOGO_2D` no histórico.
  - [ ] `.DS_Store` versionado (e atualmente modificado).
  - [ ] `game/.gitignore` tem regras da estrutura antiga (`JOGO/.vs/JOGO/v17/.suo`) que não existe mais.
  **Correção:** `.gitignore` consolidado (`build/`, `.DS_Store`, `*.user`, `x64/`) + remover do índice o que já está rastreado.

- [ ] **6.2 — Arrumar o CMake**
  [CMakeLists.txt](game/CMakeLists.txt): comando de cópia de Assets **duplicado** ([:21-29](game/CMakeLists.txt#L21-L29)); `GLOB_RECURSE` (arquivo novo não entra no build sem re-rodar o cmake); sem build types; sem warnings; copia `Saves/` sobre o build (item 2.1).

- [ ] **6.3 — Testes automatizados**
  Zero testes hoje. Alvos de maior retorno, todos de lógica pura (não precisam de janela):
  - [ ] `GerenciadorSave`: round-trip, save corrompido, versão futura com tag desconhecida, slot inválido
  - [ ] `ArvoreHabilidades`: custo por nível, compra sem pontos, nível máximo
  - [ ] `Gerenciador_Colisoes::verificaColisao`: resolução por menor penetração, contato em quina
  - [ ] `CarregadorFase::gerarProcedural`: mesma fase ⇒ mesmo layout; `calcularLimites` coerente com a geração
  - [ ] `Mundo`: pontuação, kills, slots de jogador, `registrarKill` com e sem chefão

- [ ] **6.4 — CI**
  GitHub Actions: build em Linux/macOS/Windows + testes + uma execução com ASan/UBSan. Sem isso, os bugs da Fase 1 voltam.

- [ ] **6.5 — Documentação**
  - [ ] `README.md`: o que é, GIF/screenshots, build por plataforma, controles, diagrama de arquitetura em uma imagem
  - [ ] `CREDITS.md`: os créditos de asset hoje estão como comentário solto no fim do [main.cpp:24-33](game/JOGO_2D/Sources/main.cpp#L24-L33) — é o lugar errado para uma obrigação de licença
  - [ ] `LICENSE`
  - [ ] Atualizar (ou aposentar) o `Relatório Jogo.pdf`: ele descreve uma arquitetura que não existe mais

- [ ] **6.6 — Estilo consistente**
  `.clang-format` + `clang-tidy`. Hoje conviver mistura de tabs/espaços, nomes em português e inglês (`isJumping` ao lado de `atacando`, `jumpStrength` ao lado de `forcaPulo`) e estilos de chave diferentes por arquivo.

**Critério de pronto:** clone limpo compila em três plataformas via CI, com testes verdes, e `git ls-files` sem binário nem `build/`.

---

## Fase 7 — ⚡ Performance

Nada aqui é urgente hoje — o jogo roda. Vira problema conforme as fases crescem (até 7 seções, inimigos escalando por fase).

- [ ] **7.1 — I/O por quadro**
  Os 3 arquivos de save são lidos **a cada quadro** na seleção de slot ([Menu.cpp:694-704](game/JOGO_2D/Sources/Menu.cpp#L694-L704) — o comentário admite: *"barato e mantem a tela sempre atualizada"*) e em [EstadoPausa.cpp:43-53](game/JOGO_2D/Sources/EstadoPausa.cpp#L43-L53). O `ranking.txt` é relido e reordenado a cada quadro ([Menu.cpp:166](game/JOGO_2D/Sources/Menu.cpp#L166)).

- [ ] **7.2 — `sf::Text` construído por quadro, por entidade**
  [Inimigo::desenharNivel](game/JOGO_2D/Sources/Inimigo.cpp#L226) (um "Lv N" por inimigo, por quadro), [Jogador::desenharEfeitosAtivos](game/JOGO_2D/Sources/Jogador.cpp#L199), [Fase::desenharHUD](game/JOGO_2D/Sources/Fase.cpp#L357). Objetos de texto devem ser membros reaproveitados.

- [ ] **7.3 — Colisão sem partição espacial**
  [Gerenciador_Colisoes::Executar](game/JOGO_2D/Sources/Gerenciador_Colisoes.cpp#L28) é O(corpos × (jogadores + inimigos)) por passo, testando tudo contra tudo. Um grid uniforme resolve com folga na escala deste jogo.

- [ ] **7.4 — Textura de fundo relida a cada troca de fase**
  [Fase.cpp:197](game/JOGO_2D/Sources/Fase.cpp#L197) — `loadFromFile` a cada `setFase`, num jogo cujo loop principal é justamente trocar de fase. Resolve junto com o item 3.2.

---

## Fase 8 — 🎮 Adições de produto

Daqui para baixo **não é dívida técnica**: é escopo novo. O jogo está completo e jogável sem nada disto.

- [ ] **8.1 — Áudio (a ausência mais perceptível)**
  Verificado: **zero** referência a `sf::Sound`, `SoundBuffer` ou `Music` no projeto, e o CMake não linka `sfml-audio`. Um jogo sem som não passa por profissional, independentemente da qualidade do código.
  - [ ] Linkar `sfml-audio`; criar `Gerenciador_Audio` no mesmo padrão dos outros gerenciadores
  - [ ] SFX: golpe, acerto, dano, pulo, morte, portal, compra de habilidade, slam do chefão
  - [ ] Trilha por tema, com transição entre fases
  - [ ] Volume de música e de efeitos na tela de configurações (que já existe e já persiste)

- [ ] **8.2 — Meta-progressão não sobrevive ao fechar o jogo**
  A `ArvoreHabilidades` vive no `Mundo`, que vive em `Principal`, que morre com o processo. O comentário de [Mundo.hpp:105-107](game/JOGO_2D/Headers/Mundo.hpp#L105-L107) diz que os pontos "sobrevivem entre runs" — e sobrevivem, mas **só na mesma sessão**; fechar o jogo zera tudo que não foi gravado num slot.
  **Correção:** arquivo próprio de meta-progressão, no mesmo espírito do `config.txt`.

- [ ] **8.3 — Tela de fim de run com estatísticas**
  Hoje o Game Over só pede nome ([Menu.cpp:241-268](game/JOGO_2D/Sources/Menu.cpp#L241-L268)). Falta: fase alcançada, kills, dano infligido/recebido, tempo de run, habilidades compradas.

- [ ] **8.4 — Feedback de combate**
  A `Camera` já tem tremor e flash de dano ([Camera.cpp:46-64](game/JOGO_2D/Sources/Camera.cpp#L46-L64)) — falta o resto do kit: números de dano flutuantes, partículas de impacto, *hitstop* de alguns quadros no acerto.

- [ ] **8.5 — Balanceamento orientado a dados**
  Vida, dano, velocidade, alcance e custos de habilidade estão como `constexpr`/macro no código ([Jogador.cpp:16-19](game/JOGO_2D/Sources/Jogador.cpp#L16-L19), [Personagem.cpp:13-24](game/JOGO_2D/Sources/Personagem.cpp#L13-L24), [Chefao.cpp:10-11](game/JOGO_2D/Sources/Chefao.cpp#L10-L11), `ArvoreHabilidades`). Balancear exige recompilar — o que na prática significa que ninguém balanceia.

- [ ] **8.6 — Variedade de roguelike**
  Hoje são 3 arquétipos de inimigo + chefão, e as fases variam largura, quantidade e disposição. Candidatos: modificadores de run, sala de loja/evento entre fases, afixos em inimigos, mais arquétipos, chefões distintos (hoje é sempre o mesmo, a cada 5 fases — [Fase.cpp:389](game/JOGO_2D/Sources/Fase.cpp#L389)).

- [ ] **8.7 — Suporte a gamepad** (depende do item 5.4)

- [ ] **8.8 — Acessibilidade e conforto**
  Opção para desligar tremor de câmera e flashes (hoje fixos em `Camera`), escala de HUD, pausa que não perde input.

---

## Fase 9 — 🎓 Checklist do relatório da disciplina

Itens que o relatório marcou como ausentes/parciais. Alguns já foram resolvidos pelas refatorações posteriores; os que restam são, hoje, mais **exercício acadêmico** do que necessidade do código — com duas exceções marcadas abaixo.

**Já resolvidos** (só registrar, se for atualizar o relatório):

- [x] Requisito 10 / conceito 4.3 — persistência com recuperação em execução: `GerenciadorSave` versionado + `Fase::aplicarSave` + `Principal::recuperaFase` + seletor de slot com preview
- [x] Conceito 6.2 — classes aninhadas: `Gerenciador_Estados::Transicao`, `Animacao::Quadro`, `ArvoreHabilidades::Info`, `Menu::PlayerScore`, `Chefao::Estagio`
- [x] Conceito 6.1 — namespaces autorais: `Estados`, `Sistemas`, `Persistencia`, `Lista`
- [x] Conceito 8.4 — matemática/física de ensino superior: passo fixo com acumulador, suavização exponencial da câmera, `atan2` + normalização angular + clamp de velocidade angular no míssil teleguiado, colisão por profundidade de penetração

**Pendentes:**

- [ ] **9.1 — Templates dos autores (conceito 3.3) — REGRESSÃO**
  O relatório declarava "sim" (listas via template). Hoje há **zero** `template` no projeto: `ListaEntidade` virou `std::vector<std::unique_ptr<Entidade>>` concreto.
  **Proposta:** `ListaEntidade<T>` com `T` derivado de `Entidade`, eliminando parte dos 12 `dynamic_cast` de [Fase.cpp](game/JOGO_2D/Sources/Fase.cpp) — este é o único item da Fase 9 que **melhora o código de verdade**, não só o checklist.

- [ ] **9.2 — Sobrecarga de operadores (conceito 4.2)**
  Hoje só existe `ListaEntidade::operator[]` ([ListaEntidade.cpp:36](game/JOGO_2D/Sources/ListaEntidade.cpp#L36)). Candidatos com uso real: `operator<<` para `DadosSave` (simplifica `GerenciadorSave::salvar`), `operator==` para `EstadoJogador` (útil nos testes do item 6.3), `operator+=` para pontuação.

- [ ] **9.3 — Persistência de relacionamento (conceito 4.4)**
  O save grava escalares; a fase é **regerada** pela seed determinística, então inimigos vivos, HP e posições do meio da luta somem ao carregar. Além do checklist, isto é uma **falha de experiência real**: salvar durante a luta do chefão e carregar devolve a arena cheia outra vez.
  **Proposta:** gravar seed + delta (quais inimigos morreram, HP dos vivos) em vez do estado completo. Depende do item 5.11.

- [ ] **9.4 — Sobrecarga de construtoras e métodos (conceito 4.1)**
  Hoje existe apenas `removerEntidade(Entidade*)`/`(int)` e `getArvore()` const/não-const. Candidato natural e útil: o par idiomático de carregamento (`carregar` devolvendo booleano/tupla e `carregar!` que lança).

- [ ] **9.5 — Exceções próprias (conceito 3.4)**
  Existem 3 `throw std::runtime_error` e **1** `catch`. Depois do item 3.1, criar hierarquia própria (`ErroDeRecurso`, `ErroDeSave`, `ErroDeFase`) e tratar cada uma onde faz sentido — em vez de um `catch (std::exception&)` único no `main` que só sabe imprimir e sair.

- [ ] **9.6 — Padrões GOF > 5 (conceito 9.3)**
  Identificáveis hoje: **Singleton** (5 gerenciadores), **State** (`Estado` + pilha), **Factory Method** simples (`CarregadorFase::instanciarChar`), **Flyweight** parcial (texturas compartilhadas via `Gerenciador_Recursos` + `Animacao` guardando ponteiros). Os itens desta lista acrescentam naturalmente: **Object Pool** (5.5), **Observer** (5.3, eventos), **Strategy** (5.6, comportamento de IA), **Command** (5.4, ações de input). Ou seja: fazer as Fases 5 e 6 fecha 9.6 sem esforço dedicado.

- [ ] **9.7 — Threads (conceitos 7.3 e 7.4)**
  Ausentes (só um `#include <thread>` órfão em [Chefao.hpp:7](game/JOGO_2D/Headers/Chefao.hpp#L7), a remover no item 2.5).
  **Opinião honesta:** neste jogo, thread é solução em busca de problema. O uso legítimo seria carregamento assíncrono de assets com tela de loading real (`Principal::telaCarregamento` hoje desenha um quadro só e segue). Fazer por causa do checklist é introduzir concorrência sem necessidade — exatamente o tipo de complexidade que este documento tenta remover.

---

## Resumo da ordem de execução

| Fase | Natureza | Risco de não fazer |
|---|---|---|
| 0 | Ligar warnings/sanitizers | Continuar sem enxergar a Fase 1 |
| 1 | 🔴 UB e memória | Crash e comportamento aleatório |
| 2 | 🟠 Bugs e perda de dados | Jogador perde progresso e ranking |
| 3 | 🟡 Erro e ciclo de vida | Falhas silenciosas, saída suja |
| 4 | 🧹 Código morto | Refatorar código que ninguém usa |
| 5 | 🏗️ Estrutura | A arquitetura continua não correspondendo aos nomes |
| 6 | 🔧 Infra | Os bugs da Fase 1 voltam sem CI |
| 7 | ⚡ Performance | Degrada conforme as fases crescem |
| 8 | 🎮 Produto | O jogo funciona, mas sem som e sem retenção |
| 9 | 🎓 Checklist acadêmico | Nenhum, exceto 9.1 e 9.3 |

**Dependências que importam:** 1.1 antes de 3.1 · 4.x antes de 5.x · 5.1 antes de 5.2 e 7.2 · 5.4 antes de 8.7 · 5.11 antes de 9.3 · 6.1 antes de qualquer coisa que envolva compartilhar o repositório.
