# Dashboard — Fluxo Financeiro (DRE 2026)

Dashboard financeiro do escritório Borges Macedo Advocacia, alimentado em tempo real
pela planilha do Google Sheets **"Fluxo Financeiro"**.

## Acesso

A página abre numa tela de login e **nada é carregado antes da senha correta**.

A proteção não é uma verificação em JavaScript — isso seria inútil num repositório
público, já que qualquer pessoa leria a senha no código ou pularia a checagem pelo
devtools. Em vez disso:

- O **ID da planilha e a chave da API não existem em texto puro** no código. Eles ficam
  cifrados com **AES-256-GCM** dentro do bloco `COFRE`.
- A chave de decifragem é derivada com **PBKDF2-SHA256, 310.000 iterações**, a partir do
  login + senha digitados.
- O AES-GCM é autenticado: **senha errada faz a decifragem falhar**. Não existe um "sim"
  para forjar — sem a senha, a página simplesmente não sabe qual planilha ler.
- A sessão fica só na aba (`sessionStorage`) e **termina ao fechar a aba**. O botão
  **Sair** encerra na hora.

O login não fica documentado aqui. Peça a quem administra o dashboard.

### Trocar a senha

Abra o **`gerar-senha.html`** direto do seu computador (não pela internet), preencha o ID
da planilha, a chave da API, o login e a nova senha, e substitua o bloco `const COFRE = {…};`
do `index.html` pelo texto gerado. Depois é só commit e push.

> Nunca faça commit da senha, e não cole o ID da planilha ou a chave da API em texto puro
> em nenhum arquivo do repositório.

## Como funciona

- O dashboard lê a aba **`DRE 2026`** (valores por mês × projeção) e a aba
  **`Fluxo Financeiro Anual`** (apenas as colunas de data, para saber até que dia a
  planilha está lançada).
- Os dados são buscados no navegador a cada carregamento, com atualização automática
  a cada **5 minutos** (e também ao voltar para a aba do navegador).
- Não há build nem servidor: é um único arquivo `index.html`.
- **Qualquer alteração feita na planilha aparece no dashboard sem precisar mexer no código.**

## O que é exibido

| Seção | Conteúdo |
|---|---|
| Resumo do Período | Receita bruta, receita líquida, total de despesas, impostos, lucro líquido e saldo em caixa — cada um com realizado, projeção, % e excedente |
| Realizado × Projeção | Tabela completa do DRE, com grupos expansíveis até a categoria. Colunas: projeção, realizado, % da projeção, excedente e barra de consumo |
| % Gasto por Categoria | Rosca por grupo de despesa, barras das maiores categorias e tabela com % das saídas, % da receita, projeção e excedente |
| Impostos | Seção separada: valor pago, % sobre receita bruta, % sobre receita líquida, % sobre as saídas, desvio da alíquota de referência e série mensal |
| Evolução Mensal | Receita, saídas e lucro mês a mês; consumo da projeção de despesas; saldo em caixa e balanço |
| Pontos de Atenção | Alertas gerados automaticamente (estouro de orçamento, prejuízo, queima de caixa, concentração, imposto fora da referência, etc.) |
| Observações | Leitura de contexto: composição das saídas, maiores gastos, margem, ponto de equilíbrio, cobertura dos dados |

## Filtros

- **Menu suspenso** com três blocos:
  - *Grupos de meses* — ano completo, acumulado até o mês atual, trimestres e semestres
  - *Mês individual* — Janeiro a Dezembro (indicando qual está em andamento ou sem lançamentos)
  - *Personalizado*
- **Botões de mês** (Jan…Dez) para montar qualquer combinação, inclusive meses
  não consecutivos (ex.: Janeiro + Julho). O menu suspenso se ajusta sozinho quando a
  combinação corresponde a um grupo pronto.
- **Base de comparação:**
  - `Projeção cheia` — projeção mensal × número de meses selecionados
  - `Projeção proporcional` — projeção ajustada aos dias já decorridos

## Sobre o mês em andamento

O dashboard detecta sozinho até que data a planilha está lançada (última data da aba
`Fluxo Financeiro Anual`) e marca o mês corrente como parcial — com aviso no topo,
asterisco nos gráficos e um ponto de atenção dedicado.

Vale saber que **nenhuma das duas bases de comparação é perfeita para um mês aberto**:

- a *projeção cheia* subestima os percentuais de receita, porque o faturamento do mês
  ainda não entrou todo;
- a *projeção proporcional* superestima os de despesa, porque folha, aluguel e
  mensalidades são pagos de uma vez no início do mês, e não diluídos dia a dia.

Para decisão, o **último mês fechado** continua sendo a base mais confiável.

## Como os números são calculados

| Indicador | Origem |
|---|---|
| Receita, despesas, impostos, lucro | Lidos diretamente das linhas do DRE, na coluna do mês |
| Projeção | Coluna `PROJEÇÃO` do DRE, multiplicada pelo número de meses (ou pela fração decorrida) |
| % da projeção | `realizado ÷ projeção` |
| Excedente | `realizado − projeção` (positivo em despesa = estouro; negativo = economia) |
| **Saídas totais** | `Total de Despesas do DRE + Despesas Bancárias` |

> A linha "TOTAL DESPESAS" da planilha **não** inclui as taxas bancárias, porque elas já
> são abatidas na receita líquida. O dashboard soma as duas para que o percentual por
> categoria feche em 100% — por isso a base do "% das saídas" é maior que o total de
> despesas do DRE.

As referências usadas nos alertas vêm das próprias fórmulas da planilha:
**8% da receita bruta** para impostos (`D67 = D6*8%`) e **11,6% dos honorários iniciais**
para taxas bancárias (`D15 = D7*11,6%`).

## Configuração

Os parâmetros ficam no bloco `CONFIG`, no início do `<script>` do `index.html`:

| Campo | Descrição |
|---|---|
| `COFRE` | ID da planilha e chave da API, cifrados (veja *Acesso*) |
| `ABA_DRE` / `ABA_FLUXO` | Nomes das abas lidas |
| `AUTO_REFRESH_MIN` | Intervalo da atualização automática, em minutos |
| `ALIQUOTA_IMPOSTO_REF` | Alíquota de referência de imposto (0,08 = 8%) |
| `TAXA_BANCARIA_REF` | Taxa bancária de referência (0,116 = 11,6%) |
| `LIM_*` | Limiares que disparam cada ponto de atenção |

> A planilha precisa estar compartilhada como **"qualquer pessoa com o link pode ver"**
> para que o dashboard consiga lê-la.

### Robustez a mudanças na planilha

O dashboard **não usa números de linha fixos**. Ele localiza o cabeçalho procurando a
linha que contém "PROJEÇÃO" e os nomes dos meses, e monta a hierarquia pelos prefixos
`(+)`, `(-)` e `(=)` dos rótulos. Incluir, remover ou reordenar categorias na planilha
não quebra o dashboard — a nova categoria aparece sozinha.

Se a API do Google falhar, há um plano B automático que lê a exportação CSV pública
da planilha.

## Publicação

O site é servido pelo GitHub Pages a partir da branch `main`, pasta raiz.
Qualquer alteração enviada para a `main` entra no ar em cerca de 1 minuto.

## Segurança — o que a senha protege e o que não protege

**Protege:** o acesso ao dashboard e às credenciais da planilha. Sem a senha não é
possível descobrir qual planilha ler nem qual chave usar, e o histórico do git não
contém esses valores em momento algum.

**Não protege:** a planilha em si. Para o dashboard conseguir lê-la, ela precisa estar
compartilhada como *"qualquer pessoa com o link pode ver"*. **Quem já tiver o link da
planilha vê tudo, sem passar por senha nenhuma.** A senha do dashboard não muda isso.

Duas consequências práticas:

1. Quem faz login legitimamente pode extrair o ID da planilha e guardá-lo. Se alguém com
   acesso sair do escritório, **troque a senha e considere gerar uma nova planilha**.
2. A senha é única e compartilhada — não há usuários separados nem trilha de auditoria.

### Recomendações

- Restrinja a `GOOGLE_API_KEY` no
  [Google Cloud Console](https://console.cloud.google.com/apis/credentials) para aceitar
  apenas o domínio do GitHub Pages (restrição por *HTTP referrer*) e apenas a
  **Google Sheets API**.
- Se o sigilo dos dados for crítico, o caminho correto é um repositório **privado** com
  GitHub Pages restrito (requer GitHub Pro) ou uma hospedagem com autenticação de
  verdade no servidor.
