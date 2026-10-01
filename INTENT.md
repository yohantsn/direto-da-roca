# Direto da Roça — Intent do Projeto

> Documento de definição do projeto. Reúne o problema, o escopo da v1 e as decisões técnicas tomadas. Deve viver no repositório (ex.: `docs/INTENT.md`) e ser atualizado quando uma decisão mudar.

**Status:** definição inicial · **Idioma do produto:** pt-BR · **Plataformas:** Android + Web

---

## 1. Problema

Muita gente tem frutas, verduras ou ervas em casa (um pé de laranja, jabuticaba, limão, uma horta) que produzem mais do que o consumido, e parte disso se perde. Do outro lado, há pessoas na mesma vizinhança que gostariam de comprar, ganhar ou trocar alimento fresco e local, mas não sabem quem tem.

## 2. Objetivo

Conectar quem tem excedente de produção com quem quer consumir, **no mesmo bairro/cidade**, de forma simples. A plataforma só faz a ponte: o contato é feito por WhatsApp, e **entrega e pagamento são combinados diretamente entre as partes**.

Objetivo secundário: servir como **projeto de portfólio** demonstrando arquitetura limpa, segurança no backend, testes e CI/CD em uma stack Flutter + Supabase.

## 3. Não-objetivos (v1)

- Monetização (sem comissão, anúncios ou planos pagos).
- Pagamento, entrega ou logística dentro do app.
- Chat interno, avaliações e notificações push.
- Venda de produtos processados/inspecionados (queijo, carne, mel, embutidos), que exigem regulação sanitária.
- iOS.

## 4. Usuários

| Perfil | Autenticação | O que faz |
|---|---|---|
| **Comprador** (quem procura) | Não precisa de login | Navega, filtra, vê o detalhe e contata o vendedor |
| **Vendedor** (quem anuncia) | Google ou link mágico por e-mail | Publica e gerencia os próprios anúncios |

## 5. Escopo da v1

**Fluxo central: anunciar → encontrar → contatar.**

- **Anunciar:** título, foto, categoria, quantidade aproximada, tipo (**doação / venda / troca**), cidade/bairro e contato via WhatsApp.
- **Gerir anúncios:** editar, renovar, marcar como "já acabou" e apagar.
- **Encontrar:** lista com filtro por cidade/bairro e categoria, e busca simples.
- **Detalhe do anúncio** com link compartilhável (deep link/URL na web).
- **Contatar:** botão que abre `wa.me` com mensagem pré-preenchida.
- **Expiração automática** do anúncio (prazo inicial sugerido: 7 a 14 dias).
- **Denunciar anúncio** (inclusive por visitante anônimo, com proteção contra abuso).

**Fase 2 (ainda dentro do escopo do projeto):** mapa com `flutter_map` e busca por raio de proximidade (PostGIS).

## 6. Decisões técnicas

| # | Tema | Decisão |
|---|---|---|
| 1 | Plataformas | Android + Web |
| 2 | Arquitetura | Feature-first com `data / domain / presentation` |
| 3 | Navegação | Navigator 2.0 (Router API) implementado manualmente |
| 4 | Injeção de dependência | `RepositoryProvider` / `BlocProvider` do `flutter_bloc`, sem pacote extra |
| 5 | Modelos e estados | `equatable` + classes seladas do Dart 3; conversão JSON manual |
| 6 | Mapa e localização | `flutter_map` + OpenStreetMap + `geolocator` |
| 7 | Imagens | `image_picker` + `image` (Dart puro; compressão em `Isolate`/`compute`) |
| 8 | Autenticação (vendedor) | Supabase Auth: Google + link mágico por e-mail |
| 9 | Testes | Unitários + BLoC (`bloc_test`, `mocktail`) + widget tests |
| 10 | CI/CD e hospedagem web | GitHub Actions; Cloudflare Pages (build do Flutter nas próprias Actions) |
| 11 | Ambientes | 2 projetos Supabase na nuvem: dev e produção |
| 12 | Idioma | pt-BR com arquivos ARB (`flutter_localizations` / `intl`), preparado para outros idiomas |
| 13 | Gerenciamento de estado | BLoC (`flutter_bloc`) |
| 14 | Backend | Supabase (Postgres + PostGIS, Auth, Storage, RLS, Edge Functions, `pg_cron`) |
| 15 | Nome | Direto da Roça (verificar disponibilidade, ver seção 13) |

## 7. Arquitetura

### Estrutura de pastas (feature-first)

```
lib/
  app/                    # bootstrap, tema, router, providers globais
  core/                   # erros, utilitários, constantes, config de ambiente
  features/
    auth/
      data/               # datasources Supabase, implementação de repositórios
      domain/             # entidades, contratos de repositório, casos de uso
      presentation/       # blocs, páginas, widgets
    listings/
      data/
      domain/
      presentation/
    reports/
      data/
      domain/
      presentation/
    map/                  # fase 2
  l10n/                   # arquivos .arb
```

### Princípios

- Os BLoCs dependem de **interfaces** de repositório (facilita mocks e testes); só a camada `data` conhece o Supabase.
- Estados e eventos como `sealed class` com `switch` exaustivo.
- Navegação (Router API) com o estado de rota **modelado de forma testável** (ex.: um `Cubit` de navegação), tratando também o retorno do link mágico.
- Sem geração de código na v1 (sem `build_runner`), o que simplifica a pipeline.
- Strings sempre via localização, nunca dentro dos widgets.

## 8. Modelo de dados (inicial)

- **`profiles`**: `id` (= `auth.users.id`), `display_name`, `whatsapp` (com consentimento explícito de exposição), `created_at`.
- **`listings`**: `id`, `owner_id`, `title`, `description`, `category`, `kind` (`donation | sale | swap`), `quantity_text`, `price` (opcional), `city`, `neighborhood`, `location` (`geography(Point)` exata, **nunca exposta**), `public_area` (coordenada aproximada/bairro), `status` (`active | finished | expired | removed`), `expires_at`, `created_at`.
- **`listing_images`**: `id`, `listing_id`, `storage_path`, `position`.
- **`reports`**: `id`, `listing_id`, `reason`, `details`, `created_at`, protegido contra abuso (rate limit/captcha via Edge Function).

## 9. Segurança e privacidade

- **RLS ativado em todas as tabelas.** Leitura pública (`anon`) apenas de anúncios `active`; escrita só pelo dono autenticado; limite de anúncios por usuário.
- **Telefone fora da listagem:** o WhatsApp do vendedor só é retornado por RPC quando o comprador toca em "Contatar", com limite de requisições, para dificultar coleta em massa.
- **Localização:** guardar o ponto exato apenas se necessário; expor sempre bairro/área aproximada.
- **LGPD:** consentimento do vendedor para exibir o contato, política de privacidade e opção de apagar a conta e os dados.
- **Storage:** limites de tamanho e tipo de arquivo; políticas por dono.
- **Segredos:** nunca no repositório; GitHub Secrets e `--dart-define` / `--dart-define-from-file` por ambiente.
- **Testes de RLS** em SQL (ex.: pgTAP via Supabase CLI), já que a segurança mora no banco.

## 10. Ambientes e CI/CD

**Ambientes:** projeto Supabase **dev** e projeto Supabase **produção**, com configuração injetada por `--dart-define`. Migrações versionadas no repositório (Supabase CLI), aplicadas primeiro em dev e, após merge na `main`, em produção.

**Pipeline (GitHub Actions):**

1. **Pull request:** `dart format --set-exit-if-changed`, `flutter analyze`, `flutter test` (com cobertura), build de verificação (web e Android).
2. **Merge na `main`:** build web de release e deploy no **Cloudflare Pages**; aplicação das migrações em produção.
3. **Tags:** gerar APK/AAB como artefato. Publicação na Play Store fica para uma etapa posterior.
4. **Extras:** Dependabot, badge de cobertura, pré-visualização por PR na web.

**Hospedagem web:** Cloudflare Pages. O host deve servir `index.html` para rotas desconhecidas (necessário para URLs como `/anuncio/123` com navegação manual). Confirmar no primeiro deploy.

## 11. Testes

- **Unitários** (entidades, mapeamentos, casos de uso).
- **BLoC** com `bloc_test` + `mocktail`.
- **Widget tests** das telas principais (lista, formulário de anúncio, detalhe).
- **RLS/SQL** com pgTAP (recomendado).
- Fora da v1: testes de integração ponta a ponta com Supabase local.

## 12. Roadmap sugerido

1. **Semanas 1–2:** setup do repositório e pipeline, modelagem no Supabase (com RLS), autenticação do vendedor, criar/listar anúncios, Router API.
2. **Semana 3:** imagens (seleção, compressão, upload), filtros, detalhe com deep link, contato por WhatsApp.
3. **Semana 4:** expiração automática (`pg_cron`), denúncia, gestão dos próprios anúncios, i18n completo.
4. **Depois:** mapa e proximidade (PostGIS), deploy web em produção, README e case study, uso real em um bairro/grupo.

## 13. Pontos em aberto

- [ ] Verificar disponibilidade do nome (Play Store, domínio, redes sociais e marcas existentes).
- [ ] Escolher provedor de tiles para produção (tiles públicos do OSM não são adequados para uso contínuo).
- [ ] Configurar SMTP próprio para o link mágico antes de produção.
- [ ] Confirmar limites do plano gratuito do Supabase (projetos ativos, pausa por inatividade).
- [ ] Definir prazo padrão de expiração e lista de categorias.
- [ ] Definir região/bairro do lançamento piloto.
- [ ] Confirmar o comportamento de rotas do Cloudflare Pages (fallback para `index.html`).

## 14. Riscos

| Risco | Mitigação |
|---|---|
| Pouca densidade de anúncios na mesma região | Lançar em **um** bairro/cidade, com grupo piloto |
| Anúncio sem resposta ou desatualizado | Expiração automática e opção "já acabou" |
| Abuso, spam e golpes | RLS, limites por usuário, rate limit, denúncia |
| Coleta em massa de telefones | Telefone só via RPC ao tocar em "Contatar", com limite |
| Escopo crescendo demais | Manter a v1 no fluxo anunciar → encontrar → contatar |
| Complexidade da navegação manual | Reservar tempo extra e testar deep links desde cedo |

## 15. Critérios de sucesso (portfólio)

- App web no ar e build Android disponível como artefato.
- Pipeline verde com format, análise e testes em cada PR.
- README claro, com decisões documentadas (por que WhatsApp, por que expiração, por que RLS).
- Alguns usuários reais e métricas simples (anúncios criados, contatos gerados).
