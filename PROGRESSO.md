# Desafio Vendedores — PA · Progresso

Documento de retomada. Última atualização: **21/06/2026**. HEAD na sessão: `10630bb`.

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
- Coleção **`pontuacoes`**: `{ participantId, date, a, b, c, d, notes, criadoEm }`.
- Documentos e fotos ainda ficam em **localStorage** (`bb_docs`, `bb_photos`) — não migrados.
- `refreshAll()` recarrega participantes+pontuações do Firestore a cada troca de aba.

### Critérios (`calcRanking()` + `enrichScores()`)
Captura por período (kg + clientes): **Volume vendido (kg)**, **Volume de avarias (kg)** (perda+bonif+devol) e **Qtd. de clientes**. Pesos editáveis no topo (`PESO_A/B/C/D`, default 1).
- **A) Trocas e Avarias** (menor = melhor): `avaria_% = volume_avarias / volume_vendido`; `nota = 100 - avaria_%*100*AVARIA_PTS_POR_PCT` (default 5 → 4% = 80, 20% = 0). Usa o período mais recente do vendedor.
- **B) Vendas Positivo** (crescimento, maior = melhor): `cresc_% = (vol_atual - vol_anterior)/vol_anterior` vs o **próprio período anterior**; `nota = CRESC_NOTA_BASE + cresc_%*CRESC_PTS_POR_PCT` (50 base, 2,5 pt/% → +20% = 100, manter = 50, -20% = 0). Sem período anterior → **"—" (sem nota, não pontua)**.
- **C) Precedente Positivo** → `sc = min(100, avgC*5)*PESO_C` (lógica original, média 30 dias).
- **D) Consulta de Preço** → `sd = min(100, avgD*1.5)*PESO_D` (lógica original, média 30 dias).
- **Métrica de apoio exibida:** `volume_por_cliente = volume_vendido / qtd_clientes`.
- Total = soma das quatro notas (com peso). Ranking ordenado por total.
- **SEM R$/preço** — tudo proporcional em kg, pra ser justo entre portes diferentes.

### Telas
- **Vendedor:** Meu Ranking · O Que Cumprir · Minha Evolução · Ranking Geral.
- **Admin:** Ranking · Registrar · Participantes · Solicitações (aprovar/recusar) · Evolução · Documentos · Fotos.
- Separação garantida pela nav exibida (vendedor não acessa telas de admin).

### Visual
- Logo da empresa (`adb.png`, transparente, otimizada 15MB→66KB / 400×161px) no **cabeçalho verde** (`.header-logo-img`, ~44px) e nas **3 telas de acesso** (`.auth-logo-img`, ~200px largura, responsiva).

## Histórico desta sessão (mais recente em cima)
- `10630bb` feat: logo na tela de acesso (ajuste de tamanho ~200px)
- `513b938` feat: logo da empresa nas telas de login/seleção de perfil
- `05be224` perf: otimiza logo adb.png (15MB -> 66KB)
- `76b7aa3` feat: logo da empresa no cabeçalho
- `0a2958c` fix: filtro por critério funcional no ranking admin + remove authErrorMsg morto
- `a69cabb` feat: Firestore compartilhado + acesso do vendedor por nome+celular com aprovação do gestor

## Pendências / próximos passos
- [ ] Migrar **documentos e fotos** (hoje em localStorage) para Firebase Storage/Firestore, se quiser que sejam compartilhados.
- [ ] Conferir/ajustar **regras do Firestore** conforme o uso real (já foram publicadas pelo dono).
- [ ] Eventual tela de **edição de pontuação** (hoje admin só adiciona; não edita/exclui lançamento individual).
- [ ] Definir período de avaliação real (hoje o ranking usa "últimos 30 dias" fixo).

## Notas técnicas para quem retomar
- Tudo num arquivo só: `index.html`. JS é módulo ES; funções usadas em `onclick` são expostas via `window.x`.
- Ids de participantes/pontuações são **strings** (docId do Firestore) — cuidado ao comparar (`===`, sem `parseInt`).
- `signInAnonymously` precisa estar habilitado no Firebase Console (já está).
- Checagem rápida de sintaxe do JS:
  extrair o conteúdo entre `<script type="module">` e `</script>` e rodar `node --check`.
