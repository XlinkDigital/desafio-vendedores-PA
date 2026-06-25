# Desafio Vendedores — PA · Progresso

Documento de retomada. Última atualização: **23/06/2026**. HEAD na sessão: `1c64b9b`.

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
- **Vendedor:** entra só com **nome + número de celular**.
  - Não existe → cria cadastro em `participantes` com `status: "pendente"` (ativo:false) e mostra "Cadastro enviado! Aguarde a autorização do gestor."
  - `aprovado` (ativo:true) → entra e vê só o ranking dele (seletor travado no próprio nome).
  - `pendente` → "Aguardando autorização do gestor."
  - `recusado` → "Seu acesso não foi autorizado."

### Dados (Firebase / Firestore — projeto "desafio-boombox")
- `firebaseConfig` está no código. Login **Anônimo** ativado (só dá contexto pro Firestore; ninguém loga por conta).
- Coleção **`participantes`**: `{ nome, celular, rota, ativo, status, criadoEm }` (id = docId string).
- Coleção **`pontuacoes`**: `{ participantId, date, volumeVendido, volumeAvarias, qtdClientes, c, d, notes, criadoEm }`. As notas **A** e **B** NÃO são gravadas — são derivadas em `enrichScores()`/`calcRanking()`.
- Documentos e fotos ainda ficam em **localStorage** (`bb_docs`, `bb_photos`) — não migrados.
- `refreshAll()` recarrega participantes+pontuações do Firestore a cada troca de aba **e chama `syncParticipantSelects()`**.

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
- Botão "Resetar tudo (teste)" na aba Participantes (`resetarTudoTeste()`): apaga do Firestore **`pontuacoes` E `participantes`**, depois `refreshAll()` (que via `syncParticipantSelects()` zera todos os dropdowns) + re-render de Participantes/Solicitações/Histórico/Ranking.

### Ranking automático
- A aba **Ranking** (admin) e o **Ranking Geral** (vendedor) mostram TODOS os vendedores ativos automaticamente, sem exigir seleção. Empty-state amigável quando não há vendedor. Após importação, `loadAdmRanking()` é chamado para refletir na hora.
- Seleção manual de participante só onde faz sentido: **Registrar pontuação** e telas pessoais do vendedor (Meu Ranking / O Que Cumprir / Minha Evolução).

### Critérios (`calcRanking()` + `enrichScores()`)
Captura por período (R$ + kg + clientes): **Faturamento (R$)**, **Volume vendido (kg)**, **Volume de avarias (kg)** (perda+bonif+devol) e **Qtd. de clientes**. Pesos editáveis no topo (`PESO_A/B/C/D`, default 1). Campo `faturamento` agora é persistido em `pontuacoes` (antes era lido do HTML e descartado).
- **A) Trocas e Avarias** (menor = melhor): `avaria_% = volume_avarias / volume_vendido`; `nota = 100 - avaria_%*100*AVARIA_PTS_POR_PCT` (default 5 → 4% = 80, 20% = 0). Usa o período mais recente do vendedor.
- **B) Vendas Positivo** (crescimento do **TICKET MÉDIO mensal**, maior = melhor): `ticket_mês = faturamento_R$ / volume_kg` (agregado por mês-calendário `YYYY-MM`, somando todos os lançamentos do mês); `cresc_% = (ticket_atual - ticket_anterior)/ticket_anterior` vs o **próprio mês anterior** com ticket válido; `nota = CRESC_NOTA_BASE + cresc_%*CRESC_PTS_POR_PCT` (50 base, 2,5 pt/% → +20% = 100, manter = 50, -20% = 0). Sem mês anterior → **"—" (sem nota, não pontua)**. Premia valor agregado (R$/kg), não volume bruto. Filtrar "Critério B" no ranking admin ordena pelo **% bruto** (maior→menor), desempate por maior ticket. Validado com exemplo Danylo: 05/2026 R$5,07/kg → 06/2026 R$4,71/kg = −7,1%, nota 32,1.
- **C) Precedente Positivo** → `sc = min(100, avgC*5)*PESO_C` (lógica original, média 30 dias).
- **D) Consulta de Preço** → `sd = min(100, avgD*1.5)*PESO_D` (lógica original, média 30 dias).
- **Métrica de apoio exibida:** `volume_por_cliente = volume_vendido / qtd_clientes`.
- Total = soma das quatro notas (com peso). Ranking ordenado por total.
- **SEM R$/preço** — tudo proporcional em kg, pra ser justo entre portes diferentes.

### Telas
- **Vendedor:** Meu Ranking · O Que Cumprir · Minha Evolução · Ranking Geral.
- **Admin:** Ranking · **Como funciona** · Importar · **Dados do relatório** · Registrar · Participantes · Solicitações (aprovar/recusar) · Evolução · Documentos · Fotos.
- **Como funciona** (`#adm-como-funciona`): página estática explicando que A/B/C/D são NOTAS 0-100 (não quantidade), Total = soma (máx 400). Detalha A (fórmula + tabela 0%→100…20%→0), B (crescimento do ticket médio R$/kg), C (precedentes atendidos → nota), D (consultas de preço → nota).
- **Dados do relatório** (`#adm-dados`, `loadDadosRelatorio`): lê do Firestore os registros do período selecionado (dropdown `#dadosPeriodo` com as datas existentes) e mostra tabela persistente — Vendedor, Rota, Faturamento, Vendido, Ticket médio, Avarias, Avaria %, Clientes, Vol/cliente. Mesmo formato da prévia de importação.
- Separação garantida pela nav exibida (vendedor não acessa telas de admin; as 2 novas abas vivem dentro de `#navAdmin`).

### Visual
- **Navegação = menu lateral (sidebar) à esquerda** (`.sidebar`, 248px, `position:fixed`). Logo no topo (`.sidebar-logo`), itens verticais (`.nav` vira `flex-direction:column`; `.nav-btn` alinhado à esquerda, ativo verde), rodapé com nome+selo ADMIN/VENDEDOR (`#headerName`/`#headerRole`) e botão Sair (`.sidebar-logout`). Conteúdo em `.content` (`margin-left:248px`). Os ids `navVendedor`/`navAdmin` e `headerName`/`headerRole` foram mantidos (entrarApp inalterado).
- **Responsivo (≤900px):** sidebar recolhe (`transform:translateX(-100%)`), abre/fecha pelo hambúrguer da `.mobile-topbar` (barra verde fina) via `toggleSidebar()`/`closeSidebar()` (classe `.open` + overlay `#sidebarOverlay.show`). `goTo()` chama `closeSidebar()` ao trocar de aba.
- Logo da empresa (`adb.png`, transparente, otimizada 15MB→66KB / 400×161px) na sidebar, na barra mobile e nas **3 telas de acesso** (`.auth-logo-img`). `.header-btn` ainda é usado em botões de tabela (não remover).

## Histórico recente (mais recente em cima)
- (atual) feat: critério B = crescimento do TICKET MÉDIO mensal (R$/kg); persiste faturamento; campo R$ no Registrar + preview; ticket atual←anterior e % colorido no ranking; filtro B ordena por % bruto
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
- [ ] Migrar **documentos e fotos** (hoje em localStorage) para Firebase Storage/Firestore, se quiser que sejam compartilhados.
- [ ] Conferir/ajustar **regras do Firestore** conforme o uso real (já foram publicadas pelo dono).
- [x] ~~Edição de pontuação~~ — JÁ EXISTE (`editScore`/`saveScore`/`deleteScore` no histórico do Registrar, botões editar/excluir).
- [x] ~~Dropdown de mês no ranking de crescimento~~ — FEITO. Seletor "Mês de referência" na aba Ranking admin (`#rankMonth`); `calcRanking(refMonth)` ranqueia por um mês fixo ('YYYY-MM') ou pelo mês mais recente de cada vendedor (default ''). Com mês escolhido, A/B/C/D usam os lançamentos daquele mês; sem dados no mês → vendedor sem nota. Funções: `mesesDisponiveis`/`popularMesesRanking`/`changeRankMonth`/`fmtMesLabel`.
- [ ] Definir período de avaliação real para C/D (hoje usa "últimos 30 dias" fixo; A usa período mais recente; B agora é mês-calendário).
- [ ] Decidir o que fazer com `exemplo-relatorio-VENDAS.xlsx` / `exemplo-relatorio-AVARIAS.xlsx` (hoje **não commitados** na pasta): commitar, `.gitignore` ou remover.

## Notas técnicas para quem retomar
- Tudo num arquivo só: `index.html`. JS é módulo ES; funções usadas em `onclick` são expostas via `window.x`.
- Ids de participantes/pontuações são **strings** (docId do Firestore) — cuidado ao comparar (`===`, sem `parseInt`).
- **Dropdowns de participante:** sempre passar por `syncParticipantSelects()` (fonte única). Não popular um `<select>` de participante na mão — senão volta o bug de nome em cache.
- `signInAnonymously` precisa estar habilitado no Firebase Console (já está).
- **Gráficos:** são SVG/HTML próprios (`renderLineChart`/`renderBarChart`) — NÃO usa Chart.js (único CDN é Font Awesome). Linha: com 1 período só, centraliza o ponto e mostra aviso "É preciso mais de um período"; com 2+ desenha a linha. Barras (`.bar-item`) precisam de `height:100%` no CSS senão o `height:%` do `.bar-fill` colapsa (bug corrigido).
- **Sessão do admin** persiste em `sessionStorage('bb_admin')`: `loginAdmin` grava, `restoreAdminSession()` (chamado no boot) restaura, `logout` apaga. Se a página recarregar, o admin segue logado.
- **Uploads** (importar HTML, Documentos, Fotos) limpam `input.value=''` após cada envio → dá pra reenviar o mesmo arquivo sem recarregar. Nenhum input de arquivo está em `<form>` nem chama `location.reload`.
- Checagem rápida de sintaxe do JS:
  extrair o conteúdo entre `<script type="module">` e `</script>` e rodar `node --check`.
