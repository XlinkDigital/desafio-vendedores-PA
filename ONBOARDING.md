# Onboarding — Desafio Vendedores (Armazém das Bananas)

Guia pra quem vai mexer no app pela primeira vez. Siga na ordem.

> O app é **um único arquivo**: `index.html` (HTML + CSS + JavaScript juntos).
> Hospedado no **GitHub Pages**: ao dar `push` na branch `main`, o site republica sozinho em ~1 min.
> Site: https://xlinkdigital.github.io/desafio-vendedores-PA
> Repo: https://github.com/XlinkDigital/desafio-vendedores-PA

## 1. Acesso (o gestor faz pra você)
- **GitHub:** você precisa ser **colaborador com permissão Write** no repositório.
- **Firebase (opcional):** só se for mexer em login/regras/dados. Projeto `desafio-boombox` no console.firebase.google.com.

## 2. Instalar na sua máquina
- **Git** — https://git-scm.com
- **Claude Code** — assistente de IA que edita o código a partir do que você pede (é o que acelera tudo)
- **VS Code** (editor) — opcional, mas recomendado
- **Node.js** — opcional, só pra checagens/preview

## 3. Baixar o projeto
```bash
git clone https://github.com/XlinkDigital/desafio-vendedores-PA.git
cd desafio-vendedores-PA
```
**Leia o `PROGRESSO.md`** — ele tem o estado atual, como funciona, os critérios e as pendências.

## 4. Ver o app rodando (preview local)
- Dê **2 cliques no `index.html`** → abre no navegador. O Firebase funciona mesmo local (o config já está no código).
- Editou? **Recarregue a página** (Ctrl+F5) pra ver o resultado.

## 5. Fluxo do dia a dia (editar e publicar)
1. **Antes de começar**, pegue a versão mais nova:
   ```bash
   git pull origin main
   ```
2. Faça a mudança (peça pra IA ou edite o `index.html` na mão).
3. Teste no navegador.
4. Publique:
   ```bash
   git add index.html
   git commit -m "descrição curta da mudança"
   git push origin main
   ```
5. Em ~1 min o site oficial atualiza. Dê **Ctrl+F5** pra furar o cache do navegador.

## 6. Regras pra não quebrar nada
- **Não mude a lógica de cálculo dos 4 critérios** (função `calcRanking()` no `index.html`) sem alinhar com o time.
- Trabalhem **um de cada vez na `main`**. Se forem mexer juntos ao mesmo tempo, criem **branches** e usem **Pull Request**:
  ```bash
  git checkout -b minha-mudanca
  # ...edita, commita...
  git push origin minha-mudanca
  # abre Pull Request no GitHub
  ```
- Os **ids de participantes/pontuações são strings** (docId do Firestore) — não use `parseInt` neles.

## 7. Conferir se o JavaScript não quebrou (opcional, com Node)
Extraia o conteúdo entre `<script type="module">` e `</script>` para um arquivo `.mjs` e rode:
```bash
node --check arquivo.mjs
```

## 8. Onde estão as coisas no código (`index.html`)
- **CSS:** dentro de `<style>` no topo.
- **Telas de acesso** (seleção de perfil, vendedor, admin): blocos `auth-screen`.
- **App logado:** `<div id="app">` (cabeçalho + nav + seções).
- **JavaScript:** um único `<script type="module">` no fim — funções de dados (Firestore), ranking, telas e charts. Funções usadas em `onclick` são expostas via `window.x`.

Dúvidas sobre o que cada coisa faz: comece pelo `PROGRESSO.md`.
