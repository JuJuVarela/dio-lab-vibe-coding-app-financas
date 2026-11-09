# Finance Chat

**Controle financeiro pessoal numa conversa.**
Você escreve *"Gastei R$ 50 no mercado"* e o app registra a despesa, escolhe a categoria, salva data e hora, e ainda recomenda onde cortar gastos. Sem planilha, sem formulário.

🔗 **App publicado:** https://money-talk-companion.lovable.app

---

## 1. O problema

A maioria dos apps de finanças pessoais morre na segunda semana: exige preencher campos, escolher categorias num menu e lembrar de abrir o app toda vez que compra algo.

O Finance Chat inverte o fluxo. A entrada é uma frase do dia a dia, exatamente como você contaria a um amigo. Um agente de IA entende a frase, decide o que é (receita ou despesa), classifica, grava no seu banco e responde. Consultas do tipo *"quanto gastei este mês?"* viram uma pergunta, não uma filtragem de tabela.

## 2. Como funciona

```text
"Gastei R$ 50 no mercado"
        |
        v
  /api/chat  -->  agente de IA  -->  escolhe uma ferramenta
        ^                                |
        |                                v
   resposta escrita  <--  totais e listas  <--  banco (com sua sessão)
```

1. Você manda a frase no chat.
2. O agente decide a ação: registrar, consultar, corrigir ou criar meta.
3. A gravação acontece com a **sua** sessão — as regras de acesso do banco se aplicam a ela.
4. O dashboard, as metas e os relatórios se atualizam sozinhos, sem recarregar.

## 3. Funcionalidades (MVP 1.0)

| # | Funcionalidade | Como está no app |
|---|---|---|
| 1 | **Cadastro e login** | E-mail e senha, cadastro com confirmação por e-mail, recuperação de senha e definição de nova senha. |
| 2 | **Chat financeiro** | Campo de mensagem, histórico salvo no banco, sugestões prontas na primeira tela e resposta em streaming. |
| 3 | **Registro de transações** | Receitas e despesas com valor, descrição e data/hora; aceita data diferente quando você menciona uma ("ontem", "dia 3"). |
| 4 | **Classificação automática** | A IA escolhe a categoria a partir da frase; você corrige no chat ou no dashboard. |
| 5 | **Dashboard** | Total de receitas, total de despesas, saldo atual e últimas 15 movimentações, ao vivo. |
| 6 | **Metas financeiras** | Criar meta com valor alvo e prazo, acompanhar barra de progresso e aportar direto na tela. |
| 7 | **Relatórios** | Gastos por categoria (pizza), comparação mensal de receitas × despesas (6 meses) e período customizado. |
| 8 | **Agente financeiro** | Dicas de redução de gastos, alerta quando despesas passam de 80% das receitas, sugestão de cashback e passos para a meta. |

### Telas e rotas

| Tela | Rota | Acesso |
|---|---|---|
| Boas-vindas | `/` | pública |
| Login / Cadastro / Recuperar senha | `/auth` | pública |
| Definir nova senha | `/reset-password` | link do e-mail |
| Chat | `/chat` | logado |
| Dashboard | `/dashboard` | logado |
| Metas | `/metas` | logado |
| Relatórios | `/relatorios` | logado |
| Perfil | `/perfil` | logado |
| Configurações | `/configuracoes` | logado |
| Endpoint do agente | `/api/chat` (POST) | só com sessão válida |

## 4. O agente de IA

O agente roda no servidor, na rota `/api/chat` (`src/lib/ai/chat.server.ts`). Antes de qualquer coisa ele valida o token da sessão; sem token válido, a resposta é `401`. Ele recebe o contexto do dia (data e hora no fuso de São Paulo) e tem seis ferramentas:

| Ferramenta | O que faz |
|---|---|
| `registrar_transacao` | Grava receita ou despesa, com categoria automática, data e vínculo opcional a uma meta. |
| `consultar_periodo` | Soma receitas, despesas e saldo e quebra por categoria dentro de um intervalo. |
| `ultimas_transacoes` | Lista as últimas movimentações com ID, para correções. |
| `corrigir_categoria` | Troca a categoria de uma transação existente. |
| `criar_meta` | Cria meta com nome, valor alvo e prazo. |
| `listar_metas` | Lista metas e progresso. |

Todo texto final é gravado em `conversations` com a pergunta e a resposta, mantendo o histórico completo da conversa.

## 5. Stack

| Camada | Tecnologia |
|---|---|
| Framework | TanStack Start (React 19, rotas por arquivo, SSR) |
| Build | Vite |
| Estilo | Tailwind CSS v4 + componentes Radix (shadcn/ui) + Recharts |
| Dados no cliente | TanStack Query, com atualização em tempo real |
| IA | Vercel AI SDK sobre o gateway de IA do Lovable |
| Backend | Lovable Cloud — PostgreSQL gerenciado, autenticação, realtime e Row Level Security |
| Testes | Vitest + Testing Library |

## 6. Banco de dados

Quatro tabelas, todas com **Row Level Security**: cada usuário só enxerga as próprias linhas (`auth.uid() = user_id`).

| Tabela | Campos principais |
|---|---|
| `profiles` | `id`, `nome`, `email`, `moeda`, `data_criacao` |
| `transactions` | `id`, `user_id`, `tipo`, `valor`, `categoria`, `descricao`, `data_transacao`, `goal_id` |
| `goals` | `id`, `user_id`, `nome`, `valor_objetivo`, `valor_atual`, `data_limite` |
| `conversations` | `id`, `user_id`, `mensagem`, `resposta`, `data_hora` |

Dois gatilhos no banco:

- **criação de perfil** — ao confirmar o cadastro, a linha de `profiles` é criada sozinha.
- **progresso de meta** — `transactions` com `goal_id` recalcula `goals.valor_atual` na hora, então a meta avança sozinha quando você aporta.

## 7. Regras de negócio

| Regra | Situação |
|---|---|
| **RN01** — toda movimentação tem valor, tipo e data | Aplicada na ferramenta de registro: valor precisa ser positivo e a data é preenchida automaticamente se você não mencionar outra. |
| **RN02** — a IA classifica a categoria | A IA escolhe entre as categorias ao registrar. |
| **RN03** — o usuário corrige categorias | Correção pelo chat ou pelo menu de cada linha no dashboard. |
| **RN04** — histórico completo de conversas | Cada troca é gravada em `conversations` e recarregada na tela. |
| **RN05** — metas atualizam sozinhas | Gatilho no banco recalcula o progresso a cada movimentação ligada à meta. |
| **RN06** — dashboard em tempo real | Inscrição em mudanças do banco invalida as consultas; a tela muda sem recarregar. |

### Categorias

- **Despesa:** Alimentação, Transporte, Moradia, Saúde, Educação, Lazer, Compras, Serviços, Poupança, Outros
- **Receita:** Salário, Freelance, Investimentos, Outros

Aportes para metas entram como despesa na categoria *Poupança* com a meta vinculada — assim o dinheiro sai do saldo e a meta sobe ao mesmo tempo.

## 8. Rodando localmente

Pré-requisito: Node.js (ou Bun) instalado.

```sh
git clone <url-deste-repositorio>
cd <nome-do-repositorio>
npm install        # ou: bun install
npm run dev        # abre em http://localhost:8080
```

Outros comandos:

```sh
npm run build      # build de produção
npm run preview    # serve o build
npm run test       # testes
npm run lint       # eslint
npm run format     # prettier
```

As variáveis do backend são geradas automaticamente pelo Lovable Cloud no arquivo `.env` — não versionar segredos. O agente precisa de `LOVABLE_API_KEY` no servidor.

### Para experimentar

1. Abra `/auth` e crie uma conta com e-mail e senha.
2. Confirme pelo link enviado ao e-mail.
3. No chat, mande `Gastei R$ 50 no mercado`.
4. Abra o dashboard e veja a despesa lá, com categoria já escolhida.
5. Crie uma meta e mande `Guardei R$ 200 para a viagem` — a barra de progresso sobe sozinha.

## 9. Estrutura do projeto

```text
src/
  routes/
    index.tsx            tela de boas-vindas
    auth.tsx             login, cadastro e recuperação
    reset-password.tsx   definir nova senha
    api/chat.ts          endpoint POST do agente
    _authenticated/      área logada (redireciona quem não tem sessão)
      chat.tsx           chat
      dashboard.tsx      resumo e últimas movimentações
      metas.tsx          metas e aportes
      relatorios.tsx     gráficos e período
      perfil.tsx         nome do usuário
      configuracoes.tsx  trocar senha, apagar histórico
  lib/
    ai/chat.server.ts    agente, ferramentas e gravação do histórico
    finance.ts           categorias, formatação em R$ e datas
  hooks/use-finance.ts   consultas com atualização em tempo real
  components/            app shell, navegação e componentes de UI
```

## 10. Limitações sabidas

- A entrada é só e-mail e senha: não há login social (Google) nem cadastro anônimo.
- O cadastro exige confirmação de e-mail antes do primeiro acesso.
- O exemplo de conversa da tela de boas-vindas (com "R$ 1.240,00") é ilustrativo, não vem de dados reais.
- Precisão da classificação e retenção de usuários ainda não foram medidos com uso real.

## 11. Versão futura

Integração Open Finance · sincronização bancária · WhatsApp e Telegram · cartão de crédito · investimentos · planejamento financeiro avançado · gamificação.

---

Projeto criado no [Lovable](https://lovable.dev). As mudanças feitas no editor são commitadas no repositório conectado, e o que você der um push na branch sincroniza de volta para o editor.
