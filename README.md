# Site de Agendamento — Salão de Beleza

Site público onde as **clientes agendam e reagendam horários** com as profissionais do salão.

> **Importante:** este projeto é apenas o **site de agendamento** (lado do cliente). O **painel administrativo** (agenda, clientes, PDV, relatórios, configurações) fica em **outra pasta/projeto**: [`adm-salaodebeleza`](../adm-salaodebeleza). Os dois se conectam ao **mesmo projeto Firebase (Firestore)**.

```
 Site de agendamento (este)  ──►  Firebase / Firestore  ◄──  Painel ADM (adm-salaodebeleza)
   cliente agenda/reagenda        mesmo projeto, mesmas        profissionais, serviços, horários,
   lê serviços e horários         coleções                     bloqueios e visualização da agenda
```

- O **ADM cadastra** profissionais, serviços, horários e exceções → o **site lê** e monta a tela.
- O **site grava** o agendamento → o **ADM mostra** na Agenda em tempo real.

## Funcionalidades

- Página institucional (serviços, profissionais, localização, botão de avaliação Google)
- Lista de profissionais carregada **100% do Firestore**, a partir do cadastro feito no ADM (nome, foto, WhatsApp, coleção de reservas). **Nenhuma imagem de profissional fica neste projeto**
- Fluxo de agendamento: nome/sobrenome/telefone → RA Club (membro / não / assinar) → data → serviço → horário → confirmação
- Horários livres calculados pela **duração do serviço** (passo de 30 min), descontando reservas já existentes (conflito por intervalo)
- Respeita horário de funcionamento, dias de folga, fim de expediente da profissional e **exceções/bloqueios** definidos no ADM
- **Reagendamento**: a cliente busca pelo nome/sobrenome e move o agendamento (transação, com fallback em batch)

## Tecnologias

- HTML, CSS e JavaScript puro (ES Modules), **sem build e sem `npm install`**
- Firebase **Firestore** (SDK 10.12.2 via CDN `gstatic`) — sem Authentication (a cliente não faz login)
- Boxicons (ícones) via CDN

## Estrutura

| Arquivo | Função |
|---|---|
| `index.html` | Layout, seções e modais |
| `script.js` | Lógica do site + **configuração do Firebase** (`firebaseConfig`) + leitura/gravação no Firestore |
| `style.css` | Estilos |
| `assets/` | Logo e imagens institucionais (fotos das profissionais **não** ficam aqui; vêm do Firebase) |

## Como sincronizar com o Firebase

Os dois projetos só conversam se usarem **o mesmo `firebaseConfig`** (mesmo `projectId`).

### 1. Projeto Firebase
O site já aponta para o projeto **`studio-beleza-45b1d`** (`script.js`, bloco `firebaseConfig`, por volta da linha 569).
Para usar outro projeto: [Firebase Console](https://console.firebase.google.com/) → *Project settings → Your apps → Web (`</>`)* → copie o `firebaseConfig` e substitua no `script.js`. Ative o **Firestore Database**.

### 2. Configurar o painel ADM com o mesmo projeto
No projeto [`adm-salaodebeleza`](../adm-salaodebeleza), abra `firebase.js` e cole **o mesmo objeto** (hoje está com campos vazios):

```js
const firebaseConfig = {
  apiKey: "...",
  authDomain: "studio-beleza-45b1d.firebaseapp.com",
  projectId: "studio-beleza-45b1d",
  storageBucket: "studio-beleza-45b1d.firebasestorage.app",
  messagingSenderId: "...",
  appId: "..."
};
```

Copie os valores diretamente do `script.js` deste site. `projectId` precisa ser idêntico nos dois.

### 3. Cadastrar os dados no ADM (o site depende deles)
> **Profissionais não são cadastradas no site.** O site apenas lê a coleção `profissionais` do Firebase: cada cartão (nome, foto, botões de agendar/reagendar) é gerado a partir do cadastro feito no ADM. Para incluir, editar, trocar a foto ou remover uma profissional, faça no ADM — o site reflete na próxima abertura, sem alterar código nem adicionar imagens à pasta `assets/`.

No painel ADM, aba **Configurações**, cadastre:

1. **Profissionais**: nome, **foto** (campo `fotoUrl`, link https ou upload pelo ADM), WhatsApp e coleção de reservas. Sem nome e `colecao` a profissional não aparece no site
2. **Serviços** (nome, valor, tempo em minutos)
3. **Horários de funcionamento** e dias da semana
4. (Opcional) **Exceções/bloqueios** gerais ou por profissional

### 4. Coleções e campos usados pelo site

| Caminho no Firestore | Uso no site | Campos lidos |
|---|---|---|
| `profissionais` | lê | `nome`, `fotoUrl`/`foto`, `whatsapp`/`wa`, `colecao` (ou `slug` → `reservas_<slug>`), `fimExpediente`, `diasFolga` |
| `servicos` | lê | `nome`, `valor`/`preco`, `tempoMin`/`tempo`/`duracaoMin` |
| `config/horarios` | lê | `inicio`, `fim` |
| `config/semana` | lê | `weekdays` (0=dom … 6=sáb), `dayStart`, `dayEnd` |
| `config/excecoes/dias/{AAAA-MM-DD}` | lê | `fechado`, `inicio`, `fim` (exceção geral do dia) |
| `profissionais/{id}/excecoes/{AAAA-MM-DD}` | lê | `fechado`, `inicio`, `fim` (exceção da profissional; tem prioridade) |
| `reservas_<profissional>/{AAAA-MM-DD_HH:mm}` | **lê, cria, move** | ver abaixo |

Prioridade da disponibilidade: **exceção da profissional > exceção geral > semana**.

**Agendamento gravado pelo site** (documento com ID `AAAA-MM-DD_HH:mm` na coleção da profissional):

```
data, hora, profissional, profissionalNome,
clienteNome, clienteSobrenome, clienteNomeCompleto, clienteNomeLower, clienteSobrenomeLower, clienteTelefone,
servicoNome, servicoValor, servicoTempoMin,
raclub, raclubMembro, clienteTipo, tags, createdAt
```

> O ID do documento (`data_hora`) impede dois agendamentos no mesmo horário exato. Se o ADM criar agendamentos/bloqueios, deve usar o **mesmo padrão de ID e a mesma coleção** da profissional.

### 5. Regras do Firestore
O site não usa login, então as regras precisam permitir, **sem autenticação**:

- leitura de `profissionais` (e subcoleção `excecoes`), `servicos` e `config/**`
- leitura, criação e atualização/remoção (reagendamento) em `reservas_*`

O arquivo `firestore.rules` do projeto ADM já cobre isso — publique-o em *Firestore → Rules* (ou `firebase deploy --only firestore:rules`).

> ⚠️ **Atenção de segurança:** o `firebaseConfig` é público no navegador. As regras atuais liberam leitura/escrita sem login em várias coleções (inclusive `reservas_*`, `pdv_sales`, `despesas`...). Antes de ir para produção, restrinja as coleções de gestão ao e-mail do administrador (Authentication) e deixe público apenas o que o site precisa. Exemplo no README do ADM.

### 6. Executar

Por usar ES Modules, abra por um servidor local (não por `file://`):

```bash
npx serve .
# ou
python -m http.server 5500
```

Ou use a extensão **Live Server** do VS Code. Para publicar: Vercel, Netlify, Firebase Hosting ou qualquer hospedagem estática. Se publicar em um domínio próprio e restringir a API key, autorize esse domínio nas restrições da chave.

## Checklist de integração

- [ ] Firestore ativado no projeto Firebase
- [ ] `firebaseConfig` do `script.js` **idêntico** ao `firebase.js` do ADM
- [ ] Regras do Firestore publicadas (leitura pública do que o site usa, criação em `reservas_*`)
- [ ] Profissionais (com `colecao`), serviços e horários cadastrados no ADM
- [ ] Teste: agendar pelo site e conferir na **Agenda** do ADM
- [ ] Teste: bloquear um dia/horário no ADM e conferir que ele some/fica indisponível no site

## Observações

- As fotos das profissionais vêm do campo `fotoUrl` do cadastro no ADM. Cadastre a foto lá; não é preciso colocar imagens em `assets/`.
- Se o Firestore não retornar profissionais, o código usa uma profissional de fallback (`reservas_rodrigotorre2`) com a imagem `./assets/ra-rodrigo.jpg`, que não existe. Mantenha ao menos uma profissional cadastrada no ADM e, se desejar, remova esse fallback do `script.js`.
- `MAPA_FIM_EXPEDIENTE` e `DIAS_FOLGA` no `script.js` são valores de reserva por nome; o que vale é o cadastrado em `profissionais` (`fimExpediente`, `diasFolga`).
- Consultas por `where("data","==",...)` e `where("clienteNomeLower","==",...)` usam um único campo, então **não exigem índice composto**.
