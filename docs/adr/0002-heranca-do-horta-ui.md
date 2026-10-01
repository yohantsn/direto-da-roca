# ADR 0002 — O que herdamos do repositório `yohantsn/direto_da_roca`

**Status:** decidido · **Data:** 2026-09-30 · **Decisão:** herança do design system

## Contexto

O repositório `yohantsn/direto_da_roca` (criado em jan/2026, último push em mai/2026) tem
o mesmo nome deste projeto, mas **outro escopo**: o README fala em conectar pequenos
produtores familiares a áreas urbanas, com cidade específica (Parasuráh/PR) e ambição
nacional. O [INTENT](../../INTENT.md) é sobre excedente de quintal — pé de laranja,
jabuticaba, limão — conectando vizinhos do mesmo bairro. São dois produtos com o mesmo nome.

Ele contém trabalho real que seria desperdício refazer:

| Pacote | Linhas | Conteúdo |
| --- | --- | --- |
| `core_sdk/horta_ui` | 655 | tema, cores, tipografia, cards, chips, filtros, input, grid, scaffold, app bar |
| `core_sdk/common_models` | ~120 | `Product`, `Producer`, `Location`, `Coordinates` |
| `lib/main.dart` | 22 | renderiza um `Center()` vazio — sem lógica de produto |

## Decisão

**Herdar os valores visuais do `horta_ui`. Descartar o código dos dois pacotes.**

O novo repositório nasce com o [ARCHITECTURE.md](../ARCHITECTURE.md) da A0 e reutiliza
a paleta e a escolha tipográfica. Os widgets são reescritos dentro das convenções da A0,
não migrados.

## Por que não migrar o código

Inspecionados os 22 arquivos, quatro razões, em ordem de peso:

1. **`google_fonts` está fora da lista fechada de pacotes.** O `horta_ui` usa
 `GoogleFonts.epilogue` em toda a tipografia. O pacote baixa a fonte em runtime — ou seja,
 uma requisição a terceiro em runtime, num app que manipula telefone e localização e cujo
 orçamento de primeiro conteúdo (B4) é de 4,0 s em 4G. A A0 não menciona fontes remotas em
 nenhum ponto, e a lista do plano não inclui `google_fonts`. A fonte entra **embutida no**
** bundle**, sem requisição.
2. **Os widgets são wrapper, não design.** `HortaPadding` são 62 linhas para um `enum` de
 três tamanhos e uma classe que estende `EdgeInsets`. `HortaColor` são 23. Oito arquivos
 barrel de 1 a 2 linhas. O núcleo real são `HortaProductCard` (94 linhas) e
 `HortaFilterBar` (57) — dois widgets dos treze.
3. **Conflito de convenção.** Tudo é prefixado com `Horta`, e a A0 fixa a regra contrária
 (não prefixar com o tipo: `ListingCard` sim, `ListingCardWidget` não). O código usa
 herança onde caberia composição (`HortaPadding extends EdgeInsets`,
 `HortaColor extends ColorScheme`).
4. **Vocabulário de domínio errado.** `ProductStatus` traz `seasonal`, `inStock`, `lowStock`,
 `outOfStock` — linguagem de estoque e preço, que o INTENT §3 exclui da v1. O card está
 modelado para vender, e este projeto anuncia doar.

O `common_models` é descartado inteiro: `Product`, `Producer`, `Location` e `Coordinates` não
correspondem ao `Listing` do INTENT §8, que tem `kind` (doação/venda/troca), `quantity_text`,
`public_area` e expiração.

## O que se herda

- **A paleta:** `0xFF4E342E` (primary), `0xFF386641` (secondary), `0xFFF2E8CF` (surface),
`0xFFBC4749` (tertiary). Marrom, verde e creme — adequado ao tema de produto fresco e
local. Verificado por contraste contra os requisitos do A8.
- **A escolha tipográfica:** Epilogue, **embutida no bundle** em vez de `google_fonts`.

São duas decisões, não 775 linhas. A herança de valor é quase toda a herança que existe.

## O que fica para trás

`aiuri_sdk` e `iod`. O `main.dart` antigo depende de ambos: `aiuri_sdk` vem de um `git url`
de outro repositório e parece substituir o Supabase, e `iod` é um pacote de injeção de
dependência. O INTENT §6 decide o oposto: `supabase_flutter` como backend e
`RepositoryProvider`/`BlocProvider` do `flutter_bloc` **sem pacote extra de DI**. Adotá-los
trocaria a decisão de backend inteira e traria dependência fora da lista fechada.

O repositório antigo **fica intocado**. Nada é apagado; apenas deixa de ser a base.

## Detalhe útil

O `pubspec.yaml` do repositório antigo exige `flutter: 3.41.9` e `flutter_lints: ^6.0.0` —
a mesma versão de Flutter instalada aqui, e o mesmo `flutter_lints` que a A0 escolheu de forma
independente. Indica que o ambiente local foi montado a partir daquele repositório, e não o
contrário. O histórico (7 commits num único dia, os últimos três.Utc Trying channel stable → Adding a fixed flutter version`) confirma que parou na configuração do SDK.

## Consequências

- **T-2.2 (tema e componentes base)** escreve os widgets com a paleta herdada, tipografia
embutida, e as convenções da A0 — não importa `horta_ui`.
- **A5** deve registrar, ao especificar o `ImageProcessor`, que o budget B1 (≤ 2,6 MB) é o
que força o import condicional de `package:image`.
- **Nenhum pacote do repositório antigo entra na lista fechada.** Se algo dele for
realmente necessário, passa por aprovação de A.

## Relacionado

[ADR 0001](./0001-free-tier-supabase.md) · [ARCHITECTURE.md](../ARCHITECTURE.md) ·
[INTENT.md](../../INTENT.md)
