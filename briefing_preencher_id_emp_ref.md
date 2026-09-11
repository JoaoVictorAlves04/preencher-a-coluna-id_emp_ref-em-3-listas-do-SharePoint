# Tarefa: preencher a coluna `id_emp_ref` em 3 listas do SharePoint

## Contexto

Site SharePoint Online:
`https://boninengenharialtda.sharepoint.com/sites/BANCOBJMM`

Existe um app do Power Apps que precisa filtrar registros de vistorias elétricas por
empreendimento. Hoje o filtro é feito por prefixo de texto no ID, o que é lento e não
usa índice. A solução é desnormalizar: copiar o código do empreendimento para dentro de
cada lista filha, numa coluna já criada e indexada chamada `id_emp_ref`.

A coluna `id_emp_ref` **já existe** nas três listas filhas, está indexada, e está vazia.
Sua tarefa é preenchê-la.

---

## As listas

| Lista (nome interno) | Registros | Papel |
|---|---|---|
| `db_acomp_elet_vistoria` | 63 | ORIGEM do valor — **nunca escrever nesta lista** |
| `db_acomp_elet_vistoria_uh` | 236 | preencher |
| `db_acomp_elet_vistoria_uh_comodos` | 1.338 | preencher |
| `db_acomp_elet_vistoria_uh_comodos_vistoria` | 3.045 | preencher |

---

## As colunas envolvidas

Todas são colunas de **texto**. Não há lookups. A ligação entre listas é por igualdade
exata de string.

**`db_acomp_elet_vistoria`**
- `id_vistoria_eletrica` — chave da vistoria
- `id_contrato_empreendimento` — **é este o valor a ser propagado**

**`db_acomp_elet_vistoria_uh`**
- `id_vistoria_eletrica_uh` — chave da UH
- `id_vistoria_eletrica` — aponta para a vistoria
- `id_emp_ref` — destino da escrita

**`db_acomp_elet_vistoria_uh_comodos`**
- `id_comodo` — chave do cômodo
- `id_vistoria_eletrica_uh` — aponta para a UH
- `id_emp_ref` — destino da escrita

**`db_acomp_elet_vistoria_uh_comodos_vistoria`**
- `id_comodo` — aponta para o cômodo
- `id_emp_ref` — destino da escrita

---

## Exemplo real dos dados

Vistoria de ID 82 em `db_acomp_elet_vistoria`:
```
id_contrato_empreendimento = "273/2024 - 3"
id_vistoria_eletrica       = "ACOMP-FISC273/2024 - 3-vist-eletrciaD397251-100926"
```

As UHs dessa vistoria têm `id_vistoria_eletrica` igual a essa string, e devem receber
`id_emp_ref = "273/2024 - 3"`. Os cômodos dessas UHs e os itens desses cômodos recebem o
mesmo valor.

**Atenção aos valores:** contêm barra (`/`), espaços e hífens — `"273/2024 - 3"`. Os IDs
contêm a string `vist-eletrcia` (com erro de digitação, escrito assim mesmo no banco).
Não normalize, não faça trim, não corrija nada. A comparação deve ser por igualdade exata
da string como ela está gravada.

---

## O que fazer, em 3 blocos nesta ordem

A ordem é obrigatória: cada bloco depende do anterior estar concluído.

**Bloco 1 — UHs**
Para cada registro de `db_acomp_elet_vistoria_uh`: achar em `db_acomp_elet_vistoria` o
registro cujo `id_vistoria_eletrica` seja igual ao `id_vistoria_eletrica` da UH.
Gravar `id_emp_ref` = `id_contrato_empreendimento` desse registro.

**Bloco 2 — cômodos**
Para cada registro de `db_acomp_elet_vistoria_uh_comodos`: achar em
`db_acomp_elet_vistoria_uh` o registro cujo `id_vistoria_eletrica_uh` seja igual ao
`id_vistoria_eletrica_uh` do cômodo.
Gravar `id_emp_ref` = `id_emp_ref` da UH.

**Bloco 3 — itens**
Para cada registro de `db_acomp_elet_vistoria_uh_comodos_vistoria`: achar em
`db_acomp_elet_vistoria_uh_comodos` o registro cujo `id_comodo` seja igual ao `id_comodo`
do item.
Gravar `id_emp_ref` = `id_emp_ref` do cômodo.

---

## Como implementar

**Carregue tudo em memória antes de escrever.** São 4.682 registros no total. Leia cada
lista inteira uma vez, monte um dicionário `{chave: id_emp_ref}` e resolva em memória.
Não faça uma consulta por registro dentro do laço.

Por bloco:
1. GET da lista pai, com `$select` apenas das duas colunas necessárias
2. montar o dicionário
3. GET da lista filha, com `$select` da chave de ligação, do `Id` e do `id_emp_ref`
4. resolver em memória
5. PATCH apenas nos registros cujo valor atual seja diferente do calculado

**Idempotência.** Pular registros que já estão com o valor correto. Isso permite
reexecutar depois de uma falha sem refazer o trabalho todo.

---

## Requisitos técnicos

**Autenticação.** SharePoint Online não aceita usuário e senha via API. Use um dos dois:
- app registrado no Entra ID com permissão de aplicativo `Sites.ReadWrite.All` e
  consentimento de administrador (client credentials); ou
- fluxo de device code com uma conta que tenha permissão de edição nas listas.

Se nenhuma credencial estiver disponível no ambiente, **pare e avise** — não tente
contornar.

**Paginação.** A API devolve 100 itens por página por padrão. Siga o `odata.nextLink` até
o fim. Ignorar isso faz o script processar só parte dos registros **sem sinalizar erro** —
é a falha mais perigosa aqui. Use `$top=5000` para reduzir o número de páginas.

**Throttling.** Com milhares de escritas, respostas HTTP 429 e 503 vão ocorrer. Respeite o
cabeçalho `Retry-After` quando presente e aplique backoff exponencial quando ausente.
Não use paralelismo agressivo: no máximo 4 conexões simultâneas. Rajadas grandes são
punidas e ficam mais lentas no total.

**Escrita.** PATCH (ou POST com `X-HTTP-Method: MERGE` e `IF-MATCH: *`) atualizando
**somente** o campo `id_emp_ref`. Payload mínimo. Não toque em nenhuma outra coluna.

**Nome interno das colunas.** Confirme antes de escrever, via
`_api/web/lists/getbytitle('...')/fields`. O nome de exibição pode diferir do nome
interno. Se `id_emp_ref` não existir com esse nome exato em alguma lista, pare e avise.

---

## Validação

**1. Rode primeiro em modo simulação**, sem escrever, gerando um CSV por bloco com as
colunas: `Id, chave_de_ligacao, id_emp_ref_atual, id_emp_ref_calculado, acao`.
Mostre um resumo antes de pedir autorização para escrever.

**2. Comece pelo Bloco 1** (236 registros). Só avance para o Bloco 2 depois de confirmar
que as 236 UHs foram preenchidas corretamente.

**3. Órfãos: liste, não grave em branco.** Registro cuja chave não tem correspondente no
nível acima deve ir para um relatório de inconsistências, nunca receber string vazia.
É esperado encontrar alguns.

**4. Conferência específica.** Na `db_acomp_elet_vistoria_uh_comodos` existem
**121 registros** cujo `id_comodo` segue o padrão `ACOMP-CLONE-<guid>`, fora do padrão dos
demais. Eles são o motivo principal desta carga. Eles se ligam à UH normalmente pelo
`id_vistoria_eletrica_uh`, que está correto — é só o `id_comodo` que foge do padrão.
Ao final do Bloco 2, confirme explicitamente que os 121 receberam `id_emp_ref`.

**5. Relatório final por bloco:** total de registros, quantos foram atualizados, quantos
já estavam corretos, quantos ficaram órfãos, e a lista de valores distintos gravados com
a contagem de cada um (devem ser poucos valores, repetidos muitas vezes — é o esperado).

---

## O que NÃO fazer

- Não escrever em `db_acomp_elet_vistoria`
- Não alterar nenhuma coluna além de `id_emp_ref`
- Não criar, excluir ou renomear colunas
- Não corrigir o typo `vist-eletrcia` nos IDs — o app depende dele
- Não normalizar, limpar ou fazer trim dos valores de chave
- Não gravar string vazia em nenhum caso
