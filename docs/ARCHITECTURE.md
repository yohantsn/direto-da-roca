# Arquitetura e convenções

Documento normativo do projeto. Onde este texto e o código divergirem, o código está errado
— ou este documento está desatualizado, o que é um bug do mesmo tamanho.

> **Nota de verificação (2026-09-30).** Os blocos Dart deste documento foram extraídos para
> arquivos reais e compilados com `dart analyze` em Dart 3.11.5. Isso encontrou **cinco bugs que
> quebram a compilação** e que um leitor jamais encontraria lendo: o map pattern
> `{ValidationCode.required}`, que não aceita constante de enum como chave; o
> `const Stream<E>.empty()`, que não aceita type parameter; o `late StreamController` do
> `restartable`, cujo `onDone` fecha o controller antes de qualquer evento; o `this.cause` do
> `DataError`, que redeclara um campo que já existe no pai; e imports faltando em `debounce`,
> `sequential` e `failure_messages`. Todos corrigidos aqui. `lib/core` compila limpo.
>
> Duas entidades continuam **não definidas** de propósito, e cada task as escreve quando chegar:
> `ListingFilter` e `ListingDraft` (exemplo do repositório) e `ListingDto` (mapper). Estão
> importadas pelo exemplo mas não têm corpo aqui — é o que T-4.1 e A4 produzem.

## Fontes que este documento não pode contradizer

| Fonte | O que vem de lá |
| --- | --- |
| [`INTENT.md`](../INTENT.md) §6 e §7 | arquitetura, DI, modelos, i18n, sem geração de código |
| [Plano revisado](../plano-revisado) §1 e §1.1 | papéis dos agentes, definição de pronto e **regras de função e de privilégio** |
| **D1** | contato = Edge Function (Turnstile + rate limit) **+** função SQL `security definer`; sem `service_role` no projeto |
| **D2** | pgTAP entra no CI (`supabase test db`) |
| **D3** | Supabase **free** nos dois projetos; egress de 5 GB é o limite que estoura primeiro; 300 KB por foto |
| §1.1 do plano | 7 itens obrigatórios de **toda** função e view nova — o revisor checa um por um |

O que este documento resolve sozinho: estrutura de pastas, regra de dependência entre camadas,
o contrato de erro (`Failure`), o padrão de repositório, o padrão de BLoC, orquestração
assíncrona, nomenclatura, testes, política de Git e **os budgets de performance** que os gates
`S4`, `S5` e `S8` vão medir. Sem número, esses gates não têm como aprovar nada.

O que este documento **não** resolve, porque é de outra spec: lista de categorias e prazo padrão
(A1), tema e `Router API` (A2), contrato de Auth (A3), validações de anúncio e estratégia de
imagem (A5), estratégia de denúncia (A6), CSP e deploy (A7), acessibilidade (A8), tiles e raio
(A9). Onde aparece "definido em A*", o próximo `A#` escreve.

---

## 1. Estrutura de pastas

Feature-first, conforme `INTENT.md` §7. A árvore abaixo é o **estado final** do projeto, não o
estado de T-0.1.

```
direto_da_roca/
├── analysis_options.yaml          # conjunto de lints (§7.4)
├── budgets.yaml                   # os números de §10, versionados
├── l10n.yaml                      # pipeline ARB, saída em lib/l10n/gen
├── pubspec.yaml                   # environment.sdk: ^3.9.0 (§7.4)
├── README.md
│
├── config/                        # config/*.example.json versionados; reais no .gitignore
│   ├── dev.example.json
│   └── prod.example.json
│
├── docs/
│   ├── INTENT.md
│   ├── ARCHITECTURE.md            # este arquivo
│   ├── BACKEND.md                 # A1
│   ├── SETUP.md                   # A3, T-3.4
│   ├── quality.md                 # número de cobertura do último release (§8.5)
│   └── adr/                       # uma decisão nova = um ADR curto
│
├── lib/
│   ├── main.dart                  # só chama runApp; nenhuma lógica (§11.1)
│   │
│   ├── app/                       # [README.md] composição, tema, router, providers
│   │   ├── bootstrap.dart
│   │   ├── providers.dart
│   │   ├── router/
│   │   │   ├── app_route_path.dart
│   │   │   ├── route_parser.dart
│   │   │   └── router_delegate.dart
│   │   ├── theme.dart
│   │   └── frame_recorder.dart    # addTimingsCallback, só em profile/debug (§10.5)
│   │
│   ├── core/                      # [README.md] nada aqui conhece feature
│   │   ├── error/
│   │   │   ├── failure.dart       # sealed + 13 subtypes, todos no mesmo arquivo (§3.2)
│   │   │   ├── failure_kind.dart  # enums (podem ficar em outro arquivo)
│   │   │   ├── result.dart
│   │   │   └── data_exception.dart
│   │   ├── async/
│   │   │   ├── debounce.dart
│   │   │   ├── droppable.dart
│   │   │   ├── restartable.dart
│   │   │   └── stream_extensions.dart
│   │   ├── config/app_config.dart # ⚠ parte de core que conhece Flutter (§2.4)
│   │   ├── constants/
│   │   │   ├── budgets.dart       # números de §10 em código, espelham budgets.yaml
│   │   │   ├── pagination.dart
│   │   │   └── spacing.dart
│   │   ├── l10n/failure_messages.dart
│   │   ├── logging/app_logger.dart
│   │   └── util/
│   │       ├── wa_link.dart
│   │       ├── phone.dart
│   │       └── outcome.dart
│   │
│   ├── features/
│   │   ├── auth/                  # [README.md]
│   │   │   ├── data/{datasources,models,repositories}/
│   │   │   ├── domain/{entities,repositories,usecases}/
│   │   │   └── presentation/{blocs,pages,widgets}/
│   │   ├── listings/              # [README.md]  ← o grosso do produto
│   │   │   ├── data/
│   │   │   │   ├── datasources/
│   │   │   │   ├── mappers/
│   │   │   │   ├── models/
│   │   │   │   ├── image/
│   │   │   │   │   ├── image_processor.dart        # conditional export (§10.1)
│   │   │   │   │   ├── image_processor_io.dart     # package:image + Isolate.run
│   │   │   │   │   └── image_processor_web.dart    # canvas do browser
│   │   │   │   └── repositories/
│   │   │   ├── domain/{entities,repositories}/
│   │   │   └── presentation/{blocs,pages,widgets}/
│   │   ├── reports/               # [README.md]  ← Fase 6
│   │   │   ├── data/  domain/  presentation/
│   │   └── map/                   # [README.md]  ← Fase 9
│   │       ├── data/  domain/  presentation/
│   │
│   └── l10n/
│       ├── app_pt.arb
│       ├── app_en.arb
│       └── gen/                   # saída do flutter gen-l10n (T-0.1)
│
├── supabase/
│   ├── migrations/                # versionadas; aplicadas primeiro em dev
│   ├── functions/                 # Deno; uma pasta por Edge Function
│   ├── tests/                     # pgTAP (D2): read_anon.sql, write_owner.sql, …
│   └── seed.sql                   # T-1.7, com amplitude para medir scroll
│
├── test/                          # espelha lib/ (§8.1)
│   ├── app/
│   ├── core/
│   ├── features/{auth,listings,reports,map}/
│   └── helpers/                   # mocks, fixtures, registerFallbackValue
│
└── tool/
    ├── check_budgets.dart         # §10 — o CI roda isto
    └── check_coverage.dart        # §8.5
```

### 1.1 Como o repositório lida com pasta vazia

**Decisão: cada pasta de feature ganha um `README.md`. Nenhum `.gitkeep` em pasta de código
Dart. Subpasta sem arquivo não é criada.**

O Git rastreia arquivos, não diretórios: alguma coisa precisa existir para a pasta aparecer. As
três opções, e por que as outras duas perdem:

| Opção | Veredito |
| --- | --- |
| `.gitkeep` em toda pasta vazia | **Perde.** São ~30 arquivos que ninguém lembra de apagar depois que a pasta ganhou código, e eles mentem sobre a maturidade do projeto: o repositório mostra uma arquitetura completa quando nada existe ainda. |
| Um `README.md` por feature | **Vence.** Custa 5 arquivos (`core`, `auth`, `listings`, `reports`, `map`), cada um documentando o contrato da pasta, e é removido no PR que traz o primeiro código. |
| Não criar nada | **Verdadeiro e inútil aqui.** A árvore de pastas *é* informação: é a resposta para "onde vai o repositório?", e o revisor precisa dela antes do primeiro arquivo para não adivinhar. |

Regras:

1. `lib/core/README.md` e `lib/features/<feature>/README.md` são criados em T-0.1, com o
   conteúdo da pasta: o que pode entrar, o que não pode, quem consome.
2. Subpastas (`data/`, `domain/`, `presentation/`, `blocs/`, …) **não** são criadas vazias. A
   subpasta nasce com o primeiro arquivo que vai dentro dela. Vale para `test/` também.
3. O arquivo de placeholder é **removido no mesmo PR** que adiciona o primeiro `.dart` da pasta.
4. `.gitkeep` é aceito **apenas** onde um README não faz sentido: `assets/` com subpastas
   provisionadas, diretórios de fixture. Nunca ao lado de um README, nunca em pasta de código.
5. `map/` nasce vazia e continua vazia até a Fase 9. O README dela diz isso, para ninguém achar
   que o mapa está em progresso.

---

## 2. Camadas e a regra de dependência

### 2.1 A regra

```
                    ┌──────────┐
                    │   app/   │  composição: monta config, cliente, repositórios
                    └────┬─────┘
                         │ só conhece as implementações
              ┌──────────┴──────────┐
              ▼                     ▼
      ┌───────────────┐     ┌───────────────┐
      │ presentation  │     │     data      │
      │ blocs, pages, │     │ Supabase, I/O │
      │ widgets       │     │               │
      └───────┬───────┘     └───────┬───────┘
              │                     │
              ▼                     ▼
         ┌───────────────────────────────┐
         │            domain             │  entidades e contratos, nada de framework
         └───────────────────────────────┘

                    ┌──────────┐
                    │  core/   │  importado pelos três; não conhece feature
                    └──────────┘
```

**`presentation → domain ← data`, e `core` fica abaixo dos três.**

Três regras, cada uma com a razão que a sustenta:

1. **`presentation` conhece interfaces de `domain`, nunca implementações de `data`.** Um bloc
   recebe `ListingsRepository`, nunca `ListingsRepositoryImpl`, nunca `SupabaseClient`. É o que
   permite `mocktail` no `bloc_test` e o que impede um widget de disparar uma query.
2. **`domain` não importa Flutter, nem `supabase_flutter`, nem `dart:io`.** É Dart puro:
   entidades, contratos, tipos de erro. Consequência prática: um teste de `domain` roda sem
   `TestWidgetsFlutterBinding`, sem engine, em milissegundos. Se `domain` precisar de Flutter
   para existir, algo de UI entrou no lugar errado.
3. **`data` conhece Supabase, `image`, `url_launcher`, `geolocator`** — e só `data`. É a camada
   que fala com o mundo externo, então é a única que muda quando o transporte muda.

### 2.2 Por que isso dá testabilidade de verdade

O caso que decide é o teste de bloc. Com a regra acima, `ListingsBlocTest` roda sem banco, sem
rede e sem HTTP:

```dart
class MockListingsRepository extends Mock implements ListingsRepository {}
```

`ListingsRepository` é `abstract interface class` — não `final`, não `sealed` — então o mocktail
consegue implementá-la. Nenhum outro tipo do projeto precisa ser mockado: **entidade de domínio
se constrói de verdade, `Failure` se compara por valor, evento se constrói com `const`**. Se
algum dia um teste precisar mockar uma entidade, a entidade foi declarada `final` à toa (§8.4).

### 2.3 Onde a injeção acontece

`RepositoryProvider` e `BlocProvider`, sem pacote extra (`INTENT.md` §6.4). O **composition
root** é `lib/app/providers.dart`, e é o **único** lugar do projeto que instancia implementação
de repositório e lê `AppConfig`:

```dart
// lib/app/providers.dart — trecho
RepositoryProvider<ListingsRepository>(
  create: (context) => ListingsRepositoryImpl(
    dataSource: SupabaseListingsDataSource(client: context.read<SupabaseClient>()),
    pageSize: listingsPageSize,
  ),
)
```

`data` recebe o `SupabaseClient` pronto, por construtor. `data` **não** lê `AppConfig`, não lê
`String.fromEnvironment`, não sabe em que ambiente está. Isso mantém `core/config` fora do
alcance de `domain` e `data`.

### 2.4 A ressalva de `core`

Duas partes de `core` **sabem** que o projeto é Flutter, e por isso só podem ser importadas por
`app/` e `presentation/`:

| Pasta | Sabe de Flutter? | Quem pode importar |
| --- | --- | --- |
| `core/error/`, `core/async/`, `core/util/`, `core/constants/` | não | `domain`, `data`, `presentation` |
| `core/config/`, `core/l10n/`, `core/logging/` | sim (`foundation`, `AppLocalizations`, `dart:developer`) | `app/`, `presentation` |

`domain` e `data` **nunca** importam a segunda linha. É verificável com um grep, e o revisor
faz esse grep (§9.3).

---

## 3. `Failure` — o contrato de erro

Este é o contrato mais importante do documento. `data` **nunca** expõe exceção; `domain` nunca
lança; `presentation` nunca vê nada que não seja `Failure`.

### 3.1 `Result<T, Failure>`, não exceção

**Padrão escolhido: o contrato de `domain` devolve `Result<T, Failure>`.** Exceção só escapa
*dentro* de `data`, entre datasource e repositório.

Isto é uma escolha **contra** a sugestão de "repositório lança exceção tipada interna e a
fronteira traduz", e a razão é uma só: **com `throw`, o chamador pode esquecer o erro.** Na
prática isso vira dois bugs concretos, e ambos estão no nosso fluxo:

```dart
// compila, e o usuário fica olhando um spinner para sempre
on<ListingsStarted>((event, emit) async {
  await repository.fetchPage(filter: filter);   // esqueceu de tratar a falha
  emit(state);
});

// compila, e o teste passa a depender de expect(bloc.onError, ...)
on<ContactRequested>((event, emit) async {
  await repository.requestContact(listingId: event.listingId);
  emit(const ContactIdle());
});
```

Com `Result` selado, nenhum dos dois compila sem tratamento, e o tratamento é um `switch`
exaustivo — a mesma propriedade que o projeto já exige de eventos e estados (`INTENT.md` §7).
E, para o `bloc_test`, a falha deixa de ser um erro de canal paralelo e passa a ser um estado
emitido, que é o que a tela realmente mostra.

Custo aceito: verbosidade no handler, cerca de seis linhas por operação. Ela economiza a
depuração de "a tela travou sem mensagem nenhuma".

### 3.2 O código

`sealed` exige que **todos os subtipos vivam no mesmo arquivo**. Por isso os 13 `Failure` estão
todos em `lib/core/error/failure.dart`. Os `enum` vão em `failure_kind.dart`, que **não**
importa `failure.dart` — se importasse, o ciclo quebraria o `sealed`.

```dart
// lib/core/error/failure.dart
import 'package:equatable/equatable.dart';

import 'failure_kind.dart';

/// Erro de aplicação. Selado de propósito: o `switch` que traduz para l10n é
/// exaustivo por construção, e adicionar um subtipo quebra a compilação de quem
/// esqueceu de tratá-lo. É esse o ponto do `sealed`.
sealed class Failure extends Equatable {
  const Failure();

  /// "Vale a pena mostrar o botão 'Tentar de novo' agora?"
  /// Temporal (`RateLimited`) é false; sem rede é true. A UI nunca decide isso
  /// olhando o tipo: ela lê daqui.
  bool get isRetryable => false;
}

final class NetworkFailure extends Failure {
  const NetworkFailure(this.kind, {this.retryAfter});

  final NetworkFailureKind kind;
  final Duration? retryAfter;

  @override
  bool get isRetryable => true;

  @override
  List<Object?> get props => [kind, retryAfter];
}

final class UnknownFailure extends Failure {
  const UnknownFailure({required this.operation, required this.cause});

  /// Tag sem PII e sem dado do usuário. Ex.: 'listings_page', 'contact_request'.
  final String operation;
  final Object cause;

  @override
  bool get isRetryable => true;

  /// A causa fica **fora** de `props` de propósito: duas falhas desconhecidas de
  /// origens diferentes precisam comparar iguais num `bloc_test`, e a causa não
  /// pode vazar para log, snackbar ou árvore de widget através do estado.
  @override
  List<Object?> get props => [operation];

  @override
  String toString() => 'UnknownFailure(operation: $operation)';
}

final class AuthFailure extends Failure {
  const AuthFailure({required this.reason, this.email});

  final AuthFailureReason reason;

  /// Existe só para a tela de login oferecer "reenviar link". Nunca vai para log:
  /// é PII.
  final String? email;

  @override
  List<Object?> get props => [reason, email];
}

final class UnauthorizedFailure extends Failure {
  const UnauthorizedFailure({required this.action, this.resourceId});

  final UnauthorizedAction action;
  final String? resourceId;

  @override
  List<Object?> get props => [action, resourceId];
}

final class NotFoundFailure extends Failure {
  const NotFoundFailure({required this.resourceType, required this.resourceId});

  final ResourceType resourceType;
  final String resourceId;

  @override
  List<Object?> get props => [resourceType, resourceId];
}

final class RateLimitedFailure extends Failure {
  const RateLimitedFailure({required this.scope, this.retryAfter});

  final RateLimitScope scope;

  /// Vem do corpo uniforme da Edge Function (§3.5). A UI mostra a contagem.
  final Duration? retryAfter;

  /// Não é retryable **agora**: repetir antes de `retryAfter` só gasta limite.
  @override
  List<Object?> get props => [scope, retryAfter];
}

final class QuotaExceededFailure extends Failure {
  const QuotaExceededFailure({required this.limit, required this.current});

  final int limit;
  final int current;

  @override
  List<Object?> get props => [limit, current];
}

final class PermissionDeniedFailure extends Failure {
  const PermissionDeniedFailure({
    required this.permission,
    this.permanentlyDenied = false,
  });

  final AppPermission permission;

  /// true quando o usuário negou em definitivo: aí a ação vai para as
  /// configurações do sistema, não para um segundo pedido de permissão.
  final bool permanentlyDenied;

  @override
  List<Object?> get props => [permission, permanentlyDenied];
}

final class ValidationFailure extends Failure {
  const ValidationFailure({required this.fieldErrors});

  /// Chave = id estável do campo no formulário (`title`, `neighborhood`, …), o
  /// mesmo que o `FormField` usa. Valor = conjunto de códigos, porque um campo
  /// pode falhar por dois motivos ao mesmo tempo.
  final Map<String, Set<ValidationCode>> fieldErrors;

  @override
  List<Object?> get props => [fieldErrors];

  /// Une duas listas de erros preservando todos os campos. `copyWith` aqui só
  /// alcançaria, nunca fundiria — e fundir é a operação que o formulário precisa
  /// quando valida em lote e depois valida um campo.
  ValidationFailure merge(ValidationFailure other) {
    return ValidationFailure(
      fieldErrors: {
        for (final entry in fieldErrors.entries)
          entry.key: {...entry.value, ...?other.fieldErrors[entry.key]},
        for (final entry in other.fieldErrors.entries)
          if (!fieldErrors.containsKey(entry.key)) entry.key: entry.value,
      },
    );
  }
}

final class StorageFailure extends Failure {
  const StorageFailure({required this.reason, this.retryAfter});

  final StorageFailureReason reason;
  final Duration? retryAfter;

  @override
  bool get isRetryable => switch (reason) {
    StorageFailureReason.networkDuringUpload => true,
    StorageFailureReason.aborted => true,
    _ => false,
  };

  @override
  List<Object?> get props => [reason, retryAfter];
}

final class ImageProcessingFailure extends Failure {
  const ImageProcessingFailure({
    required this.reason,
    this.sourceWidth,
    this.sourceHeight,
  });

  final ImageProcessingFailureReason reason;

  /// Dimensões do **original**. A tela precisa delas para dizer "sua foto é
  /// 4032×3024, o máximo é 1600×1200" em vez de um "imagem inválida" genérico.
  final int? sourceWidth;
  final int? sourceHeight;

  @override
  List<Object?> get props => [reason, sourceWidth, sourceHeight];
}

final class LaunchFailure extends Failure {
  const LaunchFailure({required this.target});

  final LaunchTarget target;

  @override
  bool get isRetryable => true;

  @override
  List<Object?> get props => [target];
}

final class ContactUnavailableFailure extends Failure {
  const ContactUnavailableFailure({this.retryAfter});

  final Duration? retryAfter;

  @override
  List<Object?> get props => [retryAfter];
}
```

```dart
// lib/core/error/failure_kind.dart
// Arquivo so de enums. NAO importa failure.dart - se importasse, o ciclo
// quebraria o `sealed` de 3.2. Todos os enums que os `Failure` referenciam
// moram aqui.
enum NetworkFailureKind { offline, timeout, dns, unknown }

enum AuthFailureReason {
  invalidCredentials,
  weakPassword,
  emailInUse,
  accountDisabled,
  providerError,
  tokenExpired,
  unknown,
}
/// A ação que o usuário tentou e não podia. Separate do `AuthFailureReason`,
/// que é *por que* o login falhou: aqui a sessão existe e o acesso é que foi
/// negado. Os quatro valores são os que o `switch` exaustivo de §3.5 percorre.
enum UnauthorizedAction {
  /// A row existe, mas pertence a outro `owner_id`.
  notOwner,

  /// O usuário atingiu o teto de anúncios ativos (a regra de T-1.3b).
  activeListingLimit,

  /// A sessão expirou no meio da operação — diferente de
  /// `AuthFailureReason.tokenExpired`, que é o refresh que falhou no login.
  sessionExpired,

  /// O contato não é liberado: consentimento ausente, ou anúncio não ativo.
  /// Deliberadamente genérico; nomear o motivo aqui viraria o oráculo que D1
  /// proíbe (ver §3.5).
  contactNotAllowed,
}

enum ResourceType { listing, report, profile, image }

enum RateLimitScope { perIp, perListing, perUser, otp, storage }

enum AppPermission { location }

enum ValidationCode {
  required,
  tooShort,
  tooLong,
  invalidFormat,
  outOfRange,
  duplicate,
  notAccepted,
  inThePast,
}
enum StorageFailureReason {
  tooLarge,
  wrongType,
  quotaExceeded,
  networkDuringUpload,
  forbidden,
  aborted,
  unknown,
}
enum ImageProcessingFailureReason {
  decodeFailed,
  tooManyPixels,
  unsupportedFormat,
  exifNotRemoved,
}
enum LaunchTarget { whatsapp, externalBrowser }

```

### 3.3 `Result`

```dart
// lib/core/error/result.dart
import 'package:equatable/equatable.dart';

sealed class Result<T, E extends Object> extends Equatable {
  const Result();
}

final class Ok<T, E extends Object> extends Result<T, E> {
  const Ok(this.value);

  final T value;

  @override
  List<Object?> get props => [value];
}

final class Err<T, E extends Object> extends Result<T, E> {
  const Err(this.error);

  final E error;

  @override
  List<Object?> get props => [error];
}

extension ResultX<T, E extends Object> on Result<T, E> {
  bool get isOk => this is Ok<T, E>;
  bool get isErr => this is Err<T, E>;

  R fold<R>({
    required R Function(T value) onOk,
    required R Function(E error) onErr,
  }) => switch (this) {
    Ok<T, E>(:final value) => onOk(value),
    Err<T, E>(:final error) => onErr(error),
  };

  Result<R, E> map<R>(R Function(T value) transform) => switch (this) {
    Ok<T, E>(:final value) => Ok<R, E>(transform(value)),
    Err<T, E>(:final error) => Err<R, E>(error),
  };
}
```

**Escreva os argumentos de tipo explicitamente** ao construir
(`const Ok<ListingsPage, Failure>(page)`). A inferência funciona na maioria dos casos, mas
escrever o tipo deixa o `data` mais legível e evita o `inference_failure_on_untyped_parameter`
quando o literal é o primeiro argumento de uma função que retorna
`Future<Result<T, Failure>>`.

### 3.4 A exceção interna de `data`

`DataException` existe para que a causa técnica — status HTTP, código Postgres,
`PostgrestException` — fique em `data`, e para o log ter o que registrar.

```dart
// lib/core/error/data_exception.dart
import 'failure_kind.dart';

enum DataErrorKind {
  network,
  timeout,
  unauthorized, // 401, 403, ou RLS negando (42501)
  notFound,     // 404, PGRST116
  rateLimited,  // 429
  validation,  // 400, 23514, violação de check
  server,       // 5xx
  storage,
  cancelled,
  unknown,
}

/// Erro de transporte. NÃO atravessa `data`. Convertido em `Failure` pelo mapper
/// do repositório, que é o único lugar que conhece os dois vocabulários.
sealed class DataException implements Exception {
  const DataException({required this.cause});

  final Object cause;
}

final class DataError extends DataException {
  // `super.cause` e não `this.cause`: o pai já declara `cause`, e repetir o
  // parâmetro no construtor do filho dá "initializing formal for non-existent
  // field". `super.cause` é o que inicializa o campo herdado.
  const DataError({
    required this.kind,
    required super.cause,
    this.statusCode,
    this.postgresCode,
    this.retryAfter,
  });

  final DataErrorKind kind;
  final int? statusCode;

  /// Código do Postgres, quando houver: '42501' (permissão), '23505' (duplicata),
  /// '23514' (check). Dá a distinção entre "a policy barrou" e "faltou grant",
  /// que a §1.1 do plano trata como mecanismos de negação diferentes.
  final String? postgresCode;
  final Duration? retryAfter;
}
```

O repositório mapeia com `switch` exaustivo em `DataErrorKind` e **loga** `cause` no ponto da
tradução, nunca depois. A tradução de `data` para `domain` é a fronteira: depois dela, a
informação técnica fica no log e a `Failure` carrega só o que a interface precisa.

### 3.5 Como a interface consome

Um `switch` exaustivo em um lugar só. A apresentação é dona do l10n, e é por isso que a mensagem
**não** mora na `Failure`.

```dart
// lib/core/l10n/failure_messages.dart
import 'dart:math' show max;

import 'package:direto_da_roca/core/error/failure.dart';
import 'package:direto_da_roca/core/error/failure_kind.dart';
import 'package:direto_da_roca/l10n/gen/app_localizations.dart';

extension FailureMessages on Failure {
  String message(AppLocalizations l10n) => switch (this) {
    NetworkFailure(:final kind) => switch (kind) {
      NetworkFailureKind.offline => l10n.failureNetworkOffline,
      _ => l10n.failureNetwork,
    },
    UnknownFailure() => l10n.failureUnknown,
    AuthFailure(:final reason) => switch (reason) {
      AuthFailureReason.invalidCredentials => l10n.failureAuthInvalid,
      AuthFailureReason.weakPassword => l10n.failureAuthWeakPassword,
      AuthFailureReason.emailInUse => l10n.failureAuthEmailInUse,
      AuthFailureReason.accountDisabled => l10n.failureAuthDisabled,
      AuthFailureReason.providerError => l10n.failureAuthProvider,
      AuthFailureReason.tokenExpired => l10n.failureAuthTokenExpired,
      AuthFailureReason.unknown => l10n.failureUnknown,
    },
    UnauthorizedFailure(:final action) => switch (action) {
      UnauthorizedAction.notOwner => l10n.failureNotYourListing,
      UnauthorizedAction.activeListingLimit => l10n.failureListingLimit,
      UnauthorizedAction.sessionExpired => l10n.failureSessionExpired,
      UnauthorizedAction.contactNotAllowed => l10n.failureContactNotAllowed,
    },
    NotFoundFailure(:final resourceType) => switch (resourceType) {
      ResourceType.listing => l10n.failureListingNotFound,
      ResourceType.report => l10n.failureReportNotFound,
      ResourceType.profile => l10n.failureProfileNotFound,
      ResourceType.image => l10n.failureImageNotFound,
    },
    RateLimitedFailure(:final retryAfter) => retryAfter == null
        ? l10n.failureRateLimited
        // `inMinutes` pode dar 0 e a tela mostraria "tente em 0 minutos".
        : l10n.failureRateLimitedIn(max(1, retryAfter.inMinutes)),
    QuotaExceededFailure(:final limit) => l10n.failureListingLimitCount(limit),
    PermissionDeniedFailure(:final permanentlyDenied) => permanentlyDenied
        ? l10n.failureLocationDeniedForever
        : l10n.failureLocationDenied,
    ValidationFailure() => l10n.failureValidation,
    StorageFailure(:final reason) => switch (reason) {
      StorageFailureReason.tooLarge => l10n.failureImageTooLarge,
      StorageFailureReason.wrongType => l10n.failureImageWrongType,
      StorageFailureReason.quotaExceeded => l10n.failureQuota,
      StorageFailureReason.forbidden => l10n.failureUploadForbidden,
      StorageFailureReason.networkDuringUpload => l10n.failureUploadNetwork,
      StorageFailureReason.aborted => l10n.failureUploadAborted,
      StorageFailureReason.unknown => l10n.failureUnknown,
    },
    ImageProcessingFailure(:final reason) => switch (reason) {
      ImageProcessingFailureReason.tooManyPixels => l10n.failureImageTooManyPixels,
      ImageProcessingFailureReason.decodeFailed => l10n.failureImageUnreadable,
      ImageProcessingFailureReason.unsupportedFormat => l10n.failureImageFormat,
      ImageProcessingFailureReason.exifNotRemoved => l10n.failureInternal,
    },
    ContactUnavailableFailure() => l10n.failureContactUnavailable,
    LaunchFailure(:final target) => switch (target) {
      LaunchTarget.whatsapp => l10n.failureWhatsappNotInstalled,
      LaunchTarget.externalBrowser => l10n.failureBrowserNotInstalled,
    },
  };
}
```

Três regras que acompanham:

1. **O botão "Tentar de novo" só aparece quando `failure.isRetryable`.** A decisão é da `Failure`,
   não do `switch` de texto — assim o mesmo botão não fica inconsistente entre telas.
2. **`UnknownFailure` nunca mostra `cause`.** O `toString()` já não expõe; a interface usa a
   mensagem genérica e o log tem o resto.
3. **Erro de validação tem um segundo caminho**, porque um formulário não mostra banner: ele
   marca campos.

```dart
// lib/features/listings/presentation/validation_messages.dart
extension ListingValidationMessages on ValidationFailure {
  String? messageFor(String fieldId, AppLocalizations l10n) {
    final codes = fieldErrors[fieldId];
    if (codes == null) return null;
    return switch (codes) {
      // Guarda por `contains`, e não map pattern: a chave de um map pattern precisa ser
      // uma constante, e `ValidationCode.required` é uma constante de enum — o pattern
      // {ValidationCode.required} não compila. Além disso o campo é `Set`, e Set não
      // tem igualdade por valor: mesmo que compilasse, um `containsKey` nunca casaria.
      _ when codes.contains(ValidationCode.required) &&
              !codes.contains(ValidationCode.notAccepted) =>
        l10n.validationTitleRequired,
      _ when codes.contains(ValidationCode.required) &&
              codes.contains(ValidationCode.notAccepted) =>
        l10n.validationTitleRequired,
      _ when codes.contains(ValidationCode.inThePast) => l10n.validationExpiryInThePast,
      _ when codes.contains(ValidationCode.notAccepted) => l10n.validationConsentRequired,
      _ => l10n.validationInvalid,
    };
  }
}
```

O `switch` sobre `Set` casa o conjunto como padrão: primeiro os casos exatos, depois um `_` de
reserva. É o que impede o campo marcado sem mensagem e o campo com mensagem sem marcação.

---

## 4. Padrão de repositório

### 4.1 O padrão

- **Interface em `domain/repositories/`, implementação em `data/repositories/`.** Mesmo nome de
  arquivo nos dois lados: `listings_repository.dart` nos dois.
- **`abstract interface class`, nunca `final` nem `sealed`** — é o que permite `mocktail`.
- **Fonte de verdade única:** a interface. Nenhum outro arquivo em `data` importa
  `listings_repository_impl.dart`, e nada fora de `data` importa a implementação.
- **Uma interface por agregado**, não por tela. A tela depende do agregado.
- **Conversão manual de JSON** (`INTENT.md` §6.5): DTO em `data/models/`, mapper em
  `data/mappers/`, entidade em `domain/entities/`. Sem `fromJson` na entidade, sem
  `json_serializable`, sem `build_runner`.

### 4.2 `ListingsRepository` — o caso real

É a interface que T-4.1 implementa e que T-4.2 consome.

```dart
// lib/features/listings/domain/repositories/listings_repository.dart
import 'package:equatable/equatable.dart';

import 'package:direto_da_roca/core/error/failure.dart';
import 'package:direto_da_roca/core/error/result.dart';
import 'package:direto_da_roca/features/listings/domain/entities/listing.dart';
import 'package:direto_da_roca/features/listings/domain/entities/listing_draft.dart';
import 'package:direto_da_roca/features/listings/domain/entities/listings_filter.dart';

/// Cursor de paginação opaco. `domain` só precisa saber que ele é opaco: quem
/// codifica e decodifica é `data` (ver `ListingCursorCodec`).
typedef ListingCursor = String;

/// Uma página de anúncios e o cursor da próxima, ou `null` se acabou.
final class ListingsPage extends Equatable {
  const ListingsPage({required this.items, this.nextCursor});

  final List<Listing> items;
  final ListingCursor? nextCursor;

  bool get hasMore => nextCursor != null;

  @override
  List<Object?> get props => [items, nextCursor];
}

abstract interface class ListingsRepository {
  /// Busca filtrada, paginada por cursor (keyset, não `offset` — ver §4.4).
  Future<Result<ListingsPage, Failure>> fetchPage({
    required ListingsFilter filter,
    ListingCursor? cursor,
  });

  Future<Result<Listing, Failure>> fetchById(String listingId);

  /// Os anúncios do usuário logado, incluindo `finished` e `expired`.
  Future<Result<ListingsPage, Failure>> fetchMine({ListingCursor? cursor});

  Future<Result<Listing, Failure>> create(ListingDraft draft);

  Future<Result<Listing, Failure>> update(String listingId, ListingDraft draft);

  /// Marcar como "já acabou". Não é `update`: é transição de status, e o
  /// `expires_at` de um anúncio finalizado não muda.
  Future<Result<Listing, Failure>> markAsFinished(String listingId);

  /// Renovar. O prazo vem da constante única de A1, não do cliente.
  Future<Result<Listing, Failure>> renew(String listingId);

  Future<Result<void, Failure>> delete(String listingId);
}
```

`fetchPage` **não** recebe `pageSize`. O tamanho da página é o orçamento de §10.2, é global, e
expor o parâmetro é o caminho mais curto para alguém pedir 500 linhas num orçamento de 20. A
implementação recebe `pageSize` por construtor, com default `listingsPageSize`, e o teste
injeta outro valor.

### 4.3 `Listing`, o DTO e o mapper

```dart
// lib/features/listings/domain/entities/listing.dart
import 'package:equatable/equatable.dart';

enum ListingKind { donation, sale, swap }
enum ListingStatus { active, finished, expired, removed }

/// Espelha o enum Postgres `listing_category`, definido em A1.
///
/// ⚠ O conjunto de valores abaixo é **provisório** e existe para compilar este
/// exemplo. A lista canônica é de A1 (`docs/BACKEND.md`); quando A1 fechar, o
/// único lugar a mudar é este `enum` e o `switch` do mapper. Mel, queijo e
/// cárneos **não** são categorias: `INTENT.md` §3 é explícito.
enum ListingCategory { fruit, vegetable, leafyGreen, herb, egg, other }

final class Listing extends Equatable {
  const Listing({
    required this.id,
    required this.ownerId,
    required this.title,
    required this.kind,
    required this.category,
    required this.status,
    required this.city,
    required this.neighborhood,
    required this.createdAt,
    required this.expiresAt,
    this.description = '',
    this.quantityText = '',
    this.priceInCents,
    this.coverImagePath,
    this.imageCount = 0,
    this.ownerDisplayName,
  });

  final String id;

  /// Existe para o detalhe decidir entre "Editar" e "Contatar". **Nunca é
  /// exibido** — e o `S1` audita se a view pública o expõe.
  final String ownerId;

  final String title;
  final String description;
  final ListingKind kind;
  final ListingCategory category;

  /// Texto livre ("1 kg", "1 sacola", "12 unidades"), não número: a quantidade
  /// de fruta de quintal não cabe em inteiro, e `INTENT.md` §5 pede
  /// "aproximada".
  final String quantityText;

  /// Centavos, não `double`: dinheiro em ponto flutuante é bug de arredondamento.
  final int? priceInCents;

  final String city;
  final String neighborhood;
  final ListingStatus status;
  final DateTime createdAt;
  final DateTime expiresAt;

  /// Caminho no bucket, não URL. A URL é montada em `data`, que é quem conhece
  /// o bucket e a URL pública do Storage.
  final String? coverImagePath;
  final int imageCount;

  /// O que a view pública expõe é decisão de A1; o `S1` audita.
  final String? ownerDisplayName;

  bool get isActive =>
      status == ListingStatus.active && expiresAt.isAfter(DateTime.now());

  /// Anúncio expirado **não** é erro de rede nem 404: é um anúncio, com
  /// `status == expired`. A tela de detalhe mostra "este anúncio expirou".
  /// Só `NotFoundFailure` é para id que não existe.
  bool get isExpired => status == ListingStatus.expired;

  @override
  List<Object?> get props => [
    id, ownerId, title, description, kind, category, quantityText, priceInCents,
    city, neighborhood, status, createdAt, expiresAt, coverImagePath, imageCount,
    ownerDisplayName,
  ];
}
```

`props` lista **todos** os campos, sempre. Campo novo fora de `props` é o bug mais discreto do
projeto: o `BlocBuilder` compara por igualdade, o `buildWhen` deixa de filtrar, e nenhum teste
falha.

O DTO e o mapper:

```dart
// lib/features/listings/data/models/listing_dto.dart
import 'package:equatable/equatable.dart';

/// Só o que a **view pública** devolve. Se um campo entra aqui e o `S1` disser
/// que não deveria estar exposto, o problema é do DTO, não da view.
final class ListingDto extends Equatable {
  const ListingDto({
    required this.id,
    required this.ownerId,
    required this.title,
    required this.kind,
    required this.category,
    required this.status,
    required this.city,
    required this.neighborhood,
    required this.createdAtIso,
    required this.expiresAtIso,
    this.description = '',
    this.quantityText = '',
    this.priceInCents,
    this.coverImagePath,
    this.imageCount = 0,
    this.ownerDisplayName,
  });

  /// Conversão manual, campo a campo, com default explícito — decisão do
  /// `INTENT.md` §6.5. `json['x']` sem tipo e sem default é onde nasce o
  /// `TypeError` em produção.
  factory ListingDto.fromJson(Map<String, Object?> json) {
    return ListingDto(
      id: json['id']! as String,
      ownerId: json['owner_id']! as String,
      title: json['title']! as String,
      kind: json['kind']! as String,
      category: json['category']! as String,
      status: json['status']! as String,
      city: json['city']! as String,
      neighborhood: json['neighborhood']! as String,
      createdAtIso: json['created_at']! as String,
      expiresAtIso: json['expires_at']! as String,
      description: (json['description'] as String?) ?? '',
      quantityText: (json['quantity_text'] as String?) ?? '',
      priceInCents: (json['price_cents'] as num?)?.toInt(),
      coverImagePath: json['cover_image_path'] as String?,
      imageCount: (json['image_count'] as num?)?.toInt() ?? 0,
      ownerDisplayName: json['owner_display_name'] as String?,
    );
  }

  final String id;
  final String ownerId;
  final String title;
  final String kind;
  final String category;
  final String status;
  final String city;
  final String neighborhood;
  final String createdAtIso;
  final String expiresAtIso;
  final String description;
  final String quantityText;
  final int? priceInCents;
  final String? coverImagePath;
  final int imageCount;
  final String? ownerDisplayName;

  @override
  List<Object?> get props => [id, ownerId, title, kind, category, status, city,
    neighborhood, createdAtIso, expiresAtIso, description, quantityText,
    priceInCents, coverImagePath, imageCount, ownerDisplayName];
}
```

```dart
// lib/features/listings/data/mappers/listing_mapper.dart
import 'package:direto_da_roca/features/listings/data/models/listing_dto.dart';
import 'package:direto_da_roca/features/listings/domain/entities/listing.dart';

/// Função pura em forma de classe: é assim que fica testável sem mock e sem
/// `build_runner`. T-4.1 pede "unitários de mapper"; é este arquivo.
final class ListingMapper {
  const ListingMapper();

  Listing toEntity(ListingDto dto) => Listing(
    id: dto.id,
    ownerId: dto.ownerId,
    title: dto.title,
    description: dto.description,
    kind: kindFromRaw(dto.kind),
    category: categoryFromRaw(dto.category),
    quantityText: dto.quantityText,
    priceInCents: dto.priceInCents,
    city: dto.city,
    neighborhood: dto.neighborhood,
    status: statusFromRaw(dto.status),
    createdAt: DateTime.parse(dto.createdAtIso).toLocal(),
    expiresAt: DateTime.parse(dto.expiresAtIso).toLocal(),
    coverImagePath: dto.coverImagePath,
    imageCount: dto.imageCount,
    ownerDisplayName: dto.ownerDisplayName,
  );

  /// Valor desconhecido **não** some em silêncio: o enum de Postgres pode ganhar
  /// um valor à frente do app, e o anúncio precisa aparecer mesmo assim. O
  /// default é escolhido para ser o caso mais comum na tela, nunca para esconder
  /// um dado corrompido sem log.
  static ListingKind kindFromRaw(String raw) => switch (raw) {
    'donation' => ListingKind.donation,
    'sale' => ListingKind.sale,
    'swap' => ListingKind.swap,
    _ => ListingKind.sale,
  };

  static ListingStatus statusFromRaw(String raw) => switch (raw) {
    'active' => ListingStatus.active,
    'finished' => ListingStatus.finished,
    'expired' => ListingStatus.expired,
    'removed' => ListingStatus.removed,
    _ => ListingStatus.active,
  };

  static ListingCategory categoryFromRaw(String raw) => switch (raw) {
    'fruit' => ListingCategory.fruit,
    'vegetable' => ListingCategory.vegetable,
    'leafy_green' => ListingCategory.leafyGreen,
    'herb' => ListingCategory.herb,
    'egg' => ListingCategory.egg,
    _ => ListingCategory.other,
  };
}
```

### 4.4 Por que cursor, e não `offset`

`offset` quebra com concorrência, e este produto tem concorrência: enquanto alguém rola a lista,
outro vizinho publica um anúncio. Com `offset`, o anúncio novo empurra todo mundo uma posição, e
o usuário vê duplicata e pula um — indefinidamente, quanto mais longa a lista.

Keyset ordena por `(created_at desc, id desc)`. O `id` desempata linhas com o mesmo timestamp,
que é o caso comum quando duas pessoas publicam no mesmo segundo. O cursor carrega o último par
lido; `data` pede `pageSize + 1` linhas, usa a linha extra só para saber se há mais, e codifica
o cursor em base64url de `"<rfc3339>|<uuid>"`. Ninguém fora de `data` decodifica esse formato.

---

## 5. Padrão de BLoC

### 5.1 Eventos e estados são `sealed`, sempre

- `sealed class XEvent extends Equatable`, com subtipos `final class` no mesmo arquivo.
- `final class` em eventos e entidades. `MockListing` não é possível (§2.2) e não é preciso.
- **`Equatable` em tudo que entra no estado.** `BlocBuilder` compara por igualdade; sem isso,
  qualquer `emit` reconstrói a lista inteira.
- `exhaustive_cases` é erro de compilação no analyzer: um subtipo novo quebra todo `switch` que
  não o tratar. É o mecanismo de segurança do projeto sobre o estado.

### 5.2 Estados selados ou estado único com campos?

**Decisão: os dois padrões existem, com um critério objetivo.**

| Situação | Padrão |
| --- | --- |
| A tela **não mostra dados** do fluxo: máquina de estados pura, poucos estados, transição simples | `sealed` com subtipos |
| A tela **mostra dados**: lista, formulário, detalhe com galeria | **uma classe de estado com campos + um `enum status`** |

O estado único é o padrão quando há dados, por três razões concretas: uma lista com
carregando, erro ao carregar, carregando mais, erro ao carregar mais, pronto, vazio e
recarregando daria sete subclasses carregando os mesmos seis campos; `copyWith` vira uma
operação incorreta (`copyWith(failure: null)` não limpa o campo, e quem lembra disso só na
revisão); e a combinação impossível fica representável, sem nada que impeça `isLoading` e
`hasLoaded` ao mesmo tempo.

A mitigação, que é o que torna isso seguro:

1. um `enum status` como discriminador, para o `buildWhen` ser barato e o `switch` ser exaustivo;
2. **só fábricas nomeadas**, com `assert` impedindo combinação impossível;
3. **sem `copyWith`** nesse estado. Fábrica é mais verbosa e não tem o bug do `null`.

**Erro nunca é uma classe de estado isolada.** Um `ErrorState` que só carrega erro é proibido:
obriga a interface a lembrar de voltar para um estado bom e perde o contexto — a lista continua
carregada, mas sumiu da tela. No padrão selado, a classe de erro **carrega o contexto junto**
(`ContactFailed` sabe qual anúncio falhou). No estado único, é o campo `failure`.

### 5.3 Exemplo A — padrão selado: `ContactBloc`

Dois eventos, três estados, `switch` exaustivo.

```dart
// lib/features/listings/presentation/blocs/contact/contact_event.dart
import 'package:equatable/equatable.dart';

sealed class ContactEvent extends Equatable {
  const ContactEvent();

  @override
  List<Object?> get props => const [];
}

final class ContactRequested extends ContactEvent {
  const ContactRequested({
    required this.listingId,
    required this.turnstileToken,
    required this.prefilledMessage,
  });

  final String listingId;

  /// Token do Turnstile já obtido. Quem monta o widget do Turnstile é a tela, no
  /// `build` do detalhe: se o token fosse buscado no toque, o toque pagaria
  /// +1 RTT só para isso (§10.4).
  final String turnstileToken;

  /// Texto já montado e escapado, feito pela `presentation`, que é dona do l10n.
  /// O bloc não conhece texto de interface.
  final String prefilledMessage;

  @override
  List<Object?> get props => [listingId, turnstileToken, prefilledMessage];
}

final class ContactDismissed extends ContactEvent {
  const ContactDismissed();
}
```

```dart
// lib/features/listings/presentation/blocs/contact/contact_state.dart
import 'package:equatable/equatable.dart';

import 'package:direto_da_roca/core/error/failure.dart';

sealed class ContactState extends Equatable {
  const ContactState();
}

/// Estado inicial e também estado de sucesso: abrir o WhatsApp tira o usuário da
/// tela, então não há "sucesso" a renderizar. Três estados cobrem os três desfechos
/// que a interface sabe mostrar.
final class ContactIdle extends ContactState {
  const ContactIdle();

  @override
  List<Object?> get props => const [];
}

final class ContactLoading extends ContactState {
  const ContactLoading({required this.listingId});

  /// A interface desabilita o botão **deste** anúncio. Sem isso, dois anúncios
  /// na tela e o spinner aparece no errado.
  final String listingId;

  @override
  List<Object?> get props => [listingId];
}

final class ContactFailed extends ContactState {
  const ContactFailed({required this.listingId, required this.failure});

  final String listingId;
  final Failure failure;

  @override
  List<Object?> get props => [listingId, failure];
}
```

```dart
// lib/features/listings/domain/entities/contact_details.dart
import 'package:equatable/equatable.dart';

/// O que a Edge Function devolve depois de validar o Turnstile e o rate limit.
///
/// Não carrega nada além do telefone e do nome do vendedor: é o retorno da
/// função `private.get_listing_contact`, que já aplicou o consentimento de
/// exibição. Nada de `ownerId`, `location` ou `expiresAt` chega aqui.
final class ContactDetails extends Equatable {
  const ContactDetails({required this.phoneNumber, this.sellerDisplayName});

  /// Em E.164, só dígitos, pronto para o `wa.me`. A normalização acontece na
  /// função SQL, não no cliente.
  final String phoneNumber;
  final String? sellerDisplayName;

  @override
  List<Object?> get props => [phoneNumber, sellerDisplayName];
}
```

```dart
// lib/features/listings/domain/repositories/contact_repository.dart
import 'package:direto_da_roca/core/error/failure.dart';
import 'package:direto_da_roca/core/error/result.dart';
import 'package:direto_da_roca/features/listings/domain/entities/contact_details.dart';

abstract interface class ContactRepository {
  /// Chama a Edge Function de T-1.2b. Devolve `Err` com
  /// `ContactUnavailableFailure` **tanto** para limite estourado quanto para
  /// anúncio inexistente — a resposta é uniforme de propósito (D1); distinguish
  /// os dois viraria oráculo de quais anúncios existem.
  Future<Result<ContactDetails, Failure>> requestContact({
    required String listingId,
    required String turnstileToken,
  });
}
```

```dart
// lib/features/listings/presentation/blocs/contact/contact_bloc.dart
import 'package:bloc/bloc.dart';

import 'package:direto_da_roca/core/async/droppable.dart';
import 'package:direto_da_roca/core/error/failure.dart';
import 'package:direto_da_roca/core/error/result.dart';
import 'package:direto_da_roca/core/util/wa_link.dart';
import 'package:direto_da_roca/features/listings/domain/entities/contact_details.dart';
import 'package:direto_da_roca/features/listings/domain/repositories/contact_repository.dart';
import 'package:direto_da_roca/features/listings/domain/repositories/whatsapp_launcher.dart';

import 'contact_event.dart';
import 'contact_state.dart';

class ContactBloc extends Bloc<ContactEvent, ContactState> {
  ContactBloc({
    required ContactRepository repository,
    required WhatsAppLauncher launcher,
  })  : _repository = repository,
        _launcher = launcher,
        super(const ContactIdle()) {
    // `droppable`: dois toques no mesmo botão não podem virar duas requisições.
    // O limite da D1 é precious e um duplo toque é o jeito mais fácil de gastar
    // a cota de todo mundo que está atrás do mesmo CGNAT.
    on<ContactRequested>(_onRequested, transformer: droppable());
    on<ContactDismissed>((event, emit) => emit(const ContactIdle()));
  }

  final ContactRepository _repository;
  final WhatsAppLauncher _launcher;

  Future<void> _onRequested(
    ContactRequested event,
    Emitter<ContactState> emit,
  ) async {
    emit(ContactLoading(listingId: event.listingId));

    final Result<ContactDetails, Failure> result =
        await _repository.requestContact(
          listingId: event.listingId,
          turnstileToken: event.turnstileToken,
        );

    // `droppable` já impede o segundo evento de entrar, mas o `await` acima
    // continua válido: o guard cobre o caso em que a resposta chegar depois de um
    // `close()` ou de um `emit` de outro caminho. `emit.isDone` é a verificação
    // barata que evita o `StateError` do bloc.
    if (emit.isDone) return;

    switch (result) {
      case Ok<ContactDetails, Failure>(:final value):
        final String url = WhatsAppLink.build(
          phoneNumber: value.phoneNumber,
          message: event.prefilledMessage,
        );
        final bool launched = await _launcher.launch(url);
        emit(
          launched
              ? const ContactIdle()
              : ContactFailed(
                  listingId: event.listingId,
                  failure: const LaunchFailure(target: LaunchTarget.whatsapp),
                ),
        );
      case Err<ContactDetails, Failure>(:final error):
        emit(ContactFailed(listingId: event.listingId, failure: error));
    }
  }
}
```

E o `switch` exaustivo da interface, que é o que o compilador obriga a escrever quando um
subtipo de `ContactState` novo aparece:

```dart
// lib/features/listings/presentation/pages/listing_detail_page.dart — trecho
BlocBuilder<ContactBloc, ContactState>(
  builder: (context, state) {
    return switch (state) {
      ContactIdle() => ContactButton(
        label: l10n.contactSeller,
        onPressed: () => context.read<ContactBloc>().add(requested),
      ),
      ContactLoading(:final listingId) => ContactButton.spinner(
        key: ValueKey('contact-spinner-$listingId'),
      ),
      ContactFailed(:final listingId, :final failure) => ContactError(
        listingId: listingId,
        message: failure.message(l10n),
        showRetry: failure.isRetryable,
        onRetry: () => context.read<ContactBloc>().add(requested),
      ),
    };
  },
)
```

### 5.4 Exemplo B — estado único: `ListingsState`

Três eventos, um `enum status`, `failure` como campo, e **sem `copyWith`**.

```dart
// lib/features/listings/presentation/blocs/listings/listings_event.dart
import 'package:equatable/equatable.dart';

import 'package:direto_da_roca/features/listings/domain/entities/listings_filter.dart';

sealed class ListingsEvent extends Equatable {
  const ListingsEvent();

  @override
  List<Object?> get props => const [];
}

/// Primeira carga, ou recarga explícita (pull-to-refresh). Differente de
/// `ListingsFilterChanged`, que **não** recarrega do zero.
final class ListingsStarted extends ListingsEvent {
  const ListingsStarted();

  @override
  List<Object?> get props => const [];
}

final class ListingsFilterChanged extends ListingsEvent {
  const ListingsFilterChanged(this.filter);

  final ListingsFilter filter;

  @override
  List<Object?> get props => [filter];
}

final class ListingsNextPageRequested extends ListingsEvent {
  const ListingsNextPageRequested();
}
```

```dart
// lib/features/listings/presentation/blocs/listings/listings_state.dart
import 'package:equatable/equatable.dart';

import 'package:direto_da_roca/core/error/failure.dart';
import 'package:direto_da_roca/features/listings/domain/entities/listing.dart';
import 'package:direto_da_roca/features/listings/domain/entities/listings_filter.dart';
import 'package:direto_da_roca/features/listings/domain/repositories/listings_repository.dart';

enum ListingsStatus { initial, loading, loadingMore, ready, empty, failure }

final class ListingsState extends Equatable {
  const ListingsState._({
    required this.status,
    required this.filter,
    this.items = const [],
    this.cursor,
    this.failure,
  })  : assert(
          failure == null || status == ListingsStatus.failure,
          'failure só existe no status failure',
        ),
        assert(
          status != ListingsStatus.loadingMore || items.isNotEmpty,
          'carregar mais exige itens anteriores',
        ),
        assert(
          status != ListingsStatus.empty || items.isEmpty,
          'empty com itens é estado impossível',
        );

  const ListingsState.initial()
      : this._(status: ListingsStatus.initial, filter: const ListingsFilter());

  const ListingsState.loading({required ListingsFilter filter, List<Listing> previous = const []})
      : this._(status: ListingsStatus.loading, filter: filter, items: previous);

  const ListingsState.loadingMore({
    required ListingsFilter filter,
    required List<Listing> items,
    required ListingCursor cursor,
  }) : this._(status: ListingsStatus.loadingMore, filter: filter, items: items, cursor: cursor);

  const ListingsState.ready({
    required ListingsFilter filter,
    required List<Listing> items,
    required ListingCursor? cursor,
  }) : this._(status: ListingsStatus.ready, filter: filter, items: items, cursor: cursor);

  const ListingsState.empty({required ListingsFilter filter})
      : this._(status: ListingsStatus.empty, filter: filter);

  const ListingsState.failure({
    required ListingsFilter filter,
    List<Listing> previous = const [],
    this.failure = const UnknownFailure(operation: 'listings_page', cause: ''),
  }) : this._(status: ListingsStatus.failure, filter: filter, items: previous);

  final ListingsStatus status;

  /// Fica sempre presente, mesmo em `initial` e `loading`. Sem isso, o filtro
  /// mostrado no `AppBar` pisca durante a recarga.
  final ListingsFilter filter;

  final List<Listing> items;

  /// `null` = não há mais. Distingue "ainda não sei" de "acabou", que é o que
  /// permite mostrar o fim da lista sem uma chamada extra.
  final ListingCursor? cursor;

  final Failure? failure;

  bool get hasMore => cursor != null;
  bool get isBusy =>
      status == ListingsStatus.loading || status == ListingsStatus.loadingMore;

  /// Recarga com filtro novo, preservando o que já está na tela. É um método, e
  /// não `copyWith`, porque `copyWith(filter: ...)` permitiria apagar `cursor` sem
  /// querer e recarregar a página 1 sem querer.
  ListingsState withFilter(ListingsFilter filter, {bool reload = true}) {
    if (reload) return ListingsState.loading(filter: filter);
    return ListingsState.ready(filter: filter, items: items, cursor: cursor);
  }

  @override
  List<Object?> get props => [status, filter, items, cursor, failure];
}
```

O bloc que o consome, com o guard de geração de §6.4:

```dart
// lib/features/listings/presentation/blocs/listings/listings_bloc.dart — trecho
ListingsBloc({required ListingsRepository repository})
    : _repository = repository,
      super(const ListingsState.initial()) {
    on<ListingsStarted>(_onStarted, transformer: restartable());
    on<ListingsFilterChanged>(_onFilterChanged, transformer: restartable());
    on<ListingsNextPageRequested>(_onNextPage, transformer: droppable());
  }

Future<void> _onFilterChanged(
  ListingsFilterChanged event,
  Emitter<ListingsState> emit,
) async {
  // Recarrega do zero e zera o cursor: com filtro novo, a página 2 da query
  // antiga não tem nada a ver.
  emit(ListingsState.loading(filter: event.filter));
  await _loadFirstPage(emit, filter: event.filter);
}
```

E o `buildWhen` que só reconstrói quando muda o que a tela mostra:

```dart
BlocBuilder<ListingsBloc, ListingsState>(
  buildWhen: (previous, current) =>
      previous.status != current.status ||
      previous.items != current.items ||
      previous.failure != current.failure,
  builder: /* … */,
)
```

---

## 6. Orquestração de requisição assíncrona

### 6.1 A escolha

`bloc_concurrency` **não está** na lista fechada de pacotes, então não entra sem aprovação de A
— e não precisa. A escolha é: **`EventTransformer` próprio em `core/async/`, mais um guard
explícito de geração no bloc.**

O que foi descartado e por quê:

| Opção | Veredito |
| --- | --- |
| `Future` direto no handler, sem nada | **Perde.** Sem debounce, cada tecla dispara uma query; sem cancelamento, o resultado antigo chega depois do novo e a lista pisca para trás. |
| `bloc_concurrency` (`restartable`, `droppable`) | **Fora da lista.** E o ponto que decide: `restartable` cancela a *assinatura* do stream interno, o que impede evento de entrar na fila, mas **não interrompe um handler que já está parado dentro de um `await`**. O `emit` dele continua válido. Só o guard resolve. |
| Emite manual, tudo à mão, sem transformer | **Perde.** O debounce vira um `Future.delayed` dentro do handler, que segura o handler, e o `Timer` não é cancelável, então o `await` de um debounce atrasa o próximo evento em vez de abortá-lo. |
| **Transformer próprio + guard de geração** | **Vence.** O transformer cuida de debounce, de não enfileirar e de descartar trabalho; o guard cuida de correctness. Cada um faz uma coisa, e cada um tem teste. |

### 6.2 `sequential`, `droppable` e `restartable`

`sequential` sai de graça, porque `asyncExpand` já serializa:

```dart
// lib/core/async/sequential.dart
import 'dart:async';

import 'package:bloc/bloc.dart';

/// Processa um evento por vez, em ordem. `asyncExpand` já é isso: um evento novo
/// espera o anterior terminar, em vez de cancelar.
EventTransformer<E> sequential<E>() => (events, mapper) => events.asyncExpand(mapper);
```

```dart
// lib/core/async/droppable.dart
import 'dart:async';

import 'package:bloc/bloc.dart';

/// Descarta o evento enquanto um anterior ainda está em andamento.
///
/// Diferente de `sequential`, que enfileira: aqui a intenção é "este evento não
/// tem valor agora, some o resultado". Usar no botão de contato e em
/// "carregar mais", onde o custo de repetir é real.
///
/// `asyncExpand` serializa por construção: enquanto a stream interna do evento
/// anterior não fechar, os próximos eventos nem são entregues ao `mapper`.
/// É por isso que não existe flag `isRunning` — `asyncExpand` já garante a
/// exclusividade, e um guard manual só criaria uma segunda fonte de verdade.
EventTransformer<E> droppable<E>() {
  return (events, mapper) => events.asyncExpand(
        (event) {
          final mapped = mapper(event);
          // `Stream<E>.empty()` **sem** `const`: constante não aceita type
          // parameter, e `const Stream<E>.empty()` não compila. É a diferença
          // entre `droppable` funcionar e o projeto não compilar.
          if (mapped == null) return Stream<E>.empty();
          // `where` com predicado constante é pass-through; existe só para o tipo
          // do retorno casar com `Stream<E>`. A serialização vem do `asyncExpand`.
          return mapped.where((_) => true);
        },
      );
}
```

```dart
// lib/core/async/restartable.dart
import 'dart:async';

import 'package:bloc/bloc.dart';

/// Descarta o resultado do evento anterior quando um novo chega.
///
/// O que este transformer resolve: parar de puxar eventos da fila e cancelar a
/// assinatura da streamagem interna anterior. O que ele NÃO resolve, e ninguém
/// deve achar que resolve: um handler que já está parado dentro de um `await`
/// continua rodando, e o `emit` dele continua válido. Por isso o bloc
/// precisa do guard de geração em §6.4.
EventTransformer<E> restartable<E>() {
  return (events, mapper) {
    // `late` não serve aqui: `onDone` fecha o controller antes de qualquer
    // `start`, e o analisador acusa `definitely unassigned`. O controller é criado
    // primeiro, e a stream interna é ligada em `onListen`.
    late StreamController<E> controller;
    StreamSubscription<E>? current;
    StreamSubscription<E>? outer;
    var outerDone = false;
    var started = false;

    void maybeClose() {
      if (outerDone && current == null && !controller.isClosed) {
        controller.close();
      }
    }

    void start(E event) {
      final mapped = mapper(event);
      if (mapped == null) return;
      current?.cancel();
      StreamSubscription<E>? inner;
      inner = mapped.listen(
        controller.add,
        onError: controller.addError,
        onDone: () {
          if (identical(current, inner)) current = null;
          maybeClose();
        },
      );
      current = inner;
    }

    controller = StreamController<E>(
      onListen: () {
        started = true;
        outer = events.listen(
          start,
          onError: controller.addError,
          onDone: () {
            outerDone = true;
            maybeClose();
          },
        );
      },
      onCancel: () {
        outer?.cancel();
        current?.cancel();
      },
    );

    // Sem eventos e stream já fechado: fecha sozinho, senão o `bloc_test` fica
    // esperando um `stream` que nunca produz `done`.
    if (!started) maybeClose();

    return controller.stream;
  };
}
```

### 6.3 Debounce

```dart
// lib/core/async/debounce.dart
import 'dart:async';

import 'package:bloc/bloc.dart';

/// Emite o último evento depois de `duration` sem novos eventos.
///
/// Composição: `debounce` **antes** de `restartable`/`droppable`. Na ordem
/// contrária, o evento novo chega ao `restartable` a cada tecla e cancela o
/// trabalho anterior, e o usuário vê a busca piscar.
EventTransformer<E> debounce<E>(Duration duration) {
  return (events, mapper) =>
      events.transform(_DebounceTransformer<E>(duration)).asyncExpand(mapper);
}

final class _DebounceTransformer<E> extends StreamTransformerBase<E, E> {
  _DebounceTransformer(this._duration);

  final Duration _duration;

  @override
  Stream<E> bind(Stream<E> events) {
    // `sync: true` para o evento sair no mesmo microtask em que o timer dispara:
    // com `false`, o bloc_test precisa de um `await` extra e o teste fica lento
    // sem motivo.
    late StreamController<E> controller;
    Timer? timer;
    E? pending;
    var hasPending = false;

    void flush() {
      timer = null;
      if (!hasPending) return;
      hasPending = false;
      controller.add(pending as E);
    }

    controller = StreamController<E>(
      onListen: () {
        events.listen(
          (event) {
            pending = event;
            hasPending = true;
            timer?.cancel();
            timer = Timer(_duration, flush);
          },
          onError: controller.addError,
          onDone: () {
            // O que chegou no fim do stream não pode ser engolido: sem este
            // `flush`, um evento emitido e imediatamente depois o stream fechado
            // nunca chega ao bloc.
            timer?.cancel();
            flush();
            controller.close();
          },
        );
      },
    );

    return controller.stream;
  }
}
```

**Limite conhecido, e ele é aceitável:** o transformer de debounce não implementa
`onPause`/`onResume`. O bloc nunca pausa o stream de eventos, então não há caso de uso; e
suportar pause exigiria guardar o tempo restante, o que é complexidade sem comprador.

**Constante de tempo, em um lugar só:**

```dart
// lib/core/constants/debounce.dart
/// 300 ms. Abaixo de 200 ms a busca dispara durante a digitação e a lista pisca
/// em 3G; acima de 400 ms a resposta parece desconectada do que foi digitado.
/// 300 ms é o ponto em que "parei de digitar" é detectado sem atraso perceptível,
/// e é também o que mantém o custo de uma query acidental em uma requisição em
/// vez de uma por tecla.
const Duration searchDebounce = Duration(milliseconds: 300);
```

E o handler, com os dois transformers em composição:

```dart
// lib/features/listings/presentation/blocs/listings/listings_bloc.dart
on<ListingsFilterChanged>(
  _onFilterChanged,
  // Debounce primeiro, restartable depois: na ordem contrária, cada tecla cancela
  // o trabalho anterior e o usuário vê a lista piscar a cada caractere.
  transformer: (events, mapper) =>
      restartable<ListingsFilterChanged>()(debounce(searchDebounce)(events, mapper), mapper),
);
```

Se a composição acima ficar verbosa demais para repetir, o certo é um helper com nome, e não
uma constante de `EventTransformer`: `EventTransformer<T>` carrega o tipo do evento, e
`const EventTransformer<ListingsFilterChanged> x = …` não pode ser constante porque a
transformação é função.

### 6.4 O guard de geração — obrigatório, não redundante

Como explicado em §6.1, o transformer não impede o `emit` obsoleto. Todo handler que pode ser
interrompido por outro evento **precisa** do guard, e o padrão é sempre o mesmo:

```dart
final int generation = ++_generation;
…
final result = await repository.…;
if (generation != _generation || emit.isDone) return;
switch (result) { … }
```

Com o `_generation` como campo do bloc:

```dart
class ListingsBloc extends Bloc<ListingsEvent, ListingsState> {
  ListingsBloc({required ListingsRepository repository})
      : _repository = repository,
        super(const ListingsState.initial());

  final ListingsRepository _repository;

  /// Incrementado a cada evento que invalida o anterior. Ver §6.1: o transformer
  /// para de puxar evento, mas não cancela o `await` em curso.
  int _generation = 0;

  Future<void> _loadFirstPage(
    Emitter<ListingsState> emit, {
    required ListingsFilter filter,
  }) async {
    final int generation = ++_generation;
    emit(ListingsState.loading(filter: filter));
    final result = await _repository.fetchPage(filter: filter);
    if (generation != _generation || emit.isDone) return;
    switch (result) {
      case Ok<ListingsPage, Failure>(:final value) => emit(
          value.items.isEmpty
              ? ListingsState.empty(filter: filter)
              : ListingsState.ready(
                  filter: filter,
                  items: value.items,
                  cursor: value.nextCursor,
                ),
        );
      case Err<ListingsPage, Failure>(:final error) => emit(
          ListingsState.failure(filter: filter, failure: error),
        );
    }
  }
}
```

### 6.5 Cancelamento de I/O: o que existe e o que não existe

**Não existe cancelamento de requisição HTTP neste projeto.** Nem `package:http` nem
`supabase_flutter` expõem `AbortSignal` para o cliente Dart comum, e nenhuma das duas está na
lista de pacotes com um substituto.

O que se cancela, então, é **a entrega do resultado**: a assinatura da streamagem interna
(transformer) e o `emit` obsoleto (guard). Isso é suficiente para o usuário — ele não vê
resultado velho — e é exatamente o que o `S4` audita.

Se um dia for preciso cancelamento real de I/O, é um ADR, porque significa adicionar pacote.

### 6.6 Checklist de orquestração

Toda chamada de rede, sem exceção:

- [ ] passa por um repositório de `domain`, nunca por um cliente direto no bloc;
- [ ] devolve `Result<T, Failure>`, nunca lança;
- [ ] o handler tem `switch` exaustivo sobre o `Result`;
- [ ] o handler que pode ser interrompido tem guard de geração;
- [ ] o transformer escolhido é o da tabela abaixo;
- [ ] `emit.isDone` é checado depois de todo `await` cujo bloc pode ter sido fechado;
- [ ] nenhum `Future` fica sem `await` nem sem `unawaited()` explícito.

| Situação | Transformer |
| --- | --- |
| busca por texto, filtro que muda a cada tecla | `debounce(300ms)` + `restartable()` |
| trocar o filtro (select, chip) | `restartable()` |
| carregar a próxima página | `droppable()` |
| toque em "Contatar" | `droppable()` |
| salvar formulário | `droppable()` |
| fluxo que precisa terminar em ordem (ex.: upload de 4 fotos) | `sequential()` |

---

## 7. Convenções de nomenclatura, formatação e lints

### 7.1 Nomes de arquivo e de tipo

| Elemento | Convenção | Exemplo |
| --- | --- | --- |
| Arquivo | `snake_case.dart` | `listings_repository.dart` |
| Classe, mixin, enum, extension | `PascalCase` | `ListingsRepository` |
| Função, método, variável, parâmetro | `camelCase` | `fetchPage` |
| Privado de biblioteca | `_` na frente | `_DebounceTransformer` |
| Widget privado do arquivo | `_` na frente | `class _CardImage extends StatelessWidget` |

Um tipo público por arquivo, e o arquivo se chama como o tipo. Isso vale inclusive para
arquivos com um `sealed` e seus subtipos, que **precisam** estar juntos (§3.2).

### 7.2 Nomes de widget: sem o sufixo `Widget`

`ListingCard`, não `ListingCardWidget`.

O sufixo não carrega informação: tudo que mora em `presentation/widgets/` e estende
`StatelessWidget` ou `StatefulWidget` é um widget, e o nome do arquivo já diz isso. Ele existe
na API do próprio Flutter por um motivo específico — desambiguar abstrações do framework de
substantivos comuns: `Text` é texto, `Form` é formulário, `Table` é tabela. Os nossos widgets
não são primitivas de framework, então o sufixo é ruído.

Três consequências concretas de manter o sufixo fora:

1. **Nome público é API.** Se um dia `ListingCard` virar um widget de um pacote próprio, o nome
   que as pessoas vão usar é o que está no arquivo. `ListingCardWidget` é um nome que ninguém
   escolhe de propósito.
2. **Colisão semântica.** Nomes terminados em `Widget` que não são `Widget` confundem quem lê
   a assinatura: `final Widget Function(Listing) buildCard` passa a ter duas coisas que parecem
   a mesma.
3. **O revisor ganha uma regra mecânica.** "Nenhum nome novo terminado em `Widget`" é uma
   busca de uma linha. "Nenhum nome terminado em `Widget` sem ser `StatelessWidget`" não é.

Exceções, com nome próprio, e são só duas:

- **`…Page` para tela roteável:** `ListingsPage`, `ListingDetailPage`, `LoginPage`. Distingue o
  que tem rota de URL do que é pedaço de tela, e A2 depende dessa distinção.
- **`…View` para os três estados base** de T-2.2: `LoadingView`, `ErrorView`, `EmptyView`. São
  nomes que o plano já fixou, e são bons: `Loading` sozinho colidiria com o estado.

### 7.3 Constantes

**Constantes em `lowerCamelCase`, nunca em `SCREAMING_SNAKE_CASE`.** Isto não é gosto: é o
Effective Dart ("PREFER using `lowerCamelCase` for constant names", regra do lint
`constant_identifier_names`, que vem ligada no `flutter_lints` — ou seja, o lint já impõe, e a
regra aqui é consequência).

Os três motivos que o próprio Dart deu para abandonar o `SCREAMING_CAPS` são os nossos:

- **Looks ruim justamente nos casos que temos.** `ValidationCode.tooShort` fica
  `TOO_SHORT` e `ValidationCode.inThePast` fica `IN_THE_PAST` — e `enum ListingCategory.fruit`
  vira `FRUIT`, que parece constante de C, não valor de domínio.
- **Constante muda de `const` para `final` com frequência**, e aí o nome precisa mudar junto.
  `listingsPageSize` sobrevive à mudança; `LISTINGS_PAGE_SIZE` não sobrevive sem diff.
- **`values` num enum é `const` e minúsculo.** A biblioteca seria inconsistente por dentro.

Para constantes agrupadas, o padrão é uma classe de constantes com membros `lowerCamelCase`:

```dart
// lib/core/constants/pagination.dart
import 'package:direto_da_roca/features/listings/domain/repositories/listings_repository.dart';

/// Uma tela e meia de anúncios. Abaixo de 10 a paginação fica visível durante o
/// scroll; acima de 30 a primeira página custa mais que o orçamento de payload
/// (§10.2) e a lista demora a aparecer no 3G.
const int listingsPageSize = 20;
```

E o resto de `const`:

- `const` em todo construtor que pode ser `const`. Entidade, evento, estado e `Failure` são
  todos `const` no projeto, e é por isso que o `bloc_test` consegue comparar estado por
  igualdade e não por identidade.
- `final` para campo que não pode mudar mas whose valor não é conhecida na compilação
  (injeção de dependência). `final`, nunca `late` sem necessidade.
- `late` é permitido, com justificativa visível: `AppConfig` lido de `--dart-define` (T-0.2) e
  o `SupabaseClient` criado no bootstrap (T-2.1). Em qualquer outro lugar, `late` é
  `Null` esperando acontecer.

### 7.4 Formatação e constraint de SDK

**`environment: sdk: ^3.9.0`.** E o que isso implica, que é a parte que o plano já sinalizou:

1. A constraint **define a versão de linguagem** de cada arquivo, e a partir do Dart 3.7 a
   versão de linguagem define o **estilo do formatter**. Com 3.9, `dart format` usa o *tall
   style*: 80 colunas, indentação de 2, e vírgula final **gerenciada pelo próprio formatter**,
   que adiciona e remove conforme o quebra da linha.
2. Por consequência, **é proibido fights com o formatter**: não quebrar linha à mão, não gerenciar
   vírgula final à mão, e **o lint `lines_longer_than_80_chars` fica desligado** — ele
   contradiz o formatter em string longa e URL, que o formatter não consegue quebrar.
3. Por consequência também, **`dart format` precisa rodar depois de `pub get`**, sempre. O
   formatter descobre a versão de linguagem lendo o `package_config.json`; sem ele, ele erra
   **todo** arquivo. Isso já está na ordem do CI de T-0.3, e a ordem não é vaidade.
4. **Subir a constraint é um ato deliberado**, em PR próprio, com o formatter rodando. O SDK
   manda o estilo por versão de linguagem justamente para o estilo não mudar sozinho quando o
   SDK da máquina do dev muda.
5. Não definimos `formatter.page_width`. O default é 80, e deixar o default explícito significa
   que uma mudança futura de default aparece no CI como diff, e não como surprise em seis
   máquinas diferentes.

O Flutter correspondente é `>=3.35.0` (o primeiro Flutter com Dart 3.9). O CI fixa a versão por
`flutter-version`, e `dependabot.yml` (T-7.5) propose subir a constraint em PR próprio.

### 7.5 Conjunto de lints

**Escolha: `flutter_lints` 6.0.0 como base, mais um bloco `rules:` explícito.**

```yaml
# analysis_options.yaml
include: package:flutter_lints/flutter.yaml

analyzer:
  language:
    strict-casts: true
    strict-raw-types: true
  errors:
    # Um subtipo novo de Failure/estado que ninguém trata é bug, não estilo.
    exhaustive_cases: error
    # Silenciar erro de tipo é sempre errado; a análise engole bug real.
    todo: ignore
  exclude:
    - "**/*.g.dart"
    - "lib/l10n/gen/**"

linter:
  rules:
    # --- o que o flutter_lints 6 já traz; re-declarado para o arquivo ser
    # --- auto-documentado e para uma atualização do pacote não desligar.
    avoid_print: true
    use_build_context_synchronously: true
    constant_identifier_names: true
    non_constant_identifier_names: true

    # --- adicionados pelo projeto, com motivo
    always_use_package_imports: true       # import de lib/ não é relativo; mover arquivo não quebra import
    directives_ordering: true              # dart:, package:, relativo — nesta ordem, sempre
    unawaited_futures: true                # §6: Future sem await deixa a tela em loading para sempre
    comment_references: true               # referência a símbolo que não existe mais
    require_trailing_commas: false         # o formatter do tall style é dono das vírgulas (§7.4)
    lines_longer_than_80_chars: false      # (§7.4)
    avoid_dynamic_calls: false             # ver abaixo
    one_member_abstracts: false            # ver abaixo
    public_member_api_docs: false          # documento público não é o ponto deste projeto
    prefer_single_quotes: true
    sort_pub_dependencies: true
```

**Por que `flutter_lints` e não o outro candidato.** A alternativa real hoje é
`very_good_analysis` (VGV, versão 11 exige Dart 3.13). Ela é boa e é a escolha de muita gente,
mas perde aqui por três motivos concretos:

1. **Ela habilita `avoid_dynamic_calls`.** O retorno do Supabase é `Map<String, dynamic>` e
   `Object?`. A regra transformaria cada acesso de coluna em `dynamic` em um aviso, e a resposta
   seria `// ignore:` em todo mapper — que é pior do que a regra. É uma regra que grita onde o
   domínio legítimamente é dinâmico.
2. **Ela impõe `sort_unrelated_constructors_first` e formatação própria** (a VGV 11 configura
   vírgula final automática), o que significa maintenance de duas policies de formatação
   concorrentes com o SDK.
3. **Ela sobe o piso para Dart 3.13** sem ganho para este projeto. `flutter_lints` é mantido pela
   equipe do Dart, acompanha o SDK, e é o que `flutter create` já escreve — trocar é mexer em
   todo arquivo depois de trocar.

O que o `flutter_lints` **não** cobre e este projeto precisa é pequeno e está na lista acima.
Regra geral: **preferimos um conjunto mínimo, justificado linha a linha, a um conjunto grande de
terceiro.** Cada regra que alguém não entende é uma regra que alguém dá `ignore`.

`strict-casts: true` e `strict-raw-types: true` não são lints, e valem mais que a maioria dos
lints deste arquivo: é o que faz `json['price_cents'] as num?` falhar na revisão se o tipo
mudar, em vez de explodir em produção.

### 7.6 Pacotes — lista fechada, confirmada por A0

Os que já estavam na lista do plano, mais os que A0 aprova agora. **Tudo o mais exige A.**

| Pacote | Tipo | Por que |
| --- | --- | --- |
| `flutter_bloc`, `equatable`, `supabase_flutter`, `geolocator`, `flutter_map`, `latlong2`, `image_picker`, `image`, `url_launcher`, `intl`, `flutter_localizations` | runtime | já na lista do plano |
| `bloc_test`, `mocktail` | dev | já na lista do plano |
| **`flutter_lints ^6.0.0`** | dev | **aprovado por A0** — é o slot "lints" da lista fechada |
| **`coverage ^1.15.1`** | dev | **aprovado por A0** — `flutter test --coverage` gera LCOV mas não agrega nem impõe nada; o threshold de §8.5 precisa de um agregador |
| **`fake_async ^1.3.3`** | dev | **aprovado por A0** — vem junto do `package:test`, mas o lint `depend_on_referenced_packages` (ligado pelo `flutter_lints`) exige declarar; é o que permite testar debounce sem `await` de 300 ms |

**Proibido, sem exceção:** `build_runner`, `json_serializable`, `freezed`, `rxdart`,
`bloc_concurrency`, `get_it`, `go_router`, `google_sign_in`, `dartz`, `fpdart`. Os três
primeiros são a "geração de código" que o `INTENT.md` §7 tirou da v1; `rxdart` e
`bloc_concurrency` são o que §6 substituiu; os três últimos estão fora da decisão do INTENT e
seriam decisão de arquitetura, não de implementação.

**`flutter gen-l10n` não é geração de código no sentido proibido.** É o pipeline ARB do
próprio SDK, é exigido por T-0.1 e T-2.4, e é a única "geração" permitida. Configure em
`l10n.yaml`:

```yaml
arb-dir: lib/l10n
template-arb-file: app_pt.arb
output-localization-file: app_localizations.dart
output-class-name: AppLocalizations
# Sem o pacote sintético: o arquivo gerado vira um caminho de verdade no repositório,
# importável como package:direto_da_roca/l10n/gen/app_localizations.dart, e
# sobrevive a mudança de SDK.
synthetic-package: false
output-dir: lib/l10n/gen
nullable-getter: false
```

`nullable-getter: false` importa: sem ele, todo `l10n.failureUnknown` é `String?` e o `switch`
da §3.5 precisa de `!` em catorze linhas.

### 7.7 Nome do pacote e do aplicativo

T-0.1 depende de A0 decidir org e nome de pacote. Decidido:

| Item | Valor |
| --- | --- |
| Nome do pacote Dart | `direto_da_roca` |
| `applicationId` Android | `com.projetodaro.ca` |
| `<title>` do web | `Direto da Roça` |
| `namespace` Android | `com.projetodaro.ca` |

O `applicationId` é provisório e **muda quando T-7.0 resolver o domínio**. É barato: o
application id só vira problema quando entra na Play Store, e isso está fora da v1. O nome do pacote
Dart (`direto_da_roca`) não muda nunca, porque é o prefixo de todos os imports do projeto.

**Nomes de pasta e de pasta de feature são em inglês** (`listings`, `reports`, `map`), com o
conteúdo em português. Motivo: nome de pasta aparece em import, em nome de arquivo de
configuração e em chave de analytics, e identificador de código em inglês é convenção que todo
leitor de Dart já tem.

---

## 8. Testes

### 8.1 Estrutura

`test/` espelha `lib/` campo a campo. Um arquivo de teste por arquivo de código, nomeado
`<subject>_test.dart`, na pasta correspondente:

```
lib/features/listings/presentation/blocs/listings/listings_bloc.dart
test/features/listings/presentation/blocs/listings/listings_bloc_test.dart

lib/core/error/failure.dart
test/core/error/failure_test.dart

lib/features/listings/data/mappers/listing_mapper.dart
test/features/listings/data/mappers/listing_mapper_test.dart
```

`test/helpers/` não espelha nada e é a casa de tudo que é apoio:

```
test/helpers/
├── mocks.dart              # todas as classes Mock*, em um lugar só
├── fakes.dart              # fakes reais, quando o mocktail atrapalha
├── fallbacks.dart          # registerFallbackValue (ver §8.4)
└── fixtures/               # JSON de exemplo da view pública, versionados
```

Os fixtures são importantíssimos e são o substituto do `build_runner`: um `listings_page.json`
versionado é a prova, legível em review, de que o DTO continua batendo com a view pública. Se a
view mudar e o DTO não, o teste de mapper falha com um diff legível.

`test/` **não** espelha `lib/l10n/`. E `integration_test/` não existe na v1 (`INTENT.md` §11).

### 8.2 O que é teste de quê

| Tipo | Testa | Ferramenta | Onde |
| --- | --- | --- | --- |
| **Unitário** | entidade, DTO, mapper, falha de formatação, `wa.me`, `WaLink` | `flutter_test` | `test/**` |
| **de BLoC** | sequência de estado sob evento, incluindo falha | `bloc_test` + `mocktail` | `test/**/blocs/` |
| **de widget** | os três estados da tela, validação de formulário, navegação com voltar | `flutter_test` | `test/**/pages/` |
| **de SQL** | RLS, `grant`, `revoke`, `search_path`, as funções | **pgTAP** (D2) | `supabase/tests/` |
| **de Edge Function** | token ausente/inválido/expirado, IP divergente, limite | `deno test` | `supabase/functions/*/` |

**A segurança mora no banco, então o teste de segurança mora em SQL.** Testar RLS em Dart é
testar uma cópia da política; `supabase test db` executa a política. Por isso D2 é obrigatória e
por isso a suíte pgTAP é o gate, e não a revisão de código.

E o inverso também vale: **negação tem três mecanismos e três asserções** (A1), então um
`throws_ok(..., '42501', ...)` não substitui um `is_empty` com prova de que a linha não mudou.
Nenhum teste de escrita permitida é provado com `lives_ok`, porque `lives_ok` passa quando a
escrita casou zero linhas.

### 8.3 Regra: teste de comportamento, não de implementação

**Um bom teste de `Failure` afirma sobre a falha, nunca sobre a tradução.**

```dart
// ✅ Bom: comportamento observável, e é o comportamento que importa
test('sem rede vira NetworkFailure de tipo offline, e a UI diz que não há conexão', () async {
  when(
    () => dataSource.fetchListings(filter: filter),
  ).thenThrow(const DataError(kind: DataErrorKind.network, cause: 'offline'));

  final result = await repository.fetchPage(filter: filter);

  expect(result, isA<Err<ListingsPage, Failure>>());
  expect(
    result,
    isA<Err<ListingsPage, Failure>>().having(
      (e) => e.error,
      'error',
      const NetworkFailure(NetworkFailureKind.offline),
    ),
  );
});

test('o contato indisponível não distingue anúncio inexistente de limite estourado', () async {
  // Duas respostas que o servidor dá de forma idêntica (D1). O cliente não pode
  // transformar isso em informação: este teste é a prova de que não transforma.
  when(() => dataSource.requestContact(listingId: 'a', turnstileToken: 't'))
      .thenThrow(const DataError(kind: DataErrorKind.notFound, cause: '404'));
  when(() => dataSource.requestContact(listingId: 'b', turnstileToken: 't'))
      .thenThrow(const DataError(kind: DataErrorKind.rateLimited, cause: '429', retryAfter: Duration(minutes: 5)));

  final first = await repository.requestContact(listingId: 'a', turnstileToken: 't');
  final second = await repository.requestContact(listingId: 'b', turnstileToken: 't');

  expect(first, isA<Err<ContactDetails, Failure>>().having((e) => e.error, 'error', isA<ContactUnavailableFailure>()));
  expect(second, isA<Err<ContactDetails, Failure>>().having((e) => e.error, 'error', isA<ContactUnavailableFailure>()));
});
```

```dart
// ❌ Ruim: testa a string, e a string é proibida fora do l10n (§11.3)
test('traduz erro do Supabase', () async {
  final failure = await repository.fetchPage(filter: filter);
  expect(failure, isA<NetworkFailure>());
  expect(failure.message, 'Não foi possível carregar os anúncios');
});

// ❌ Ruim: acopla à implementação interna. Passa enquanto o log some; quebra
// quando alguém decide usar dart:developer no lugar de print.
test('traduz erro do Supabase', () async {
  when(() => dataSource.fetchListings(filter: filter)).thenThrow(const DataError(kind: DataErrorKind.network, cause: 'x'));
  final failure = await repository.fetchPage(filter: filter);
  verify(() => logger.error(any())).called(1);
});

// ❌ Ruim: passa hoje, falha amanhã por um motivo que não é o motivo do teste.
test('mapper de kind', () {
  expect(ListingMapper.kindFromRaw('sale'), equals(ListingKind.sale));
  expect(ListingMapper.kindFromRaw('lixo'), equals(ListingKind.sale)); // e por que?
});
```

O critério em uma frase: **se o teste falha quando o usuário percebe alguma coisa, ele é bom; se
falha quando alguém renomeou uma função privada, ele é ruim.**

### 8.4 Como mockar com `mocktail`

```dart
// test/helpers/mocks.dart
import 'package:mocktail/mocktail.dart';

import 'package:direto_da_roca/features/listings/domain/repositories/listings_repository.dart';
import 'package:direto_da_roca/features/listings/domain/repositories/contact_repository.dart';

class MockListingsRepository extends Mock implements ListingsRepository {}
class MockContactRepository extends Mock implements ContactRepository {}
```

Três coisas que o mocktail exige e que ninguém descobre sozinho:

**1. `registerFallbackValue` para todo argumento não nulável.** `any(named: 'filter')` devolve
`null`, e o cast para `ListingsFilter` estoura em runtime, dentro do teste, com uma mensagem
que não menciona o mocktail. O registro é **uma vez por tipo**, em `setUpAll`:

```dart
// test/helpers/fallbacks.dart
import 'package:mocktail/mocktail.dart';

import 'package:direto_da_roca/features/listings/domain/entities/listings_filter.dart';

void registerAllFallbacks() {
  registerFallbackValue(const ListingsFilter());
  registerFallbackValue(const <String>[]);
}
```

Sem isso, o sintoma é `type 'Null' is not a subtype of type 'ListingsFilter'` em um teste que
"nem mexe com filtro".

**2. Não mocke entidade.** `Listing` é `final class`, e `MockListing extends Mock implements
Listing` **não compila** — é o comportamento correto, e a entidade se constrói de verdade:

```dart
final listing = Listing(
  id: 'l1', ownerId: 'u1', title: 'Laranja', kind: ListingKind.sale,
  category: ListingCategory.fruit, status: ListingStatus.active,
  city: 'Recife', neighborhood: 'Boa Viagem',
  createdAt: DateTime(2026, 1, 1), expiresAt: DateTime(2026, 1, 15),
);
```

**3. `when` com `any` nomeado, e `thenAnswer` quando o retorno importa.**

```dart
setUpAll(registerAllFallbacks);

late MockListingsRepository repository;
late ListingsFilter filter;

setUp(() {
  repository = MockListingsRepository();
  filter = const ListingsFilter(city: 'Recife');
});

blocTest<ListingsBloc, ListingsState>(
  'carrega a primeira página e fica pronta',
  setUp: () {
    when(() => repository.fetchPage(filter: any(named: 'filter')))
        .thenAnswer((_) async => Ok<ListingsPage, Failure>(
              ListingsPage(items: [listing], nextCursor: 'c1'),
            ));
  },
  build: () => ListingsBloc(repository: repository),
  act: (bloc) => bloc.add(const ListingsStarted()),
  expect: () => [
    isA<ListingsState>().having((s) => s.status, 'status', ListingsStatus.loading),
    isA<ListingsState>()
        .having((s) => s.status, 'status', ListingsStatus.ready)
        .having((s) => s.items.length, 'itens', 1)
        .having((s) => s.hasMore, 'hasMore', true),
  ],
);

blocTest<ListingsBloc, ListingsState>(
  'sem rede: o estado vai a failure com a falha, e não fica em loading para sempre',
  setUp: () {
    when(() => repository.fetchPage(filter: any(named: 'filter')))
        .thenAnswer((_) async => Err<ListingsPage, Failure>(
              const NetworkFailure(NetworkFailureKind.offline),
            ));
  },
  build: () => ListingsBloc(repository: repository),
  act: (bloc) => bloc.add(const ListingsStarted()),
  expect: () => [
    isA<ListingsState>().having((s) => s.status, 'status', ListingsStatus.loading),
    isA<ListingsState>()
        .having((s) => s.status, 'status', ListingsStatus.failure)
        .having((s) => s.failure, 'failure', const NetworkFailure(NetworkFailureKind.offline)),
  ],
);
```

O segundo teste é o que mais vale na lista: é o que pega o bug do §3.1 — o handler que esquece
a falha e deixa o usuário preso no spinner.

### 8.5 Cobertura

**Threshold: 80% global, com piso por pasta, e a regra que realmente importa é não poder cair.**

Por que 80: abaixo disso a métrica deixa de correlacionar com defeito encontrado, e 80 é o ponto
onde a suíte começa a pagar por si. Por que a regra mais importante é outra: um piso que sobe é
inútil, e um piso que pode ser atravessado para baixo é decorativo — o plano já disse que badge
sem threshold é decoração, e threshold que ninguém compara com o valor anterior é a mesma coisa
com um número.

| Pasta | Piso | Por que este número |
| --- | --- | --- |
| `lib/core/error/**`, `lib/core/async/**`, `lib/**/domain/**` | **90%** | É Dart puro, sem engine e sem rede: testar é barato, e é onde mora a lógica que custa dinheiro quando falha |
| `lib/**/presentation/blocs/**` | **85%** | `bloc_test` é barato e é onde mora a lógica assíncrona, que é a parte mais difícil de acertar à mão |
| `lib/**/data/**` | **70%** | Parte é mapper (testável) e parte é I/O, e a segurança dessa camada é testada em SQL, não em Dart |
| `lib/app/**`, `lib/**/pages/**`, `lib/**/widgets/**` | **60%** | Widget é testado por estado, não linha a linha; `main.dart` e o `RouterDelegate` são difíceis de cobrir |
| **Global** | **80%** | — |

O que **não** conta para o número: `lib/l10n/gen/**` (gerado) e `**/*.g.dart`. Estão no
`analyzer.exclude` de §7.5 e no filtro do `tool/check_coverage.dart`.

**Execução:** `tool/check_coverage.dart` lê `coverage/lcov.info` com o pacote `coverage`
(aprovado em §7.6), agrega por pasta e **falha** se algum piso for violado **ou** se o global
cair em relação ao valor gravado em `docs/quality.md`.

**Sobre o badge.** T-7.5 pede badge de cobertura, e a intenção está certa; a implementação
proposta no plano exigiria um job com `permissions: contents: write` escrevendo de volta no
repositório, ou uma action de terceiro com SHA fixado. Em uma cadeia de entrega que o `S0` e o
`S7` auditam, não se adiciona um token com escrita no repositório **para mostrar um número que o
CI já está impondo**. A decisão está em "Divergências propostas" e precisa do A.

---

## 9. Commits, branches e PRs

### 9.1 Conventional Commits, subject em português

O **tipo e o escopo ficam em inglês**, porque é isso que ferramenta, filtro e changelog
consomem. O **assunto vai em português**, porque é isso que humano lê.

```
<tipo>(<escopo>): <assunto em português, imperativo, ≤ 72 caracteres no total>
```

| Parte | Valores |
| --- | --- |
| tipo | `feat`, `fix`, `docs`, `refactor`, `test`, `perf`, `build`, `ci`, `chore`, `revert` |
| escopo | `auth`, `listings`, `reports`, `map`, `core`, `l10n`, `app`, `ci`, `supabase`, `deps` |

Regras do subject:

- **≤ 72 caracteres na linha inteira**, contando tipo, escopo, dois-pontos e espaço. É o que
  aparece no `git log` e no histórico do GitHub sem cortar.
- **Imperativo e no presente**: "adiciona filtro por categoria", não "adicionado" nem
  "adicionando". O texto completa "commit que _…_".
- **Sem ponto final.**
- **Acentuação normal, sempre.** Acento em subject é perfeitamente legível, e proibir acento
  cria uma regra que ninguém aplica de forma consistente. A regra que existe sobre ausência de
  acento é a do slug de branch (§9.2), e ela vale porque nome de branch passa por shell e por
  URL, não por leitura.

`style` **não** é um tipo usado: `dart format` roda no CI, então um commit só de formatação não
acontece, e se acontecer é sinal de que alguém rodou format local em arquivo de outra pessoa.

Exemplos do padrão que o projeto segue:

```
feat(listings): adiciona paginação por cursor na listagem
fix(listings): impede emit obsoleto ao trocar o filtro durante o carregamento
test(listings): cobre o caso de página vazia com filtro ativo
perf(listings): reduz miniaturas para 400x300 com 20 KB de teto
docs(adr): registra o método de aproximação da localização pública
ci: falha quando o bundle web passa de 2.2 MB
fix(auth): reenvia o link mágico quando o token expira
refactor(core): extrai o guard de geração para um helper
```

### 9.2 Branches

| Prefixo | Quando |
| --- | --- |
| `feat/` | funcionalidade nova visível |
| `fix/` | correção de defeito |
| `perf/` | trabalho com budget como critério de aceite (§10) |
| `chore/` | infra, CI, dependências, refactor sem mudança de comportamento |
| `docs/` | documentação e ADR |

`main` é **protegida**: sem push direto, PR obrigatório, e "Do not allow bypassing the above
settings" ligado — porque regra de proteção que o admin contorna não é regra, e o `S7` audita a
configuração do repositório, não só o workflow.

Branch nomeada `tipo/slug`, slug em `kebab-case`, sem acento, sem emoji, minúsculo:

```
feat/lista-paginacao-por-cursor
fix/emit-obsoleto-ao-trocar-filtro
chore/ci-check-de-budgets
docs/adr-0001-free-tier
```

Um PR = uma branch = um assunto. Duas coisas não misturadas.

### 9.3 O que o revisor checa, em ordem

1. **Camadas.** `grep -rn "package:supabase_flutter" lib/features/*/presentation lib/features/*/domain`
   tem que voltar vazio. `grep -rn "package:flutter/" lib/features/*/domain` tem que voltar
   vazio. `grep -rn "package:direto_da_roca/core/config" lib/features/*/domain lib/features/*/data`
   tem que voltar vazio.
2. **Imports de `package:`**, nunca relativos entre pastas (§7.5, `always_use_package_imports`).
3. **Sem string de UI fora de l10n.** `grep -rn "Text('" lib/` e `grep -rn "'.*: .*'" lib/**/pages`
   voltam vazios.
4. **`Failure` nova tratada** — o compilador garante, mas se o PR adiciona um subtipo, o `switch`
   de `core/l10n/failure_messages.dart` e o `switch` do repositório mudaram junto.
5. **`props` completo** em qualquer entidade, evento ou `Failure` tocada.
6. **Se o PR toca `supabase/`**, o checklist da §1.1 do plano, item por item:
   - [ ] `revoke execute … from public` e `grant execute to <papel>` explícito em **toda** função
   - [ ] `set search_path = ''` em **toda** `security definer`, com todos os objetos qualificados
   - [ ] `security definer` em schema **não exposto** (`private`), nunca em `public`
   - [ ] `security_invoker = true` em **toda** view
   - [ ] `supabase db lint` sem erro
   - [ ] nenhuma tabela com RLS desligada
   - [ ] nenhuma tabela com grant de escrita para `anon`
   - [ ] `revoke all on table … from anon, authenticated` presente **antes** de qualquer `grant`
7. **Se o PR toca SQL de negação**, a asserção certa para o mecanismo (A1): `42501` para falta de
   grant e para `with check`, `is_empty` com `returning` mais leitura de prova para clause
   `using`. E **nenhuma** escrita permitida provada com `lives_ok`.
8. **Nenhum segredo, nenhuma URL de ambiente** no código.

Severidade conforme a §1 do plano: Critical bloqueia o merge, Major corrige antes do merge ou
vira task explícita, Minor pode esperar.

O revisor **não** aprova sem `format`, `analyze` e `test` verdes. Se eles estão vermelhos, o
comentário é "rode o CI", não uma revisão de código.

### 9.4 Tamanho de PR

**Regra: < 400 linhas alteradas por PR de agente.** Adicionadas **+** removidas.

Como medir, exatamente:

```bash
git diff --numstat origin/main...HEAD | awk '{s += $1 + $2} END {print s}'
```

Se passar de 400, o dev **pede divisão ao A** antes de continuar — não abre o PR grande e não
divide por conta própria, porque divisão por critério errado (metade de uma função, por exemplo)
custa mais que o PR grande. O A decide os cortes, e o corte certo é quase sempre **por task do
plano**, ou por camada dentro de uma task.

A definição de pronto do plano vale inteira para cada PR: `pub get` antes do
`format --set-exit-if-changed`, format sem alterações, `analyze` sem aviso, testes da task
escritos e verdes, nenhum segredo, strings de UI via l10n, pastas e nomes conforme a spec,
nenhuma dependência fora da lista, e documentação atualizada quando a task muda algo que outro
dev precisa saber.

### 9.5 Squash merge, e por quê

**Squash, sempre, sem `--no-ff`.** Quatro razões:

1. **O histórico da branch é rascunho, não conteúdo.** "wip", "corrigindo lint", "teste passando".
   Squashado, isso não chega à `main`.
2. **`main` fica linear.** Sem merge commit, `git log` é a sequência do que foi feito, e
   `git bisect` funciona sem precisar adivinhar qual é o commit bom.
3. **O PR já foi revisado como unidade.** O valor da revisão está no conjunto, não nos commits
   intermediários. Preservar os commits intermediários guarda a estrutura de trabalho de uma
   pessoa, que é informação sem valor para quem lê depois.
4. **Dependabot continua reverteível.** Um PR de dependabot vira **um** commit; reverter é
   `git revert` de um hash, não de uma sequência.

O que o squash exige: **o título do PR já é o subject do Conventional Commit**, porque o GitHub
usa o título do PR como mensagem do commit. Se o título estiver errado, o histórico fica errado
mesmo com squash.

Nenhuma branch de release, nenhuma tag movida para o merge — tags são de T-7.4, e é a tag que
dispara o APK.

---

## 10. Budgets de performance

Budget sem método de medição não é budget, é número decorativo. Por isso a estrutura é: número,
derivação, como medir, e onde no CI falha.

### 10.1 O sistema de medição, antes dos números

**O CI mede bytes. Não mede segundos.** A razão é determinismo: bytes são reproduzíveis, e
segundos num runner compartilhado não são. Então:

| Budget | Onde é medido | Quando falha |
| --- | --- | --- |
| Bytes (bundle, payload, foto) | CI, todo PR | **Falha o build** |
| Tempo (primeiro conteúdo, contato, jank) | Gate `S4`/`S8`, com relatório | **Falha o gate** |
| Cobertura | CI, todo PR | **Falha o build** |

**Consequência para T-0.3: o build web de release roda em todo PR.** Sem build não há byte, e sem
byte nenhum budget de bundle é verificável. Com cache agressivo de pub e do `build/web`, o custo
é de poucos minutos no runner do GitHub, que é gratuito em repositório público (é a mesma razão
que pagou D2).

O número vive em `budgets.yaml`, versionado, e é espelhado em `lib/core/constants/budgets.dart`
para o código poder respeitá-lo. O CI roda `dart run tool/check_budgets.dart --explain`, que
imprime o medido, o orçamento, a diferença e **os cinco maiores arquivos de `build/web/`** — para
o dev saber onde olhar sem fazer bisect de dependência.

### 10.2 Budgets, com derivação

#### B1 — `main.dart.js` em release

| | |
| --- | --- |
| **Alvo** | **≤ 2.200.000 bytes** (`main.dart.js` cru) e **≤ 600.000 bytes** gzip |
| **Teto duro** | **2.600.000 bytes** |
| **Faixa de aviso** | 1.900.000 – 2.200.000 → anotação `warning::`, não falha |
| Medir | `flutter build web --release --dart-define-from-file=config/prod.json` e depois `wc -c < build/web/main.dart.js` |
| Onde no CI | step `budgets`, todo PR que toque `lib/**` ou `pubspec.yaml` |

**O que cabe:** Flutter + Material, `flutter_bloc`, `equatable`, `supabase_flutter` (o cliente
Dart puro, sem SDK nativo no bundle web), Turnstile (que é JS externo do Cloudflare, não entra) e
`url_launcher` (que na web é um `window.open`, ~2 KB).

**O que não cabe, e por quê:** **`package:image`.** O pacote traz todos os codecs — JPEG, PNG,
WebP, GIF, TIFF, BMP, TGA, PNM, exaRLE, DDS, farbfeld — e o padrão de registro dele
(tree-shaking ruim em dart2js) faz o Dart2JS levar quase tudo. Somado ao resto, passa de
2,6 MB sozinho. A solução **não** é cortar o pacote (o `INTENT.md` §6.7 fixa `image` para
compressão e para zerar EXIF), e sim **import condicional**, que deixa o código inacessível no
build web e o remove do bundle:

```dart
// lib/features/listings/data/image/image_processor.dart
export 'image_processor_io.dart'
    if (dart.library.js_interop) 'image_processor_web.dart';
```

- `image_processor_io.dart`: `package:image`, `Isolate.run`, decodifica, redimensiona, **zera o
  EXIF explicitamente** e re-codifica. É o caminho do Android.
- `image_processor_web.dart`: `createImageBitmap` + `canvas` + `toBlob('image/jpeg', 0.82)`. O
  re-encode do canvas **descarta o EXIF por construção**, e o custo em bytes no bundle é zero.
  É o caminho da web, e resolve o problema que a §5 do plano registra: `compute` **roda na main
  thread na web** e `package:image` não existe de isolate lá.

Efeito colateral desejável: **o budget impõe uma regra de arquitetura.** Se alguém importar
`package:image` direto em código alcançável da web, o `main.dart.js` estoura o teto e o CI
falha. O orçamento é o guarda.

**A5 precisa especificar T-5.1 assim.** Está em "Divergências propostas".

**Fora do budget da v1, e por quê:** `flutter_map` e `latlong2` (Fase 9, entram depois) e o
plugin `geolocator` na web (T-9.2). Quando a Fase 9 subir, o budget é **re-baselineado com o
delta declarado**, nunca silenciosamente reescalado.

#### B2 — payload de uma página de listagem

| | |
| --- | --- |
| **Linhas por página** | **20** |
| **JSON da resposta** | **≤ 64.000 bytes** comprimida, **≤ 256.000 bytes** crua |
| **Miniatura** | **≤ 20.000 bytes**, **400 × 300 px**, JPEG |
| **Bytes para preencher a tela** | **≤ 480.000 bytes** comprimidos (64 KB de JSON + 20 × 20 KB) |

Derivação: 20 linhas é uma tela e meia de cartões. Abaixo de 10 a paginação fica visível durante
o scroll; acima de 30 a primeira página custa mais que o orçamento de payload e demora a
aparecer em 3G. E o teto de 64 KB de JSON tem folga enorme sobre o consumo real — 20 anúncios com
título, categoria, bairro e caminho de imagem dão cerca de **7 KB**. A folga é proposital: ela
existe para pegar regressão, do tipo alguém incluir `description` na consulta da lista, ou
selecionar a coluna `location` por engano. **O JSON é barato; as imagens são o custo.** Por isso
o orçamento que aperta de verdade é o da miniatura.

E aqui está a conta de egress, porque D3 diz que **5 GB de egress é o limite que estoura
primeiro**:

```
100 usuários do piloto × 5 páginas × 20 anúncios × 20 KB de miniatura = 200 MB
100 usuários × 10 detalhes × 300 KB de foto (só a capa)              = 300 MB
                                                                    ─────────
                                                                    ~500 MB
```

Folga de 10×. O orçamento da miniatura é o que garante essa folga: dobrar a miniatura de 20 KB
para 40 KB dobra o consumo do piloto e é exatamente o tipo de economia que mata o free tier.

| | |
| --- | --- |
| Medir | `Content-Length` da resposta do endpoint de listagem, mais a soma dos `Content-Length` das 20 miniaturas |
| Onde no CI | `S4` com as seeds ampliadas do T-1.7; o teto de 64 KB também é testável em Dart, com o fixture de `test/helpers/fixtures/` |

#### B3 — foto de anúncio

| | |
| --- | --- |
| **Peso por foto** | **≤ 300.000 bytes** |
| **Dimensões máximas** | **≤ 1600 × 1200 px** |
| **Fotos por anúncio** | **≤ 4** (1 capa + 3) |
| **Miniatura gerada** | **≤ 20.000 bytes**, **400 × 300 px** |

O número de 300 KB **não é novo**: é o que D3 já usa para a conta de egress, e ficar com outro
número criaria duas verdades. 300 KB por foto é ~16.700 visualizações no egress de 5 GB.

1600 × 1200 é o teto porque a foto aparece em meia tela de celular, em até 3× de densidade: 1600
px de largura cobrem ~533 dp com folga, e um usuário de celular nunca vê mais que isso. Acima
disso é byte e upload jogados fora.

**4 fotos** porque: 4 já mostra condição, quantidade e escala do produto; 5 seria mais um upload
de 300 KB para o comprador pagar em 3G, e o excedente de quintal não se vende por catálogo de
loja.

| | |
| --- | --- |
| Medir | o `UploadPayload` que T-5.2 produz, em teste unitário de T-5.1: `expect(bytes.length, lessThanOrEqualTo(300000))` e as dimensões |
| Onde no CI | todo PR; e `S5` baixa o objeto **efetivamente gravado no bucket** e mede de novo |

E o **teto de tempo de processamento na web**, que é o par real deste budget: **≤ 400 ms de
bloqueio da main thread** por foto. Passar disso é o que a A5 tem de resolver, e a razão pela
qual a miniatura tem que ser gerada no browser e não em `package:image`. `compute` na web é
`main thread`; o orçamento de 400 ms é o que impede um usuário em 3G de ver a interface congelar
por um segundo e meio ao escolher quatro fotos.

#### B4 — tempo até o primeiro conteúdo

| | |
| --- | --- |
| **Alvo (gate)** | **≤ 4,0 s** até o primeiro cartão de anúncio, em **Fast 4G (9 Mbps / 85 ms RTT)** |
| **Referência (não bloqueante)** | **≤ 20 s** em **3G (1,6 Mbps / 150 ms RTT)** |
| **Segundo carregamento, com cache** | **≤ 2,5 s** em Fast 4G |

Os perfis são declarados à mão no DevTools, não confiados ao preset: o número só é reproduzível
se o perfil é escrito.

Derivação do caminho crítico (bytes comprimidos):

| Item | Orçamentado |
| --- | --- |
| `main.dart.js` gzip | ≤ 600 KB |
| Renderizador (CanvasKit ou SkWasm) | ≤ 2.600 KB |
| `flutter_bootstrap.js`, fontes, `MaterialIcons` | ≤ 350 KB |
| 1ª página de dados | ≤ 64 KB |
| 4 miniaturas acima da dobra | ≤ 80 KB |
| **Total** | **≤ 3.710 KB** |

Em Fast 4G (≈ 1.125 MB/s) são 3,3 s de transferência mais ~0,35 s de RTTs, o que fecha em 4,0 s
com pouca folga. Em 3G (200 KB/s) são **18,5 s**.

**Esse número de 20 s é o achado mais importante desta seção, e ele não é culpa do nosso
código.** O renderizador sozinho responde por ~70% do caminho crítico, e ele é downloaded, não
escrito por nós. Nenhuma otimização de bundle muda isso de 20 s para 4 s em 3G. Três
consequências, todas de produto:

1. **O alvo de primeira pintura é medido em 4G, e isso está escrito.** O piloto é Brazilian, é 4G, e
   quem está em 3G ainda é minoria. Decidir isso explicitamente é melhor do que descobrir no
   S8 que o orçamento é insatisfazível.
2. **O número de 3G é registrado mesmo assim.** Ele é o que justifica a proposta em "Divergências
   propostas" de um **shell HTML estático de primeira pintura** de ~30 KB: nome do app, uma
   linha de carregamento e nada mais. Com ele, quem está em 3G vê "carregando" em 2 s em vez de
   página branca por 20 s. É a diferença entre "lento" e "quebrado".
3. **A7 precisa comparar CanvasKit e SkWasm** e escolher pelo número medido, não pelo padrão.

| | |
| --- | --- |
| Medir | DevTools, perfil declarado, `performance.mark`/`measure` em volta do primeiro `addTimingsCallback` com `rasterSpan > 0`; repetir 5 vezes e reportar a mediana |
| Onde no CI | **não roda no CI** (não é determinístico). Roda em T-7.3a e é **remediída em S8**; o resultado vai no corpo do relatório do gate |

#### B5 — botão "Contatar"

| | |
| --- | --- |
| **Alvo p50** | **≤ 1,5 s** do toque até `launchUrl` retornar `true` |
| **Teto p99** | **≤ 4,0 s**, **incluindo** o caso em que o rate limit estourou |
| **Feedback visual** | **≤ 100 ms** entre o toque e o spinner aparecer |

Decomposição do pior caso, que é o número que importa, porque o orçamento tem que caber no pior
caso e não no melhor:

| Etapa | p50 | p95 |
| --- | --- | --- |
| Verificação do Turnstile (`siteverify` do Cloudflare) | 150 ms | 400 ms |
| Insert na tabela de rate limit (2 limites: IP e anúncio) | 30 ms | 80 ms |
| RPC `private.get_listing_contact` (D1) | 40 ms | 120 ms |
| Rede até a borda do Supabase, ida e volta | 150 ms | 250 ms |
| Montagem do `wa.me` e `launchUrl` | 60 ms | 200 ms |
| **Total** | **430 ms** | **1.050 ms** |

Even no p95 há folga de 3× até os 4,0 s. A folga é proposital: em 3G com CGNAT o RTT passa de
150 ms para 300 ms com facilidade, e o orçamento precisa absorver isso sem virar incidente.

Duas decisões que o número força:

1. **O widget do Turnstile é montado no `build` do detalhe, não no toque.** Se o token fosse
   buscado no toque, o toque pagaria +1 RTT (~150 ms) e a Edge Function pagaria outro quando
   validasse. Com o token pré-obtido, o caminho do toque tem só as cinco etapas acima.
2. **O orçamento termina em `launchUrl` retornar `true`, e não em "o WhatsApp abriu".** Abrir
   outro aplicativo é fora do nosso controle e varia por dispositivo; o que o app controla é ter
   disparado a abertura com o número certo. O que está depois é o sistema operacional.

E o caminho **sem** estourar limite é o mesmo código com o mesmo orçamento: a resposta é uniforme
e de tempo constante (D1), então o caso de limite não custa mais — ele custa **menos**, porque
não espera verificação de captcha. A interface mostra a contagem de `retryAfter` (§3.5).

| | |
| --- | --- |
| Medir | `Stopwatch` no bloc, do `add(ContactRequested)` ao retorno do launcher; `p50` e `p99` de 30 execuções em 4G, com token pré-obtido |
| Onde no CI | não roda no CI; automatizável em teste com relógio injetado, mas o **número real** é do gate `S4`, com o dispositivo real e a Edge Function real |

#### B6 — jank ao rolar a lista

| | |
| --- | --- |
| **p95 de tempo de frame** | **≤ 12 ms** |
| **Frames acima de 16,7 ms** | **≤ 5%** |
| **Frames acima de 100 ms** | **0** |
| **Cena** | lista com 200 itens, seeds do T-1.7, build **profile**, device Android físico |

Derivação: 16,7 ms é o orçamento de um frame a 60 Hz, e um frame que o estoura perdeu um vsync.
5% é tolerável porque um pico isolado é normal em scroll real; o que não é normal é 5%
sustentado. **Zero acima de 100 ms** porque um frame de 100 ms é um travamento que o usuário
sente como travamento, e aí ele para de rolar — o que é pior que alguns frames perdidos.

| | |
| --- | --- |
| Medir | `FlutterFrameRecorder` em `app/frame_recorder.dart` usando `SchedulerBinding.instance.addTimingsCallback`, registrando `buildSpan` + `rasterSpan` em `Listenable` e imprimindo p50/p95/p99 e a contagem acima de cada limiar. Só compila em `kDebugMode` ou `kProfileMode` |
| Onde no CI | **não roda no CI** (ver a ressalva abaixo). Medido no `S4`, e **remedição no `S8`** |

**Ressalva honesta, e ela é uma limitação real:** jank **não** é automatizável na v1. Medir
frame de verdade exige `integration_test` rodando em device, e `integration_test` não está na
lista de pacotes e o `INTENT.md` §11 tira e2e da v1. A alternativa seria rodar no CI e medir
por um caminho que não é medição.

O que dá para automatizar, e o CI faz, é um **proxy de causa, não de sintoma**: um widget test
que rola 200 itens contando builds, e falha se um item visível reconstruir mais de duas vezes
por frame, ou se o total de builds passar de 4× o número de itens visíveis. Isso pega tempestão
de rebuild — a causa real de jank em lista — e não pega um `rasterSpan` ruim. Está escrito assim
no teste, e o relatório do gate diz que é proxy.

### 10.3 O que **não** cabe no orçamento

| Item | Por quê | O que fazer |
| --- | --- | --- |
| `flutter_map`, `latlong2` | Fase 9, depois do orçamento | T-9.3 re-baselineia com o delta declarado |
| `geolocator` na web | plugin com interop JS; mesma razão | T-9.2 idem |
| `package:image` na web | excluído **por design** (§B1); entra = orçamento estoura | conditional import, sempre |
| 5ª, 6ª foto | máximo é 4 | T-4.x e A5 recusam no formulário |
| Imagem embutida em base64 | +33% de bytes e o Storage conta o texto | proibido; o budget de payload pega |
| `flutter_service_worker.js`, `version.json` | irrelevantes | ignorados pelo `check_budgets` |
| `lib/l10n/gen/**` no bundle | é código gerado, entra no tamanho do `main.dart.js` | conta normalmente; é por isso que o ARB é enxuto |

### 10.4 O que fazer quando estoura

**Estourar falha o CI. Não avisa.** Decisão, e o motivo é o ciclo vicioso: um budget que só
avisa vira um aviso que já estava no log da build anterior, e o PR seguinte é 5% maior.

A regra, inteira:

1. **Na faixa de aviso** (1,9 MB–2,2 MB): anotação `warning::`, build passa. Existe para dar
   sinal **antes** do precipício.
2. **Entre o alvo e o teto duro** (2,2 MB–2,6 MB): **falha**.
3. **Acima do teto duro** (2,6 MB): **falha**, com mensagem diferente: "orçamento estourado; se
   o crescimento é inevitável, peça re-baseline ao A, não edite o número no mesmo PR".
4. **Aumentar um budget é sempre Major, e só por PR que edita `budgets.yaml` com justificativa no
   corpo, revisado pelo A e com ADR quando a razão é arquitetural.** Não existe caminho para o
   número mudar junto com o código que o estourou — é o que separa orçamento de meta pessoal.
5. **Ordem das correções legítimas:** conditional import → inicialização preguiçosa → remover
   dependência → reduzir o que entra no payload. Nessa ordem, e pela razão de custo: a primeira
   devolve bundle, a última devolve dados.

`tool/check_budgets.dart --explain` imprime, junto da falha, os cinco maiores arquivos de
`build/web/`, para o dev saber para onde olhar. Um budget que falha sem dizer onde é um budget que
a equipe aprende a ignorar.

---

## 11. O que este documento proíbe

Lista curta, explícita, com uma linha de razão cada. Todas as regras de lint e grep acima têm
correspondência aqui — a diferença é que estas são para o revisor citar.

| # | Proibido | Por quê |
| --- | --- | --- |
| 11.1 | `print` e `debugPrint` em qualquer arquivo que vá para produção | `avoid_print` está ligado; log de produção vaza PII — telefone, IP, coordenada — e polui o console de quem está usando o app. Para log, `core/logging/app_logger.dart`, com redação de PII |
| 11.2 | Usar `context` depois de `await` sem checar `mounted` | `use_build_context_synchronously` está ligado; usar o `BuildContext` de um widget descartado lança em runtime, e a tela de contato tem `await` no meio |
| 11.3 | String de interface fora do `.arb` | o produto é pt-BR e o projeto se prepara para outros idiomas; a tela é o único lugar que conhece texto, e `Failure` guarda código, não frase (§3.2) |
| 11.4 | `import 'package:supabase_flutter/…'` fora de `data/` | é o que mantém a camada `data` substituível e o bloc testável; um `SupabaseClient` num widget quebra a §2.1 sem nenhum aviso do compilador |
| 11.5 | `import 'package:flutter/…'` em `domain/` | `domain` é Dart puro de propósito: sem isso o teste de entidade precisa de engine, e a direção da dependência deixa de ser verificável |
| 11.6 | Segredo, `service_role` ou URL de ambiente no repositório | a chave anônima vai parar no bundle web, então é pública por definição; o que não pode ir é nada que não seja publicável, e D1 eliminou o `service_role` justamente para isso |
| 11.7 | `build_runner` e qualquer gerador de código | `INTENT.md` §7 tira da v1; a tentação de "só mais um gerador" sempre termina em um `.g.dart` que ninguém revisa. Exceção única e explícita: `flutter gen-l10n`, que é do SDK (§7.6) |
| 11.8 | `setState` dentro de `presentation/blocs/` | estado fora do fluxograma quebra o `bloc_test` e a ordem de emission; o bloc existe justamente para ser o único dono do estado de tela |
| 11.9 | `Future` sem `await` e sem `unawaited()` explícito | `unawaited_futures` está ligado; o `await` esquecido no caminho de falha deixa a tela em `loading` para sempre, que é o bug do §3.1 |
| 11.10 | Campo novo em entidade, evento ou `Failure` sem entrar em `props` | `BlocBuilder` compara por igualdade; fora de `props`, o `buildWhen` deixa de filtrar e nenhum teste falha |
| 11.11 | Criar tabela, função ou view sem a §1.1 inteira | tabela nova em `public` **nasce** com `select, insert, update, delete` liberados para `anon`; policy não remove grant, e nenhuma revisão em Dart enxerga isso |
| 11.12 | `usecases/` com classe que só delega para o repositório | indireção com nome, arquivo e teste, e nenhuma lógica. Se um caso de uso aparecer, é porque ele tem lógica real (compor validação com upload, por exemplo), e aí A aprova |
| 11.13 | Lógica de negócio dentro de um widget | lógica em widget só se testa com widget test, que é o teste mais caro do projeto. A regra corta na fronteira `presentation`/`domain` |
| 11.14 | Ler `location` ou `whatsapp` pelo cliente | `location` nunca é exposta e o telefone só sai pela RPC da borda; um SELECT a mais é um vazamento de PII que o `S1` não vê no diff do Dart |
| 11.15 | Pacote fora da lista de §7.6 | a lista é fechada de propósito: cada dependência é superfície de manutenção e de supply chain, e o `S0`/`S7` audita a cadeia |
| 11.16 | `push` direto na `main` | `main` é protegida, e o caminho de emergência precisa ser um PR, porque é o PR que o revisor lê |
| 11.17 | Envolver arquivo de outra pessoa em um PR de formatação | o diff fica ilegível e o revisor perde o que era para revisar; se o format está errado, é tarefa de `D` e entra sozinha |
| 11.18 | Subir um budget de §10 no mesmo PR que o estourou | essa é a diferença entre orçamento e meta pessoal (§10.4) |
| 11.19 | `toString()` de estado ou `Failure` que exponha PII ou causa técnica | o estado é logável e a árvore de widget o renderiza; `UnknownFailure` é o exemplo do contrato (§3.2) |
| 11.20 | `// TODO` sem link de issue | `todo` está em `ignore` no analyzer, então ninguém é avisado; a regra vira Major na revisão, e o link é o que prova que alguém pensou no assunto |

---

## Divergências propostas

Pontos em que este documento se afasta do que foi pedido ou do que estava em aberto. Nenhum deles
foi aplicado por conta própria onde o outro artefato é dono da decisão.

### D-a · Nomenclatura de constantes: `lowerCamelCase`, não `PascalCase`

O pedido original para A0 era "constantes em `PascalCase` (padrão Dart), e por que não
`SCREAMING_SNAKE`". Adotei `lowerCamelCase`, que é o padrão Dart **de verdade**: o Effective Dart
diz "PREFER using `lowerCamelCase` for constant names", e o lint `constant_identifier_names`
vem ligado no `flutter_lints` — ou seja, `PascalCase` geraria aviso em toda constante do
projeto. E os três motivos que o Dart deu para abandonar o `SCREAMING_CAPS` são exatamente os
nossos casos: valor de enum parece constante de C (`FRUIT`), constante muda de `const` para
`final` e o nome precisaria mudar junto, e `enum.values` é `const` e minúsculo.

Se a preferência for `PascalCase` mesmo assim, é uma linha de configuração
(`constant_identifier_names: false`) e uma linha em `§7.3` — mas perde a proteção do lint.

### D-b · Badge de cobertura: sem badge automático na v1

T-7.5 exige "badge com threshold real". O threshold existe e é §8.5. O **badge** foi retirado: a
implementação natural exige um job com `permissions: contents: write` escrevendo no repositório,
ou uma action de terceiros com SHA fixado, para mostrar um número que o CI já está impondo e
falhando. Numa cadeia de entrega que o `S0` e o `S7` auditam, adicionar um token com escrita
pelo peso de um badge é o tradeoff errado.

Proposta alternativa, e ela é o que peço que o A confirme: o número fica em `docs/quality.md`,
escrito no passo de release, e o README mostra o do **último release** — que envelhece de forma
**visível**, o que é melhor do que um badge autoatualizado que ninguém compara com nada. Se o A
preferir o badge, ele precisa vir com o `permissions: contents: write` justificado por escrito.

### D-c · `package:image` fora do bundle web, por conditional import

`INTENT.md` §6.7 fixa `image` para compressão, e a §5 do plano já registra que `compute` roda na
main thread na web. Este documento acrescenta a terceira peça: **import condicional**, com
`package:image` apenas no caminho de Android e `canvas` no caminho da web. Sem isso, o budget de
B1 é inalcançável e o T-5.1 de A5 precisa ser reescrito (§B1). A5 precisa saber disso antes de
especificar T-5.1.

### D-d · `ContactUnavailableFailure` e a "mensagem clara" de T-4.5

T-4.5 diz "o limite exibe mensagem clara". D1 exige resposta uniforme entre "anúncio inexistente"
e "limite estourado", para não virar oráculo. As duas coisas não podem ser satisfeitas ao
mesmo tempo: uma mensagem que distingue os dois casos **é** o oráculo.

A solução proposta é `ContactUnavailableFailure`: os dois casos viram a mesma falha, com uma
mensagem verdadeira nos dois ("não foi possível liberar o contato agora, tente mais tarde") e
`retryAfter` para a contagem. A1/D1 precisam confirmar que o corpo uniforme pode carregar
`retry_after` sem virar oráculo — é o único campo de que o cliente precisa, e ele é igual nos dois
casos. Se A1 discordar, T-4.5 precisa de spec de tela nova.

### D-e · `domain/usecases/` não é criada

`INTENT.md` §7 lista "casos de uso" na pasta `domain`. Nenhuma task do plano produz um caso de
uso, e um caso de uso que só delega para o repositório é indireção com nome, arquivo e teste, e
nenhuma lógica. Decidi **não criar a pasta**, e criar quando houver lógica real (§11.12). Se A
preferir manter a pasta vazia na árvore, é cosmético.

### D-f · `applicationId` — decidedo pelo dono do projeto, não por A0

O `applicationId` é **`com.projetodaro.ca`**, escolhido pelo dono do projeto antes do T-0.1,
e **não** é uma escolha de A0 como este documento supunha. É deliberadamente genérico e
reversível: a verificação de disponibilidade da marca é a task humana T-7.0, que roda **antes**
do deploy, e nada aqui presume que "Direto da Roça" esté livre. Só vira problema na Play Store,
que está fora da v1 (§3). O nome do pacote Dart (`direto_da_roca`) não muda.

### D-g · Enum de categorias provisório no exemplo

O `ListingCategory` neste documento tem seis valores e existe para compilar o exemplo. A lista
canônica é de A1, e `INTENT.md` §13 deixa a lista em aberto. Mel, queijo e cárneos **não** entram,
porque `INTENT.md` §3 é explícito sobre produtos processados. Quando A1 fechar, o único lugar a
mudar é o `enum` e o `switch` do mapper.

### D-h · Tempo e jank não são gate do CI

B4, B5 e B6 são medidos nos gates, não no CI, porque segundos e frames não são determinísticos em
runner compartilhado, e jank de verdade exigiria `integration_test` em device — que está fora da
v1 (`INTENT.md` §11). O CI mede byte, que é reprodutível. O que o CI faz por jank é um proxy de
causa (contagem de builds por frame), declarado como proxy no próprio teste. Se o A quiser jank no
CI, precisa aprovar `integration_test` como pacote e abrir uma task de runner de device.

### D-i · `perf/` nos prefixos de branch

O pedido listou `feat/`, `fix/`, `chore/`, `docs/`. Acrescentei `perf/`, porque trabalho de
performance neste projeto tem critério de aceite numeral (§10) e forçá-lo dentro de `feat/`
mistura duas coisas na história. Mudar é uma linha em §9.2.

### D-j · Shell HTML de primeira pintura (proposta, não decisão)

B4 mostra que em 3G o primeiro conteúdo leva ~20 s, e ~70% disso é o renderizador, que não é
nosso. Um shell estático de ~30 KB — nome do app e uma linha de carregamento — transforma
"quebrado" em "lentinho" para quem está em 3G, que no Brasil não é nulo. Custa pouco e é
incompatível com nada. Mas mexe no deploy, que é território de A7, então fica aqui como
proposta e não como decisão.

---

## Checklist de conformidade

O que um PR novo precisa satisfazer. É a soma deste documento, e é o que o revisor confere.

- [ ] Arquivo novo em pasta que já tem README, e o README removido se era placeholder
- [ ] `domain` sem `package:flutter` e sem `package:supabase_flutter`
- [ ] `presentation` sem import de `data/repositories/*_impl.dart` e sem `SupabaseClient`
- [ ] Toda chamada de rede devolve `Result<T, Failure>` e o handler tem `switch` exaustivo
- [ ] Toda falha nova tem caso no `switch` de `core/l10n/failure_messages.dart` e chave no ARB
- [ ] Toda entidade, evento e `Failure` com `props` completo
- [ ] Estado com carga de dados: `enum status` + fábricas nomeadas, **sem** `copyWith`
- [ ] Handler interrompível com guard de geração e `emit.isDone`
- [ ] `props` de `ValidationFailure` sem texto, e `merge` em vez de `copyWith`
- [ ] Nenhum texto de interface fora do `.arb`
- [ ] Nenhum `print`, nenhum segredo, nenhum `--dart-define` com valor
- [ ] Testes: comportamento e não implementação; `registerFallbackValue` para todo tipo não nulável
- [ ] SQL novo com a §1.1 inteira, e negação provada com a asserção do mecanismo
- [ ] Budget estourado: falhou, e o número **não** subiu no mesmo PR
- [ ] PR com menos de 400 linhas alteradas, ou divisão pedida ao A
- [ ] Conventional Commit com tipo em inglês, assunto em português, ≤ 72 caracteres

## Decisões que este documento fixa, resumidas

Para quem quiser a versão de uma página:

| # | Decisão | Onde |
| --- | --- | --- |
| 1 | `README.md` por pasta de feature; sem `.gitkeep`; subpasta nasce com o primeiro arquivo | §1.1 |
| 2 | `presentation → domain ← data`, com `core` abaixo e duas partes de `core` sabedoras de Flutter isoladas | §2.1, §2.4 |
| 3 | `Result<T, Failure>` no contrato de `domain`; exceção só dentro de `data` | §3.1 |
| 4 | 13 `Failure` seladas, com `isRetryable` e sem texto; `ContactUnavailableFailure` colapsa not-found e rate-limit | §3.2 |
| 5 | Repositório: interface em `domain`, `Result` de volta, `pageSize` fora da assinatura, paginação por cursor | §4 |
| 6 | Estado com dados é classe única + `enum status` + fábricas, sem `copyWith`; erro é campo, nunca classe | §5.2 |
| 7 | Transformer próprio em `core/async` + guard de geração; sem `bloc_concurrency`; sem cancelamento de I/O | §6 |
| 8 | Constantes em `lowerCamelCase`; widget sem sufixo `Widget`; `…Page` e `…View` como exceções | §7.2, §7.3 |
| 9 | `sdk: ^3.9.0`, tall style, `flutter_lints` 6 mais um bloco curto justificado | §7.4, §7.5 |
| 10 | Cobertura: 80% global com piso por pasta, e a regra que vale é não poder cair | §8.5 |
| 11 | Squash merge, `main` protegida, PR < 400 linhas | §9 |
| 12 | Os budgets de §10 falham o CI, e subir número exige PR separado com ADR | §10.4 |
| 13 | `package:image` fora do bundle web por conditional import | §10.2 (B1) |

