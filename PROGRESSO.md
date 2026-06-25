# Desafio Vendedores — PA · Progresso

Documento de retomada. Última atualização: **25/06/2026**. HEAD anterior à sessão: `83a14a0`.

## O que é o projeto
App de ranking de vendedores do cliente **Armazém das Bananas** ("Desafio Boombox").
Página única `index.html` (HTML + CSS + JS num único `<script type="module">`), hospedada no **GitHub Pages**.

- **Repo:** https://github.com/xlinkdigital/desafio-vendedores-PA (org renomeada para `XlinkDigital`; o push redireciona sozinho).
- **Site publicado:** https://xlinkdigital.github.io/desafio-vendedores-PA
- **Clone local:** `C:\Users\marce\desafio-vendedores-PA`
- ⚠️ Ignorar `C:\Users\marce\Downloads\DESAFIO-VENDEDORES-PA.txt` — é uma versão ANTIGA.

## Como funciona hoje

### Acessos
- **Admin/Gestor:** entra por **chave** `ABANANAS2026` (nome + chave). Sem Firebase Auth.
- **Vendedor:** entra com **usuário (ou nome) + SENHA** — **[MUDOU 25/06/2026: era nome + celular].** O **gestor cria/gerencia o login** de cada vendedor no painel **Acessos & Permissões** (define `login` opcional + `senha`, libera/bloqueia). Não há mais auto-cadastro pendente na tela de login.
  - `acessarVendedor` carrega `participantes` e casa pelo `login` (se houver) **ou** pelo `nome`, normalizado (trim+lowercase). Ex.: usuário "Danylo" → cadastro "Danylo PA" (via `login`). Em homônimos, prioriza o cadastro liberado.
  - Erros distintos: **não encontrado** / **não liberado** (ativo:false ou status≠aprovado) / **senha incorreta**.
  - Liberado (`aprovado`/ativo:true) → entra e vê só o ranking dele (seletor travado no próprio nome).
  - ⚠️ **Senha em texto puro** no Firestore (modelo de baixa segurança, coerente com a chave de admin hardcoded + regras permissivas). Aceitável p/ desafio interno; não usar senha sensível.
  - Vendedores **importados** (celular vazio) agora logam normalmente: basta o gestor definir a senha deles no painel. Campo `celular` virou dado opcional de cadastro.

### Dados (Firebase / Firestore — projeto "desafio-boombox")
- `firebaseConfig` está no código. Login **Anônimo** ativado (só dá contexto pro Firestore; ninguém loga por conta).
- Coleção **`participantes`**: `{ nome, celular, rota, ativo, status, criadoEm }` (id = docId string).
- Coleção **`pontuacoes`**: `{ participantId, date, volumeVendido, volumeAvarias, qtdClientes, c, d, notes, criadoEm }`. As notas **A** e **B** NÃO são gravadas — são derivadas em `enrichScores()`/`calcRanking()`.
- `refreshAll()` recarrega participantes+pontuações do Firestore a cada troca de aba **e chama `syncParticipantSelects()`**.
- **Abas Documentos e Fotos REMOVIDAS** (24/06/2026). Junto saíram: inputs `docInput`/`photoInput`, funções `uploadDocs`/`loadAdmDocs`/`downloadDoc`/`deleteDoc`(local)/`uploadPhotos`/`loadAdmPhotos`/`viewPhoto`, helpers `getLocal`/`setLocal`/`initLocalData`, buckets localStorage `bb_docs`/`bb_photos` e CSS `.upload-zone`/`.gallery*`/`.doc-row-ico`. (Mantidos `.doc-row`/`.doc-row-info`/`.doc-row-actions` — reusados em Participantes/Solicitações — e o import Firestore `deleteDoc as fbDeleteDoc`.)

### Fonte única de verdade dos seletores (importante)
- `syncParticipantSelects()` é a **única** função que popula TODOS os dropdowns de participante a partir de `DB.participants`: `scoreParticipant` (Registrar), `selParticipant` (Meu Ranking), `selEvolucao` (Minha Evolução) e `selAdminEvo` (Evolução admin).
- Preserva a seleção atual só se o participante ainda existir; senão volta ao placeholder.
- É chamada dentro de `refreshAll()` → todos os selects ficam sempre sincronizados e **zeram sozinhos após um reset** (sem nome em cache).
- `initParticipantSelects` / `initRegistrar` / `initAdminEvo` **delegam** a ela (não populam dropdown por conta própria).

### Importação de relatórios
- **HTML** (relatório único da unidade): separa por vendedor, **somente unidade Parauapebas**; cria vendedores ausentes já como `ativo:true / status:'aprovado'`. Modal resolve homônimos. Lança/atualiza `pontuacoes` por período (não zera C/D existentes).
- **Parser flexível (`parseRelatorioVendedoresHTML`) aceita os 2 formatos:** Vendas (Vendedor/Loja/Faturamento/Volume/Clientes) e Avarias ("Extrato das Operacoes", tabelas "Ranking por vendedor (volume)" com #/Vendedor/Loja/Operacoes/Volume kg, **SEM Clientes**). Qualifica QUALQUER tabela com coluna Vendedor + coluna de volume ("Volume"/"Volume kg"); Clientes e Faturamento opcionais. **Soma o volume por vendedor entre todas as tabelas** (avarias pode ter várias). Número robusto via `parseNum`: "8.619,8" (BR, ponto=milhar, vírgula=decimal) e "134485" (sem separador), tira " kg". Filtra Parauapebas pela coluna Loja quando existe. Erro claro só se nenhuma tabela Vendedor+Volume for achada.
- **XLSX** (SheetJS via CDN): uploads de Relatório de Vendas e de Avarias; lê aba "Resumo" → volume (linha "TOTAL GERAL") e nº de clientes; preenche os 3 campos kg (editáveis antes de salvar).
- Ao escolher o vendedor no Registrar, o campo **Precedentes (C)** é auto-preenchido com o último registro dele (editável).

### Reset de teste
- Botão "Resetar tudo (teste)" na aba Participantes (`resetarTudoTeste()`): apaga do Firestore **`pontuacoes` E `participantes`**, depois `refreshAll()` (que via `syncParticipantSelects()` zera todos os dropdowns) + re-render de Participantes/Acessos/Histórico/Ranking.

### Acessos & Permissões (admin) — NOVO 25/06/2026
Aba `#adm-acessos` ("Acessos & Permissões", ícone `fa-user-lock`) que **absorveu a antiga aba "Solicitações"**. Lista TODOS os vendedores (pendentes no topo); por linha: usuário (`login`, opcional) + `senha` (input texto) + **Liberar/Salvar** (`salvarAcesso` → grava `login`/`senha`, `ativo:true`, `status:'aprovado'`) e **Bloquear** (`bloquearAcesso` → `ativo:false`, `status:'recusado'`); pendentes têm **Recusar** (`recusarSolicitacao`). Funções: `loadAcessos`/`salvarAcesso`/`bloquearAcesso`/`recusarSolicitacao`. Removidos `loadAdmPending`/`aprovarSolicitacao` (chamada em `resetarTudoTeste` trocada p/ `loadAcessos`). `refreshParticipants` agora mapeia `login` e `senha`. O badge `#pendingBadge` (na nav) segue contando pendentes via `updatePendingBadge`. **Não há mais auto-cadastro** na tela de login do vendedor — o acesso é criado/liberado aqui pelo gestor.

### Ranking automático
- A aba **Ranking** (admin) e o **Ranking Geral** (vendedor) mostram TODOS os vendedores ativos automaticamente, sem exigir seleção. Empty-state amigável quando não há vendedor. Após importação, `loadAdmRanking()` é chamado para refletir na hora.
- Seleção manual de participante só onde faz sentido: **Registrar pontuação** e telas pessoais do vendedor (Meu Ranking / O Que Cumprir / Minha Evolução).

### Critérios (`calcRanking()` + `enrichScores()`)
**São 3 critérios: A, B e C.** O critério **D (Consulta de Preço) foi REMOVIDO** em 25/06/2026 — não conta mais. Saiu do cálculo, do form Registrar, do ranking, das telas, da "Como funciona", do **CSS** (regras mortas `.criteria-icon.d`/`.criteria-bar-fill.d`/`.explain-card.d`/`.explain-card.d .explain-badge`/`.input-group.d` removidas) e do **JS** (não lê nem grava mais o campo `d`: removido de `normalizeScores`, do payload de import HTML e do default de `latest`). "Como funciona" e a Dica agora dizem **3 critérios / máx 300** (era "4 critérios / 400"). Docs antigos do Firestore podem ter `d`, mas é simplesmente ignorado.
Captura por período (R$ + kg + clientes): **Faturamento (R$)**, **Volume vendido (kg)**, **Volume de avarias (kg)** (perda+bonif+devol) e **Qtd. de clientes**. Pesos editáveis no topo (`PESO_A/B/C`, default 1). Campo `faturamento` persistido em `pontuacoes`.
- **A) Trocas e Avarias** (menor = melhor): `avaria_% = volume_avarias / volume_vendido`; `nota = 100 - avaria_%*100*AVARIA_PTS_POR_PCT` (default 5 → 4% = 80, 20% = 0). Usa o período mais recente do vendedor.
- **B) Vendas Positivo** (crescimento do **TICKET MÉDIO semanal**, maior = melhor) — **[MUDOU 25/06/2026: mensal → semanal].** Captura é **semanal**: **1 lançamento = 1 semana**. `ticket = faturamento_R$ / volume_kg` do **próprio lançamento** (sem agregar por mês); `cresc_% = (ticket_atual - ticket_anterior)/ticket_anterior` vs o **lançamento (semana) anterior** do vendedor com ticket válido — semana sem R$/kg válido **não pontua e não vira referência** (a próxima compara com a última semana válida, pulando a inválida); `nota = CRESC_NOTA_BASE + cresc_%*CRESC_PTS_POR_PCT` (50 base, 2,5 pt/% → +20% = 100, manter = 50, -20% = 0). Sem semana anterior → **"—" (sem nota, não pontua)**. Premia valor agregado (R$/kg), não volume bruto. Filtrar "Critério B" no ranking admin ordena pelo **% bruto** (maior→menor), desempate por maior ticket. Lógica em `enrichScores()` (varredura cronológica com `prevTicket`). (Campos `ticketMes`/`ticketPrevMes`/`crescTicketPct` mantiveram o nome, mas agora = ticket da **semana** atual/anterior.)
- **C) Precedente Positivo** → `sc = min(100, avgC*5)*PESO_C` (lógica original, média 30 dias).
- **Métrica de apoio exibida:** `volume_por_cliente = volume_vendido / qtd_clientes`.
- Total = soma das **três** notas (com peso). **Máx 300.** Ranking ordenado por total. Barra de progresso do vendedor usa `total/300`.
- **SEM R$/preço** — tudo proporcional em kg, pra ser justo entre portes diferentes.

### Visualização do vendedor (vínculos titular ⊃ assistentes) — NOVO 25/06/2026
Recurso onde o **admin parametriza o que o VENDEDOR vê**, começando por **unificar vendas** (titular + assistente). Caso: Mailson é assistente do Danilo → na visão do Danilo os resultados aparecem **somados** e o Mailson **some do ranking** que o Danilo enxerga. **Admin intocado** (vê todos separados, números reais). **Não altera o cálculo de A/B/C** — a soma é só na **camada de exibição**. (Tentativas anteriores `09feb6b`/`fd90654` mexiam no cálculo e foram **revertidas**; esta abordagem é diferente.)
- **Coleção Firestore `vinculos`**: doc id = id do **titular**, `{ titularId, assistentes: [ids], atualizadoEm }`. Salvo via `setDoc` (idempotente); sem assistentes → `deleteDoc`. Carregado em `refreshVinculos()` (entra no `Promise.all` de `refreshAll`), exposto por `getVinculos()`. ⚠️ **Precisa de regra própria no Firebase Console** (ver Pendências).
- **Roteamento na camada de dados:** `getParticipants()`/`getScores()` devolvem o dataset **unificado** (`DB.viewParticipants`/`DB.viewScores`) **só em sessão de vendedor** (`useVendorView()`); no admin devolvem os dados crus. Assim TODAS as telas do vendedor (Meu Ranking, O Que Cumprir, Minha Evolução, Ranking Geral) e o `calcRanking` pegam a soma sem alteração individual; o admin usa o mesmo `calcRanking` com dados crus.
- **`rebuildVendorView()`** (roda em toda `refreshAll`): monta a visão **relativa a quem está logado**. Aplica todos os vínculos **menos aquele em que o viewer é assistente** (assim o assistente continua se vendo individual → no caso Mailson-só-assistente, `aplicar` fica vazio e usa dados crus). Esconde os assistentes (`DB.viewParticipants` sem eles) e, por **data**, **soma** os campos crus do titular+assistentes (volumeVendido, volumeAvarias, qtdClientes, faturamento, c) num lançamento sintético `participantId=titular`; depois `enrichScores(scores)` recalcula A/B com as **mesmas fórmulas**. `enrichScores(list=DB.scores)` foi parametrizado p/ aceitar a lista da visão.
- **Aba admin `#adm-visualizacao`** ("Visualização do vendedor", ícone `fa-eye`, em `#navAdmin`): dropdown `#vincTitular` + checkboxes de assistentes (`onChangeTitular` exclui titulares e assistentes já usados em outros vínculos, evitando duplo-vínculo/ciclo) + `salvarVinculo()`; card "Vínculos configurados" (`renderVincLista`/`removerVinculo`). Init via `initVisualizacao()` no `goTo`. Validado por simulação (admin separado / Danilo somado sem Mailson / Mailson individual).

### Telas
- **Vendedor:** Meu Ranking · O Que Cumprir · Minha Evolução · Ranking Geral.
- **Admin:** Ranking · **Como funciona** · Importar · **Dados do relatório** · Registrar · Participantes · **Acessos & Permissões** · Evolução · **Visualização do vendedor**.
- **Ranking Geral do vendedor (visibilidade) — NOVO 25/06/2026:** o vendedor vê a **posição e o nome** de todos (a dele e a dos colegas), mas os **pontos/dados só aparecem pra ele mesmo** — os dos colegas viram "—" (pódio e tabela). Implementado em `loadRankingGeral` (vale igual para todos os vendedores). O **admin** continua vendo tudo (`loadAdmRanking` intocado).
- **Como funciona** (`#adm-como-funciona`): página estática explicando que A/B/C são NOTAS 0-100 (não quantidade), Total = soma (máx 300). Detalha A (fórmula + tabela 0%→100…20%→0), B (crescimento do ticket médio R$/kg) e C (precedentes atendidos → nota).
- **Dados do relatório** (`#adm-dados`, `loadDadosRelatorio`): lê do Firestore os registros do período selecionado (dropdown `#dadosPeriodo` com as datas existentes) e mostra tabela persistente — Vendedor, Rota, Faturamento, Vendido, Ticket médio, Avarias, Avaria %, Clientes, Vol/cliente. Mesmo formato da prévia de importação.
- Separação garantida pela nav exibida (vendedor não acessa telas de admin; as 2 novas abas vivem dentro de `#navAdmin`).

### Visual
- **Pódio (top 3):** tamanho/altura/cor seguem a COLOCAÇÃO via classes `.pos-1/.pos-2/.pos-3` (não `nth-child`). Desenho em `order=[1,0,2]` (2º esquerda · 1º centro · 3º direita); 1º sempre maior/mais alto, ouro+brilho+coroa, 2º prata, 3º bronze. (Antes o CSS usava `nth-child` e o 2º saía maior.)
- **Coroa 👑:** `.podium-slot` tem `position:relative` (contexto), `.podium-crown` fica `top:-28px; z-index:5` (acima do avatar, que é irmão posicionado posterior senão a cobre), `.podium` tem `padding-top:38px` + `overflow:visible` p/ não cortar. No mobile (≤768px) o pódio vira coluna com `gap:30px` p/ a coroa não encostar no slot acima. Validado por screenshot (desktop + mobile) com puppeteer.
- **Importar:** botão **Cancelar importação** (`#batchCancelBtn`, `cancelarImportacaoLote()`, estilo vermelho claro) ao lado do Confirmar — descarta o staging via `limparBatch()` sem salvar no Firestore. `renderBatchPreview` mostra/esconde os dois juntos. (`limparBatch` agora aponta p/ `#adm-importar`.)
- **Navegação = menu lateral (sidebar) à esquerda** (`.sidebar`, 248px, `position:fixed`). Logo no topo (`.sidebar-logo`), itens verticais (`.nav` vira `flex-direction:column`; `.nav-btn` alinhado à esquerda, ativo verde), rodapé com nome+selo ADMIN/VENDEDOR (`#headerName`/`#headerRole`) e botão Sair (`.sidebar-logout`). Conteúdo em `.content` (`margin-left:248px`). Os ids `navVendedor`/`navAdmin` e `headerName`/`headerRole` foram mantidos (entrarApp inalterado).
- **Responsivo (≤900px):** sidebar recolhe (`transform:translateX(-100%)`), abre/fecha pelo hambúrguer da `.mobile-topbar` (barra verde fina) via `toggleSidebar()`/`closeSidebar()` (classe `.open` + overlay `#sidebarOverlay.show`). `goTo()` chama `closeSidebar()` ao trocar de aba.
- Logo da empresa (`adb.png`, transparente, otimizada 15MB→66KB / 400×161px) na sidebar, na barra mobile e nas **3 telas de acesso** (`.auth-logo-img`). `.header-btn` ainda é usado em botões de tabela (não remover).

## Histórico recente (mais recente em cima)
- (atual) feat: **Acessos & Permissões** — login do vendedor vira usuário/nome + **senha** (gestor cria/libera no painel, que absorve "Solicitações"); ranking do vendedor mostra posição+nome dos colegas mas **esconde os pontos/dados dos outros** (só os dele aparecem). `login`/`senha` no `participantes`; `loadAcessos`/`salvarAcesso`/`bloquearAcesso`. Admin intocado.
- feat: **Visualização do vendedor** — admin parametriza o que o vendedor vê; unifica vendas (titular + assistentes somados na visão do titular, assistentes somem do ranking dele). Coleção `vinculos`, roteamento em `getScores`/`getParticipants` por sessão, `rebuildVendorView()`, `enrichScores(list)` parametrizado, aba `#adm-visualizacao`. Admin e cálculo A/B/C intocados.
- feat: critério B passa a ser **semanal** (semana vs semana anterior, 1 lançamento = 1 semana) em vez de mensal; dropdown do ranking vira **"Semana de referência"** (`#rankWeek`, datas `YYYY-MM-DD`); textos "Como funciona"/preview/badge atualizados pra "semana". Funções renomeadas: `mesesDisponiveis→semanasDisponiveis`, `popularMesesRanking→popularSemanasRanking`, `fmtMesLabel→fmtSemanaLabel`, `changeRankMonth→changeRankWeek`; `calcRanking(refMonth)→calcRanking(refDate)`. `enrichScores` sem agregação por mês-calendário.
- refactor: remover critério D (Consulta de Preço) — agora são 3 critérios (A/B/C), máx 300; saiu do cálculo, form, ranking, telas e "Como funciona"
- revert do ajuste "assistente/Mailson" (2 commits) — Mailson volta a ser vendedor normal; importação e ranking sem lógica de assistente
- `2ee6531` feat: critério B = crescimento do TICKET MÉDIO mensal (R$/kg); persiste faturamento; campo R$ no Registrar + preview; ticket atual←anterior e % colorido no ranking; filtro B ordena por % bruto
- `1c64b9b` fix: reset limpa participantes e sincroniza; ranking automatico (fonte única `syncParticipantSelects`)
- `6600d69` feat: apagar todos os registros / reset de teste
- `fb47653` feat: importar HTML, somente unidade Parauapebas
- `8be9260` feat: importar relatório único HTML e separar por vendedor
- `5201799` feat: auto-preencher Precedentes com o último registro ao escolher vendedor
- `a1b855c` feat: importar relatórios xlsx e calcular A/B por volume
- `c6c8471` feat: pontuação A e B por volume/avarias/clientes em kg
- `f73c526` docs: ONBOARDING.md com guia de setup para novos colaboradores
- `ae8e702` docs: PROGRESSO.md com estado do projeto e próximos passos
- `10630bb` feat: logo na tela de acesso (ajuste de tamanho ~200px)
- `0a2958c` fix: filtro por critério funcional no ranking admin + remove authErrorMsg morto
- `a69cabb` feat: Firestore compartilhado + acesso do vendedor por nome+celular com aprovação do gestor

## Pendências / próximos passos
- [ ] ⚠️ **AÇÃO DO DONO — regra Firestore p/ `vinculos`:** as regras são por coleção (`participantes`/`pontuacoes`). A coleção nova **`vinculos`** precisa de regra própria, senão salvar/ler vínculo dá *permission-denied*. No Console (https://console.firebase.google.com/project/desafio-boombox/firestore/rules) adicionar: `match /vinculos/{doc} { allow read, write: if request.auth != null; }`
- [ ] Conferir/ajustar **regras do Firestore** conforme o uso real (já foram publicadas pelo dono).
- [x] ~~Edição de pontuação~~ — JÁ EXISTE (`editScore`/`saveScore`/`deleteScore` no histórico do Registrar, botões editar/excluir).
- [x] ~~Dropdown de semana no ranking de crescimento~~ — FEITO (era "de mês"). Seletor **"Semana de referência"** na aba Ranking admin (`#rankWeek`); `calcRanking(refDate)` ranqueia por uma semana fixa (`'YYYY-MM-DD'`) ou pela semana mais recente de cada vendedor (default ''). Com semana escolhida, A/B/C usam o lançamento daquela data; sem dados na semana → vendedor sem nota. Funções: `semanasDisponiveis`/`popularSemanasRanking`/`changeRankWeek`/`fmtSemanaLabel`.
- [ ] Definir período de avaliação real para C (hoje usa "últimos 30 dias" fixo; A usa período mais recente; B é semana vs semana anterior). Com captura semanal, talvez C deva virar média das últimas N semanas.
- [ ] Decidir o que fazer com `exemplo-relatorio-VENDAS.xlsx` / `exemplo-relatorio-AVARIAS.xlsx` (hoje **não commitados** na pasta): commitar, `.gitignore` ou remover.

## Notas técnicas para quem retomar
- Tudo num arquivo só: `index.html`. JS é módulo ES; funções usadas em `onclick` são expostas via `window.x`.
- Ids de participantes/pontuações são **strings** (docId do Firestore) — cuidado ao comparar (`===`, sem `parseInt`).
- **Dropdowns de participante:** sempre passar por `syncParticipantSelects()` (fonte única). Não popular um `<select>` de participante na mão — senão volta o bug de nome em cache.
- `signInAnonymously` precisa estar habilitado no Firebase Console (já está).
- **Gráficos:** são SVG/HTML próprios (`renderLineChart`/`renderBarChart`) — NÃO usa Chart.js (único CDN é Font Awesome). Linha: com 1 período só, centraliza o ponto e mostra aviso "É preciso mais de um período"; com 2+ desenha a linha. Barras (`.bar-item`) precisam de `height:100%` no CSS senão o `height:%` do `.bar-fill` colapsa (bug corrigido).
- **Sessão do admin** persiste em `sessionStorage('bb_admin')`: `loginAdmin` grava, `restoreAdminSession()` (chamado no boot) restaura, `logout` apaga. Se a página recarregar, o admin segue logado.
- **Uploads** (importar HTML) limpam `input.value=''` após cada envio → dá pra reenviar o mesmo arquivo sem recarregar. Nenhum input de arquivo está em `<form>` nem chama `location.reload`.
- Checagem rápida de sintaxe do JS:
  extrair o conteúdo entre `<script type="module">` e `</script>` e rodar `node --check`.
