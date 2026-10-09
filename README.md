# Arquitectura inicial — Fertilidade do Solo, Cafeicultura e Agrofloresta
> **Escopo desta entrega:** conjunto de componentes na pasta raiz de um projecto Google Apps Script. O foco é a arquitectura, os contratos e os cabeçalhos funcionais; os fluxos completos de negócio serão implementados por fases.

## 🎯 Otimização Mobile (2026-10-09)

Este projeto foi completamente otimizado para dispositivos móveis, com foco principal em **Samsung S20 em orientação vertical** (viewport 360-412px).

### Características Mobile-First

**Responsividade Completa:**
- Tela de login adaptável com 4 breakpoints (<400px, 400-640px, 640-900px, >900px)
- Topbar responsivo com logo ajustável (32px mobile, 36px tablet, 44px desktop)
- Sidebar deslizante com menu hambúrguer em dispositivos móveis
- Cards e grids adaptáveis (minmax(280px, 1fr))
- Tabelas com scroll horizontal suave

**Otimizações de UX Mobile:**
- Inputs com font-size 16px (previne zoom automático no iOS/Android)
- Touch targets mínimos de 44px (Apple/Google guidelines)
- Textos longos com quebra automática (word-wrap, overflow-wrap, hyphens)
- Toast notifications full-width em mobile
- Webkit-overflow-scrolling: touch para scroll suave

**Estabilidade Android:**
- CSS simplificado para prevenir crash `render_process_gone`
- Position sticky removido do topbar
- Transitions simplificadas
- Box-shadows otimizadas
- Meta tags otimizadas (maximum-scale=5, mobile-web-app-capable)

### Documentos de Teste

- **[VERIFICACAO_MOBILE_S20.md](VERIFICACAO_MOBILE_S20.md)** - Guia completo de verificação mobile com 10 seções de checklist
- **[README_TESTES.md](README_TESTES.md)** - Infraestrutura de testes automatizados
- **[INICIO_RAPIDO_TESTE.md](INICIO_RAPIDO_TESTE.md)** - Guia rápido para começar a testar
- **[TESTE_PRODUCAO.md](TESTE_PRODUCAO.md)** - Guia de testes em produção

### Comandos de Deploy

```powershell
# Deploy para produção
clasp push

# Abrir webapp no navegador
clasp open --webapp
```

### Testes Locais (Desenvolvimento)

```powershell
# Instalar dependências
npm install

# Iniciar servidor mock
node tools/test-server.js

# Executar testes automatizados
python -m pytest test_user_workflows.py -v
```

**Ferramentas auxiliares:**
- `.env.example` - Template de variáveis de ambiente
- `test-prod.ps1` - Script PowerShell para testes em produção
- `check-config.ps1` - Validação de configuração
- `tools/test-server.js` - Mock server para testes locais
- `test_user_workflows.py` - Suite pytest (produtor, técnico, admin)
- `analyze-buttons.js` - Análise de botões no código

### Breakpoints Responsivos

| Breakpoint | Viewport | Dispositivo | Características |
|---|---|---|---|
| **XS Mobile** | <400px | Smartphones pequenos | Contexto oculto na login, sidebar full-width |
| **Mobile** | 400-640px | Samsung S20, iPhone 12/13 | Layout coluna única, logo 32px, sidebar 280px |
| **Tablet** | 640-900px | iPad Mini, tablets Android | Grid 2 colunas, logo 36px, sidebar 200px |
| **Desktop** | 900-1024px | Laptops pequenos | Grid 3 colunas, logo 40px, sidebar 260px |
| **Desktop Large** | >1024px | Monitores grandes | Grid 4 colunas, logo 44px, sidebar 280px |

## Decisões estruturais
- **Persistência única:** todo o CRUD passa por `SpreadsheetGateway.gs`, que abre a Google Sheet pelo Script Property `SPREADSHEETS_ID`.
- **Raiz plana:** todos os ficheiros `.gs` e `.html` estão directamente na raiz; não há subpastas de componentes.
- **Camadas:** entrada web/RPC → autenticação/autorização → serviços de domínio → repositories → gateway de Sheets.
- **Domínio:** propriedades, parcelas, SAF, espécies, culturas, amostras, camadas, análises químicas/físicas/biológicas, avaliação visual 0–2, IQV/IQS, manejo, fogo, literatura, comparações e AHP/Fuzzy.
- **Autenticação em texto plano (padrão do projeto):** `AuthService.gs` usa `PLAINTEXT_SHEET`: a senha fica em texto claro na coluna `password_plain` da folha `Users`. `SHEET_HASH` (senha com salt e HMAC-SHA256 iterado) é opcional e fora do padrão; é ativado pela propriedade `AUTH_MODE`. A interface está preparada para futura delegação à conta Google ou IdP.
- **AHP/Fuzzy:** `AhpFuzzyService.gs` é um ponto de extensão inicial; resultados devem permanecer explicáveis, com pesos, critérios, julgamentos e consistência visíveis.
## Configuração inicial
No editor do Apps Script, configure em **Project Settings → Script properties**:

```text
SPREADSHEETS_ID=ID_DA_GOOGLE_SHEET_CENTRAL
AUTH_MODE=PLAINTEXT_SHEET
SESSION_TTL_SECONDS=3600
BOOTSTRAP_ADMIN_USERNAME=admin
BOOTSTRAP_ADMIN_PASSWORD=senha-inicial-forte
```

> `AUTH_MODE` aceita `PLAINTEXT_SHEET` (padrão do projeto) ou `SHEET_HASH` (opcional). Qualquer outro valor faz o login falhar.
> `BOOTSTRAP_ADMIN_*` só é usado para criar o primeiro administrador. Depois de criá-lo, o setup apaga essas duas propriedades sozinho.

> O Apps Script não fornece `process.env` como um servidor Node. Por isso, a variável operacional é representada por **Script Properties**; o nome `SPREADSHEETS_ID` é mantido exactamente como solicitado.
## Folhas previstas
| Folha | Finalidade |
|---|---|
| `Users` | identidade temporária e role |
| `Roles/Permissions` | autorização |
| `Properties/Plots` | contexto rural |
| `AgroforestrySystems/Species/Crops` | arranjos produtivos |
| `SoilSamples/SoilLayers` | amostragem e profundidade |
| `ChemicalAnalyses/PhysicalAnalyses/BiologicalAnalyses` | dados laboratoriais |
| `VisualIndicators/VisualAssessments/FieldObservations` | protocolo visual e IQV |
| `ManagementPractices/FireEvents` | manejo, queimadas e recuperação |
| `LiteratureSources/ReviewSearches/Comparisons` | revisão e evidência |
| `Recommendations` | acções explicáveis |
| `DecisionCriteria/DecisionAlternatives/DecisionJudgments/DecisionResults` | AHP/Fuzzy e cenários |
| `AuditLogs/AppEvents/Settings` | governança técnica |

## Componentes `.gs`
| Ficheiro | Responsabilidade principal | Integrações |
|---|---|---|
| [`AppsScriptEntryPoints.gs`](AppsScriptEntryPoints.gs) | Pontos de entrada web e fachada RPC do projecto. | HtmlTemplateService, AuthService, AuthorizationService, QueryService, DashboardService. |
| [`AppConfig.gs`](AppConfig.gs) | Configuração central, nomes de propriedades e parâmetros de negócio. | EnvironmentService, SchemaService, AhpFuzzyService, UI. |
| [`EnvironmentService.gs`](EnvironmentService.gs) | Leitura validada das propriedades do script que funcionam como ambiente. | PropertiesService, AppConfig, SpreadsheetGateway. |
| [`SheetRegistry.gs`](SheetRegistry.gs) | Catálogo estável das folhas que representam as tabelas do sistema. | SchemaService, SpreadsheetGateway, todos os repositories. |
| [`SchemaService.gs`](SchemaService.gs) | Esquemas e cabeçalhos das folhas que centralizam os dados CRUD. | SheetRegistry, SetupService, SpreadsheetGateway. |
| [`SetupService.gs`](SetupService.gs) | Inicialização idempotente da Google Sheet central e carga do catálogo de indicadores. | SchemaService, SpreadsheetGateway, VisualIndicatorRepository, PermissionRepository. |
| [`IdService.gs`](IdService.gs) | Geração de identificadores e chaves técnicas portáveis. | Utilities, SpreadsheetGateway, repositories. |
| [`DateService.gs`](DateService.gs) | Normalização de datas e carimbos de auditoria. | Utilities, SpreadsheetGateway, services. |
| [`JsonService.gs`](JsonService.gs) | Serialização segura de campos compostos armazenados em células. | JSON, ValidationService, repositories. |
| [`ErrorService.gs`](ErrorService.gs) | Normalização de erros para a camada RPC e interface. | ResponseService, LoggingService, AppsScriptEntryPoints. |
| [`LoggingService.gs`](LoggingService.gs) | Registo técnico e funcional de operações do sistema. | Logger, SpreadsheetGateway, AuditService. |
| [`AppCacheService.gs`](AppCacheService.gs) | Cache de curta duração para sessões e leituras de catálogo. | CacheService, SessionService, repositories. |
| [`AppLockService.gs`](AppLockService.gs) | Coordenação de escritas concorrentes na Google Sheet. | LockService, SpreadsheetGateway, SetupService. |
| [`ValidationService.gs`](ValidationService.gs) | Validações de domínio e de integridade dos payloads CRUD. | QueryService, repositories, services, UI. |
| [`NormalizationService.gs`](NormalizationService.gs) | Normalização de texto, enumerações e estruturas de entrada. | ValidationService, QueryService, repositories. |
| [`PaginationService.gs`](PaginationService.gs) | Paginação uniforme de listas provenientes da folha. | QueryService, SpreadsheetGateway, UI. |
| [`QueryService.gs`](QueryService.gs) | Roteador de consultas e mutações para repositories conhecidos. | EntityRegistry, ValidationService, AuthorizationService, repositories. |
| [`AuditService.gs`](AuditService.gs) | Auditoria funcional das operações de leitura sensível e escrita. | AuditLogsRepository equivalente via SpreadsheetGateway, SessionService. |
| [`AuthorizationService.gs`](AuthorizationService.gs) | Autorização por permissões funcionais depois da identidade. | AuthService, SessionService, RoleRepository, PermissionRepository. |
| [`AuthService.gs`](AuthService.gs) | Autenticação transitória contra a folha Users com password em texto plano. | UserRepository, SessionService, AuthorizationService. |
| [`SessionService.gs`](SessionService.gs) | Sessões server-side de curta duração para o modo de autenticação transitório. | CacheService, IdService, UserRepository, RoleRepository. |
| [`RateLimitService.gs`](RateLimitService.gs) | Ponto de extensão para limitação de tentativas de login e endpoints caros. | AppCacheService, AuthService, ExportService. |
| [`ResponseService.gs`](ResponseService.gs) | Contrato de resposta uniforme para a camada client-side. | ErrorService, RequestService, AppsScriptEntryPoints. |
| [`HtmlTemplateService.gs`](HtmlTemplateService.gs) | Renderização de templates HTML e includes de componentes na raiz. | HtmlService, AppsScriptEntryPoints, todos os .html. |
| [`BaseRepositoryFactory.gs`](BaseRepositoryFactory.gs) | Fábrica de repositories CRUD com fonte de dados exclusivamente em Sheets. | SpreadsheetGateway, IdService, DateService, ValidationService. |
| [`SpreadsheetGateway.gs`](SpreadsheetGateway.gs) | Gateway único de persistência CRUD na Google Spreadsheet configurada. | EnvironmentService, SchemaService, AppLockService, DateService, IdService. |
| [`EntityRegistry.gs`](EntityRegistry.gs) | Mapa de entidades públicas para repositories de folhas concretas. | QueryService, todos os repositories. |
| [`UserRepository.gs`](UserRepository.gs) | Repository CRUD da entidade User. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`RoleRepository.gs`](RoleRepository.gs) | Repository CRUD da entidade Role. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`PermissionRepository.gs`](PermissionRepository.gs) | Repository CRUD da entidade Permission. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`PropertyRepository.gs`](PropertyRepository.gs) | Repository CRUD da entidade Property. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`PlotRepository.gs`](PlotRepository.gs) | Repository CRUD da entidade Plot. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`AgroforestryRepository.gs`](AgroforestryRepository.gs) | Repository CRUD da entidade Agroforestry system. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`SpeciesRepository.gs`](SpeciesRepository.gs) | Repository CRUD da entidade Species. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`CropRepository.gs`](CropRepository.gs) | Repository CRUD da entidade Crop. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`SoilSampleRepository.gs`](SoilSampleRepository.gs) | Repository CRUD da entidade Soil sample. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`SoilLayerRepository.gs`](SoilLayerRepository.gs) | Repository CRUD da entidade Soil layer. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`ChemicalAnalysisRepository.gs`](ChemicalAnalysisRepository.gs) | Repository CRUD da entidade Chemical analysis. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`PhysicalAnalysisRepository.gs`](PhysicalAnalysisRepository.gs) | Repository CRUD da entidade Physical analysis. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`BiologicalAnalysisRepository.gs`](BiologicalAnalysisRepository.gs) | Repository CRUD da entidade Biological analysis. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`VisualIndicatorRepository.gs`](VisualIndicatorRepository.gs) | Repository CRUD da entidade Visual indicator. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`VisualAssessmentRepository.gs`](VisualAssessmentRepository.gs) | Repository CRUD da entidade Visual assessment. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`FieldObservationRepository.gs`](FieldObservationRepository.gs) | Repository CRUD da entidade Field observation. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`ManagementPracticeRepository.gs`](ManagementPracticeRepository.gs) | Repository CRUD da entidade Management practice. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`FireEventRepository.gs`](FireEventRepository.gs) | Repository CRUD da entidade Fire event. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`LiteratureSourceRepository.gs`](LiteratureSourceRepository.gs) | Repository CRUD da entidade Literature source. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`ReviewSearchRepository.gs`](ReviewSearchRepository.gs) | Repository CRUD da entidade Review search. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`ComparisonRepository.gs`](ComparisonRepository.gs) | Repository CRUD da entidade Literature comparison. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`RecommendationRepository.gs`](RecommendationRepository.gs) | Repository CRUD da entidade Recommendation. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`DecisionCriteriaRepository.gs`](DecisionCriteriaRepository.gs) | Repository CRUD da entidade Decision criterion. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`DecisionAlternativeRepository.gs`](DecisionAlternativeRepository.gs) | Repository CRUD da entidade Decision alternative. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`DecisionJudgmentRepository.gs`](DecisionJudgmentRepository.gs) | Repository CRUD da entidade Decision judgment. | BaseRepositoryFactory, SpreadsheetGateway, QueryService, serviços do domínio. |
| [`DecisionResultRepository.gs`](DecisionResultRepository.gs) | Repository CRUD da entidade Decision result. | BaseRepositoryFactory, SpreadsheetGateway, ScenarioService, ReportService. |
| [`AgroforestryService.gs`](AgroforestryService.gs) | Regras de negócio para sistemas agroflorestais e arranjos de sombra. | AgroforestryRepository, SpeciesRepository, ManagementPracticeRepository. |
| [`SoilAssessmentService.gs`](SoilAssessmentService.gs) | Orquestra amostras, camadas e análises laboratoriais. | SoilSampleRepository, SoilLayerRepository, ChemicalAnalysisRepository, PhysicalAnalysisRepository, BiologicalAnalysisRepository. |
| [`VisualAssessmentService.gs`](VisualAssessmentService.gs) | Casos de uso para avaliação visual participativa da qualidade do solo. | VisualIndicatorRepository, VisualAssessmentRepository, FieldObservationRepository, FertilityIndexService. |
| [`FireImpactService.gs`](FireImpactService.gs) | Avaliação de eventos de fogo e monitorização pós-queimada. | FireEventRepository, VisualAssessmentService, SoilAssessmentService, RecommendationRepository. |
| [`DashboardService.gs`](DashboardService.gs) | Agregação das métricas para o painel principal. | Todos os repositories e FertilityIndexService. |
| [`ExportService.gs`](ExportService.gs) | Exportação controlada de dados e relatórios para CSV/JSON. | QueryService, AuthorizationService, Utilities, ReportService. |
| [`SeedDataService.gs`](SeedDataService.gs) | Carga idempotente de catálogos científicos e de autorização mínimos. | VisualIndicatorRepository, PermissionRepository, DecisionCriteriaRepository. |
| [`HealthCheckService.gs`](HealthCheckService.gs) | Verificações de saúde técnica da implantação. | EnvironmentService, SetupService, SchemaService, SessionService. |
| [`ReportService.gs`](ReportService.gs) | Modelos de relatórios de diagnóstico e decisão para consumo pela UI/exportação. | SoilAssessmentService, VisualAssessmentService, RecommendationRepository, ExportService. |
| [`AhpFuzzyService.gs`](AhpFuzzyService.gs) | Motor extensível de análise multicritério AHP e pontuação fuzzy. | DecisionCriteriaRepository, DecisionJudgmentRepository, DecisionResultRepository conceptual, FertilityIndexService. |

**Total `.gs` na raiz: 111** (a tabela lista os principais)

## Componentes `.html`
| Ficheiro | Responsabilidade principal | Integrações |
|---|---|---|
| [`Index.html`](Index.html) | Documento raiz do web app com meta tags mobile otimizadas. | HtmlTemplateService inclui AppStyles, AppShell e AppClient. |
| [`AppShell.html`](AppShell.html) | Shell visual principal com logo PNG base64 e regiões responsivas. | AppNavigation, LoadingView, Toast, ErrorView, DashboardView. |
| [`AppNavigation.html`](AppNavigation.html) | Navegação por módulos do domínio. | AppClient route handling e AuthorizationService. |
| [`AppStyles.html`](AppStyles.html) | Estilos base mobile-first com 5 breakpoints e CSS simplificado para Android. | Index, todas as vistas HTML. |
| [`AppClient.html`](AppClient.html) | Cliente JavaScript com initSidebarToggle() para menu hambúrguer. | google.script.run, todas as vistas data-route/data-entity. |
| [`LoginView.html`](LoginView.html) | Vista de login responsiva com grid adaptável (4 breakpoints). | AuthService via AppClient. |
| [`LoadingView.html`](LoadingView.html) | Estado visual de carregamento. | AppClient. |
| [`ErrorView.html`](ErrorView.html) | Estado visual de erro recuperável. | ErrorService/ResponseService via AppClient. |
| [`Toast.html`](Toast.html) | Notificações breves de sucesso, aviso e erro. | AppClient. |
| [`Modal.html`](Modal.html) | Modal de resultados (IQV, relatórios de amostra/talhão, relatórios gerais). | AppClient (showModal/renderValue). |
| [`RecordFormView.html`](RecordFormView.html) | Formulário único de cadastro e edição, gerado pelo esquema (`apiSchema(entity).form`). | AppClient, FormSchemaService, QueryService. |
| [`ConfirmDialog.html`](ConfirmDialog.html) | Confirmação visual para remoções e operações de impacto. | Modal, AppClient, QueryService. |
| [`NotFoundView.html`](NotFoundView.html) | Vista para rotas desconhecidas. | AppClient. |
| [`DashboardView.html`](DashboardView.html) | Painel de indicadores do projecto. | DashboardService via apiDashboard, DashboardCards, DashboardFilters, ChartPanel. |
| [`DashboardCards.html`](DashboardCards.html) | Cartões de KPI do painel. | DashboardService e AppClient evento dashboard-loaded. |
| [`DashboardFilters.html`](DashboardFilters.html) | Filtros de painel. | DashboardService e filtros QueryService. |
| [`ChartPanel.html`](ChartPanel.html) | Área de gráficos de tendências e domínios. | DashboardService, FertilityIndexService, biblioteca de gráficos futura. |
| [`PropertyListView.html`](PropertyListView.html) | Lista de propriedades rurais. | PropertyRepository/PropertyService via QueryService. |
| [`PlotListView.html`](PlotListView.html) | Lista de parcelas/unidades de manejo. | PlotRepository, PropertyService. |
| [`AgroforestryListView.html`](AgroforestryListView.html) | Lista de sistemas agroflorestais. | AgroforestryService, AgroforestryRepository. |
| [`SpeciesCatalogView.html`](SpeciesCatalogView.html) | Catálogo de espécies e funções. | SpeciesRepository, AgroforestryService. |
| [`SoilSampleListView.html`](SoilSampleListView.html) | Lista de amostras de solo. | SoilAssessmentService, SoilSampleRepository. |
| [`VisualAssessmentListView.html`](VisualAssessmentListView.html) | Lista de avaliações visuais. | VisualAssessmentService, VisualAssessmentRepository. |
| [`ManagementPracticeListView.html`](ManagementPracticeListView.html) | Lista de práticas de manejo e acesso ao registro de eventos de fogo. | QueryService via apiList/apiCreate. |
| [`VisualIndicatorCatalog.html`](VisualIndicatorCatalog.html) | Catálogo e guia do protocolo visual. | VisualIndicatorRepository, SeedDataService. |
| [`LiteratureSourceListView.html`](LiteratureSourceListView.html) | Lista de fontes científicas e técnicas. | LiteratureReviewService, LiteratureSourceRepository. |
| [`ComparisonMatrixView.html`](ComparisonMatrixView.html) | Matriz comparativa SAF versus sistema convencional/pastagem/mata. | ComparisonRepository, LiteratureReviewService. |
| [`RecommendationPanel.html`](RecommendationPanel.html) | Painel de recomendações de manejo e recuperação. | RecommendationRepository, FireImpactService, FertilityIndexService. |
| [`UserAdminView.html`](UserAdminView.html) | Administração de utilizadores. | UserService, UserRepository, AuthorizationService. |
| [`RolePermissionView.html`](RolePermissionView.html) | Administração de roles e permissões. | RoleRepository, PermissionRepository, AuthorizationService. |
| [`AuditLogView.html`](AuditLogView.html) | Consulta de auditoria. | AuditService, SpreadsheetGateway. |
| [`ReportsView.html`](ReportsView.html) | Selecção de relatórios e exportações. | ReportService, ExportService. |
| [`SettingsView.html`](SettingsView.html) | Configurações operacionais não secretas. | HealthCheckService, SetupService, EnvironmentService. |
| [`HelpView.html`](HelpView.html) | Ajuda funcional e protocolos de campo. | VisualIndicatorCatalog, FireImpactService, LiteratureReviewService. |
| [`AboutView.html`](AboutView.html) | Sobre o projecto e fontes de inspiração. | README, LiteratureSources. |
| [`DecisionScenarioView.html`](DecisionScenarioView.html) | Vista de cenários de manejo. | ScenarioService, DecisionAlternativeRepository. |
| [`DecisionCriteriaView.html`](DecisionCriteriaView.html) | Catálogo de critérios de decisão. | DecisionCriteriaRepository, AhpFuzzyService. |
| [`AhpMatrixForm.html`](AhpMatrixForm.html) | Formulário de comparações pareadas AHP. | DecisionJudgmentRepository, AhpFuzzyService. |
| [`DecisionResultView.html`](DecisionResultView.html) | Resultado AHP/Fuzzy explicável. | ScenarioService, AhpFuzzyService, DecisionResultRepository conceptual. |
| [`FieldProtocolView.html`](FieldProtocolView.html) | Protocolo de visita e recolha de campo. | SoilAssessmentService, VisualAssessmentService. |

**Total `.html`: 45**

## Fluxos prioritários sugeridos

1. Configurar as propriedades acima e executar `SetupService.setupProject()`. Na ordem: cria as folhas, semeia os papéis `ADMIN`, `TECNICO` e `PRODUTOR`, grava a matriz de permissões (`DEFAULT_ROLE_PERMISSIONS`, em `SetupService.gs`), semeia os catálogos e cria o primeiro administrador se ainda não houver usuário ativo. Pode ser executado de novo sem duplicar nada. Política padrão (conservadora): `TECNICO` e `PRODUTOR` só leem e gravam entidades com escopo por grupo ou por dono; análises de solo, monitoramento ambiental, fauna, resumos, recomendações e o dashboard ficam só com o `ADMIN` até ganharem escopo.
2. Entrar com o administrador e trocar a senha inicial.
3. Ligar o formulário de login ao provider transitório e, em seguida, substituir o provider por identidade Google/IdP.
4. Implementar primeiro o ciclo Property → Plot → SoilSample → SoilLayer.
5. Implementar Chemical/Physical/Biological analyses e o protocolo VisualAssessment com scores 0–2.
6. Activar IQV relativo, referência mata/área de referência e comparação SAF/SCC.
7. Adicionar práticas de manejo, eventos de fogo, monitorização pós-fogo e recomendações de recuperação.
8. Activar LiteratureReview e ComparisonMatrix para guardar a origem de cada evidência.
9. Implementar AHP/Fuzzy com matriz validada, CR, pesos e análise de sensibilidade.

## Boas práticas e limites

### Segurança e Autenticação

- A password em texto plano existe **apenas porque foi explicitamente solicitada para esta fase**. Não é recomendada para produção.
- `password_plain`, tokens de sessão e IDs sensíveis são omitidos de logs, exportações e respostas seguras.
- A UI nunca acede à SpreadsheetApp; a folha é uma base de dados simples, com `SpreadsheetGateway` como ponto único de persistência.

### Compatibilidade Mobile

- **Testado em:** Samsung S20 (Android), viewport 360-412px vertical
- **Navegadores:** Chrome Android, Safari iOS (iOS 13+)
- **Prevenção de zoom:** Inputs com font-size mínimo de 16px
- **Touch targets:** Botões e links com mínimo 44px (WCAG 2.1 AAA)
- **Performance:** CSS simplificado para prevenir crash em Chrome Android
- **Limitações conhecidas:** 
  - Position sticky removido por instabilidade em Android
  - Transitions simplificadas para melhor performance
  - Maximum-scale=5 (permite zoom até 500%)

### Responsabilidades e Limites Técnicos

- Os resultados de fertilidade/qualidade do solo são apoio técnico; não substituem interpretação laboratorial, agronómica, ambiental ou legal.
- Registos de fogo não autorizam queimadas nem substituem licenças, normas ou profissionais treinados.
- Os textos de referência sugerem indicadores visuais como compactação, infiltração, estabilidade de agregados, erosão, estrutura, serapilheira, fauna, odor/cor, actividade microbiológica, raízes, espontâneas, cobertura, pedregosidade e profundidade; estes encontram-se representados no catálogo de seed.
- Os materiais também sustentam a integração de atributos químicos, físicos e biológicos, comparação SAF/SCC, efeitos de fogo e decisão multicritério; cada tema possui camada própria na arquitectura.

---

## 📱 Verificação Mobile

Após o deploy, siga o **[VERIFICACAO_MOBILE_S20.md](VERIFICACAO_MOBILE_S20.md)** para verificar:
- ✅ Tela de login responsiva
- ✅ Sidebar com menu hambúrguer
- ✅ Sem zoom automático em inputs
- ✅ Touch targets adequados
- ✅ Sem crash render_process_gone
- ✅ Textos quebram linha corretamente

**Reportar problemas com:**
1. Screenshot da tela
2. Viewport dimensions (chrome://inspect)
3. Console errors (se houver)
4. Comportamento esperado vs. atual
