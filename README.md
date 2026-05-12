# 🏗️ Arquitectura del Proyecto — Laravel API

> **Patrón:** Layered Architecture + Repository Pattern  
> **Framework:** Laravel · **Tipo:** REST API · **Estado:** Refactorización activa

---

## 📋 Tabla de Contenidos

- [🎯 Arquitectura Implementada](#-arquitectura-implementada)
- [⚙️ Principios Aplicados](#️-principios-aplicados)
- [🔄 Cambios Principales](#-cambios-principales)
- [📁 Estructura de Carpetas](#-estructura-de-carpetas)
- [📦 Resumen de Cambios](#-resumen-de-cambios-por-carpeta)
- [🔀 Flujo de Datos](#-flujo-de-datos)
- [🗺️ Mapeo Original → Nuevo](#️-mapeo-original--nuevo)
- [📐 Convención de Nombres](#-convención-de-nombres)
- [🚨 Regla de Oro](#-regla-de-oro)

---

## 🎯 Arquitectura Implementada

```
┌─────────────────────────────────────────────────────────────────┐
│                        HTTP REQUEST                             │
│                             ↓                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  CONTROLLER  —  delega, no contiene lógica de negocio   │   │
│   └──────────────────────┬──────────────────────────────────┘   │
│                          ↓                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  FORM REQUEST  —  validación y sanitización de entrada  │   │
│   └──────────────────────┬──────────────────────────────────┘   │
│                          ↓                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  SERVICE  —  lógica de negocio, orquesta operaciones    │   │
│   └──────────────────────┬──────────────────────────────────┘   │
│                          ↓                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  REPOSITORY (Interface + Impl.)  —  acceso a datos      │   │
│   └──────────────────────┬──────────────────────────────────┘   │
│                          ↓                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  MODEL (Eloquent)  —  mapeo ORM                         │   │
│   └──────────────────────┬──────────────────────────────────┘   │
│                          ↓                                       │
│              Base de datos / Stored Procedures                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## ⚙️ Principios Aplicados

| Principio | Descripción |
|-----------|-------------|
| **SRP** — Single Responsibility | Cada clase tiene una sola razón de cambio |
| **DIP** — Dependency Inversion | Services dependen de interfaces, no de implementaciones concretas |
| **DRY** — Don't Repeat Yourself | Lógica compartida entre roles centralizada en Services genéricos |
| **Thin Controllers** | Controllers solo reciben, delegan al Service y devuelven respuesta |
| **DTOs por dominio** | Un DTO por concepto de negocio, no uno por rol de usuario |

---

## 🔄 Cambios Principales

1. ✅ Se agrega la capa `Repositories/` con interfaces y sus implementaciones
2. ✅ Se eliminan Traits de lógica y se convierten en clases inyectables
3. ✅ Los DTOs se unifican por dominio (no duplicados por rol)
4. ✅ Los Services se desacoplan de Eloquent vía Repositories
5. ✅ Los Controllers se agrupan por dominio de negocio, no por rol
6. ✅ Se agrega la carpeta `Contracts/` para todas las interfaces
7. ✅ Los Mappers se consolidan bajo su módulo correspondiente

---

## 📁 Estructura de Carpetas

### 📂 `app/Console/`

```
app/
└── Console/
    ├── Commands/
    │   ├── ProcessAttendanceCommand.php
    │   ├── SyncConnectionTimesCommand.php
    │   └── PruneTokensCommand.php
    └── Kernel.php
```

> ⚠️ Los comandos se nombran por lo que hacen, no como `"comandoPrueba"`. Cada comando tiene una responsabilidad clara y documentada.

---

### 📂 `app/Contracts/` — ⭐ NUEVA CAPA

> Interfaces para todos los Repositories y Services.
> Permite inyección de dependencias y facilita testing con mocks.

```
app/
└── Contracts/
    ├── Repositories/
    │   ├── Auth/
    │   │   ├── UserRepositoryInterface.php
    │   │   └── TokenRepositoryInterface.php
    │   ├── Attendance/
    │   │   ├── AttendanceRepositoryInterface.php
    │   │   └── AbsenceRepositoryInterface.php
    │   ├── ConnectionTime/
    │   │   └── ConnectionTimeRepositoryInterface.php
    │   ├── VoiceTime/
    │   │   └── VoiceTimeRepositoryInterface.php
    │   ├── Recruitment/
    │   │   ├── ApplierRepositoryInterface.php
    │   │   ├── InterviewRepositoryInterface.php
    │   │   ├── PublicationRepositoryInterface.php
    │   │   └── ProfileRepositoryInterface.php
    │   ├── Organization/
    │   │   ├── AreaRepositoryInterface.php
    │   │   ├── DivisionRepositoryInterface.php
    │   │   └── TeamRepositoryInterface.php
    │   ├── Communication/
    │   │   ├── MessageRepositoryInterface.php
    │   │   └── NotificationRepositoryInterface.php
    │   └── Shared/
    │       ├── StoredProcedureInterface.php
    │       └── ReportRepositoryInterface.php
    └── Services/
        ├── ImageStorageServiceInterface.php
        ├── PasswordResetServiceInterface.php
        └── NotificationServiceInterface.php
```

---

### 📂 `app/DTOs/` — 🔁 REFACTORIZADO

> DTOs unificados por dominio. Se elimina la duplicación por rol
> (`Colaborador / LiderArea / LiderDivision / Gerencia`).

```
app/
└── DTOs/
    ├── Core/
    │   └── BaseDTO.php                           # antes: BaseValidDTO.php
    ├── Auth/
    │   ├── LoginDTO.php
    │   ├── TokenDTO.php
    │   └── PasswordResetDTO.php
    ├── Attendance/
    │   ├── AttendanceSummaryDTO.php               # unifica Dashboard*DTO por rol
    │   ├── AttendanceCalendarDTO.php              # unifica Calendar*DTO por rol
    │   ├── AttendanceKpiDTO.php                   # unifica Kpi*DTO por rol
    │   ├── AttendanceRankingDTO.php               # unifica Ranking*DTO por rol
    │   ├── AttendanceReportDTO.php                # unifica Reporte*DTO por rol
    │   ├── AttendanceDistributionDTO.php
    │   └── AbsenceDTO.php
    ├── ConnectionTime/
    │   ├── ConnectionTimeSummaryDTO.php           # unifica dashboards por rol
    │   ├── ConnectionTimeDetailDTO.php
    │   ├── ConnectionTimeCreateDTO.php
    │   ├── ConnectionTimeSearchDTO.php
    │   ├── ConnectionTimeKpiDTO.php
    │   ├── ConnectionTimeEvolutionDTO.php
    │   ├── ConnectionTimeRankingDTO.php
    │   ├── ConnectionTimeReportDTO.php
    │   └── ConnectionTimeAgreementDTO.php
    ├── VoiceTime/
    │   ├── VoiceTimeSummaryDTO.php                # unifica dashboards por rol
    │   ├── VoiceTimeCalendarDTO.php
    │   ├── VoiceTimeKpiDTO.php
    │   ├── VoiceTimeRankingDTO.php
    │   ├── VoiceTimeEvolutionDTO.php
    │   ├── VoiceTimeDetailDTO.php
    │   └── VoiceTimeRadialDTO.php
    ├── Recruitment/
    │   ├── ApplierDTO.php
    │   ├── ApplierDetailDTO.php
    │   ├── ApplierScheduleDTO.php
    │   ├── WorkExperienceDTO.php
    │   ├── WorkCredentialDTO.php
    │   ├── InterviewDTO.php
    │   ├── InterviewScheduledDTO.php
    │   ├── EvaluationQuestionDTO.php
    │   ├── EvaluationScoreDTO.php
    │   ├── EvaluationSummaryDTO.php
    │   ├── PublicationDTO.php
    │   ├── PlatformDTO.php
    │   ├── PersonnelRequirementDTO.php
    │   ├── ProfileDescriptionDTO.php
    │   └── DiscResultDTO.php
    ├── Organization/
    │   ├── AreaDTO.php
    │   ├── DivisionDTO.php
    │   ├── TeamDTO.php
    │   └── OrganizationChartDTO.php
    ├── Communication/
    │   ├── MessageDTO.php
    │   ├── MessageFilterDTO.php
    │   ├── SmsDTO.php
    │   └── TelegramDTO.php
    ├── Certificate/
    │   ├── CertificateDTO.php
    │   └── CertificateStructureDTO.php
    ├── Training/
    │   └── TrainingDTO.php
    └── Shared/
        ├── PaginationDTO.php
        ├── DateRangeDTO.php
        └── ApiResponseDTO.php
```

---

### 📂 `app/Enums/`

```
app/
└── Enums/
    ├── AttendanceStatus.php       # antes: AssistStatus.php
    ├── TokenStatus.php
    ├── UserLevel.php
    ├── UserStatus.php
    └── VerificationChannel.php
```

---

### 📂 `app/Exceptions/`

```
app/
└── Exceptions/
    ├── Handler.php
    ├── ApiErrorResponse.php
    ├── ApiExceptionMapper.php
    ├── DatabaseExceptionMapper.php
    ├── ErrorType.php
    ├── Business/
    │   ├── BusinessException.php
    │   ├── NotFoundException.php
    │   └── UnauthorizedException.php
    ├── Validation/
    │   ├── InvalidDtoException.php
    │   ├── DtoBatchValidator.php
    │   ├── DtoFieldError.php
    │   └── ValidatesDto.php
    └── Infrastructure/
        └── StoredProcedureException.php
```

---

### 📂 `app/Http/Controllers/` — 🔁 REFACTORIZADO

> Controllers agrupados por dominio. Máximo 5–7 métodos por controller.
> **Sin lógica de negocio:** solo recibe, delega al Service y devuelve respuesta.

```
app/
└── Http/
    └── Controllers/
        ├── Auth/
        │   ├── LoginController.php
        │   ├── LogoutController.php
        │   ├── PasswordController.php          # agrupa change/forgot/reset
        │   ├── TokenController.php
        │   └── PermissionController.php
        │
        ├── Dashboard/                          # ⭐ 12 controllers → 3
        │   ├── AttendanceDashboardController.php
        │   ├── ConnectionTimeDashboardController.php
        │   └── VoiceTimeDashboardController.php
        │
        ├── Attendance/
        │   ├── AttendanceController.php
        │   ├── AbsenceController.php
        │   └── ScheduleController.php
        │
        ├── ConnectionTime/
        │   └── ConnectionTimeController.php
        │
        ├── VoiceTime/
        │   └── VoiceTimeController.php
        │
        ├── Recruitment/
        │   ├── ApplierController.php
        │   ├── InterviewController.php
        │   ├── PublicationController.php
        │   ├── PlatformController.php
        │   ├── ProfileDescriptionController.php
        │   ├── WorkCredentialController.php
        │   ├── EvaluationController.php
        │   ├── PersonnelRequirementController.php
        │   └── DiscController.php
        │
        ├── Organization/
        │   ├── AreaController.php
        │   ├── DivisionController.php
        │   └── TeamController.php
        │
        ├── Communication/
        │   ├── MessageController.php
        │   ├── SmsController.php
        │   ├── TelegramController.php
        │   ├── WhatsAppController.php
        │   └── BotController.php
        │
        ├── Certificate/
        │   ├── CertificateController.php
        │   └── CertificateStructureController.php
        │
        ├── Training/
        │   └── TrainingController.php
        │
        ├── Users/
        │   ├── UserController.php
        │   ├── RoleController.php
        │   └── ProfileController.php
        │
        ├── Reports/
        │   └── ReportController.php
        │
        ├── Speech/
        │   └── SpeechController.php
        │
        ├── Calls/
        │   ├── CallController.php              # antes: V1CallsController
        │   └── CallModelController.php         # antes: V1CallsModelController
        │
        ├── Cv/
        │   ├── CvController.php               # antes: V1CvController
        │   └── CvKeywordsController.php        # antes: V1CvkeywordsController
        │
        ├── PersonalData/
        │   └── PersonalDataController.php      # antes: V1PersonalDataController
        │
        ├── Material/
        │   └── MaterialController.php          # antes: V1MaterialController
        │
        ├── Turn/
        │   └── TurnController.php              # antes: V1TurnController
        │
        ├── Staff/
        │   └── StaffController.php             # antes: V1StaffController
        │
        ├── Contract/
        │   └── EndContractController.php       # antes: V1EndContractController
        │
        ├── GroupList/
        │   └── GroupListController.php         # antes: V1GroupListController
        │
        ├── Route/
        │   ├── RouteController.php
        │   ├── RouteInputController.php
        │   ├── RouteOutputController.php
        │   └── RouteRoutinesController.php
        │
        ├── Permissions/
        │   ├── PermissionsController.php
        │   └── PermissionToRoleController.php
        │
        ├── Api/
        │   ├── ExternalApiController.php       # antes: V1ApiController
        │   └── ApiDetailController.php
        │
        ├── Integrations/
        │   ├── RustDeskController.php          # antes: V1RustDeskController
        │   └── NgrokController.php             # antes: NgrokIpController
        │
        ├── Util/
        │   └── FileToUrlController.php
        │
        └── Controller.php                      # base controller
```

---

### 📂 `app/Http/Middleware/`

```
app/
└── Http/
    └── Middleware/
        ├── Authenticate.php
        ├── ApiKeyValidate.php
        ├── BotTokenMiddleware.php
        ├── PermissionMiddleware.php
        ├── RoleMiddleware.php
        ├── EncryptCookies.php
        ├── PreventRequestsDuringMaintenance.php
        ├── RedirectIfAuthenticated.php
        ├── TrimStrings.php
        ├── TrustHosts.php
        ├── TrustProxies.php
        ├── ValidateSignature.php
        └── VerifyCsrfToken.php
```

---

### 📂 `app/Http/Requests/` — 🔁 REFACTORIZADO

> FormRequests agrupados por dominio. Base requests reutilizables para
> validaciones comunes (rango de fechas, paginación, etc.)

```
app/
└── Http/
    └── Requests/
        ├── Base/
        │   ├── BaseRequest.php
        │   ├── DateRangeRequest.php            # antes: RangoFechasRequest
        │   ├── YearMonthRequest.php
        │   ├── PaginationRequest.php
        │   └── SpecificDateRequest.php
        ├── Auth/
        │   ├── LoginRequest.php
        │   ├── ChangePasswordRequest.php
        │   ├── ForgotPasswordRequest.php
        │   ├── ValidatePasswordRequest.php
        │   ├── ConfirmTokenRequest.php
        │   └── SelectDocumentRequest.php
        ├── Attendance/
        │   ├── AttendanceSearchRequest.php
        │   ├── AddAttendanceRequest.php
        │   └── UpdateAbsenceRequest.php
        ├── Dashboard/
        │   └── DashboardFilterRequest.php      # unifica todos los DashboardRequest por rol
        ├── ConnectionTime/
        │   ├── ConnectionTimeRequest.php
        │   ├── CreateConnectionTimeRequest.php
        │   ├── UpdateConnectionTimeRequest.php
        │   └── SearchConnectionTimeRequest.php
        ├── Recruitment/
        │   ├── ApplierRequest.php
        │   ├── InterviewRequest.php
        │   ├── PublicationRequest.php
        │   ├── PlatformRequest.php
        │   ├── EvaluationRequest.php
        │   ├── PersonnelRequirementRequest.php
        │   ├── ProfileDescriptionRequest.php
        │   ├── WorkCredentialRequest.php
        │   └── DiscRequest.php
        ├── Organization/
        │   ├── AreaRequest.php
        │   ├── DivisionRequest.php
        │   └── TeamRequest.php
        ├── Communication/
        │   ├── MessageRequest.php
        │   ├── SmsRequest.php
        │   └── TelegramRequest.php
        ├── Certificate/
        │   ├── CertificateRequest.php
        │   └── CertificateStructureRequest.php
        ├── Users/
        │   ├── UserRequest.php
        │   ├── RoleRequest.php
        │   └── UpdateUserRoleRequest.php
        ├── Speech/
        │   ├── SpeechRequest.php
        │   └── ParagraphRequest.php
        ├── Api/
        │   ├── ApiRequest.php
        │   └── ApiDetailRequest.php
        ├── Access/
        │   └── AccessRequest.php
        ├── Filters/
        │   ├── RangeWithAreaRequest.php        # antes: RFAreaRequest
        │   ├── RangeWithDivisionRequest.php    # antes: RFAyDivisionRequest
        │   ├── RangeOptionalRequest.php        # antes: RFOpcionalesRequest
        │   ├── RangeWithSearchRequest.php      # antes: RFAySearchRequest
        │   └── PractitionerRangeRequest.php    # antes: RFPractitionerRequest
        └── Shared/
            ├── SoloIdRequest.php               # antes: Puros/SoloIdRequest
            ├── SoloAreaRequest.php
            ├── SoloPractitionerRequest.php
            └── SoloProfileRequest.php
```

---

### 📂 `app/Mail/`

```
app/
└── Mail/
    ├── SendTokenResetPassword.php
    └── WelcomeMail.php
```

---

### 📂 `app/Mappers/` — 🔁 REFACTORIZADO

> Mappers consolidados por dominio. Ya **no hay duplicados por rol**.
> Cada Mapper transforma entre entidad DB/SP y DTO.

```
app/
└── Mappers/
    ├── AttendanceMapper.php        # unifica: ColaboradorMapper, GerenciaMapper,
    │                               #           LiderAreaMapper, LiderDivisionMapper
    ├── ConnectionTimeMapper.php
    ├── VoiceTimeMapper.php
    ├── RecruitmentMapper.php
    └── OrganizationMapper.php
```

---

### 📂 `app/Models/`

```
app/
└── Models/
    ├── User.php                ├── Publications.php        ├── Speech.php
    ├── Rol.php                 ├── Platform.php            ├── SpeechParagraphs.php
    ├── Area.php                ├── ProfilesDescription.php ├── Meet.php
    ├── Division.php            ├── Profile.php             ├── Certificate.php
    ├── Assist.php              ├── Message.php             ├── BlackList.php
    ├── Absense.php             ├── GenerateMessage.php     ├── ForgotPassword.php
    ├── ConexionTime.php        ├── QuestionMessage.php     ├── TokenPassword.php
    ├── IndividualSchedule.php  ├── GroupList.php           ├── TotalStaff.php
    ├── Holiday.php             ├── Call.php                ├── GeneralData.php
    ├── Turn.php                ├── ModelCalls.php          ├── Country.php
    ├── Applier.php             ├── BulkSms.php             ├── Province.php
    ├── AnswerAppliers.php      │                           ├── Department.php
    ├── AnswerStatus.php        │                           ├── District.php
    ├── Interview.php           │                           ├── Institution.php
    ├── Post.php                │                           ├── Profession.php
    │                           │                           └── OptionAnswer.php
```

---

### 📂 `app/Providers/`

```
app/
└── Providers/
    ├── AppServiceProvider.php
    ├── AuthServiceProvider.php
    ├── BroadcastServiceProvider.php
    ├── EventServiceProvider.php
    ├── RepositoryServiceProvider.php    # ⭐ NUEVO: registra Interface → Implementación
    └── RouteServiceProvider.php
```

**`RepositoryServiceProvider.php`** — registrar bindings en el contenedor IoC:

```php
$this->app->bind(AttendanceRepositoryInterface::class, AttendanceRepository::class);
$this->app->bind(ConnectionTimeRepositoryInterface::class, ConnectionTimeRepository::class);
// ... etc para cada repositorio
```

Registrar en `config/app.php` bajo `'providers'`:

```php
App\Providers\RepositoryServiceProvider::class,
```

---

### 📂 `app/Repositories/` — ⭐ NUEVA CAPA

> Toda la lógica de acceso a datos vive aquí.
> Los Services **NO** llaman a Eloquent directamente.
> Cada repositorio implementa su Interface de `Contracts/`.

```
app/
└── Repositories/
    ├── Auth/
    │   ├── UserRepository.php
    │   └── TokenRepository.php
    ├── Attendance/
    │   ├── AttendanceRepository.php
    │   └── AbsenceRepository.php
    ├── ConnectionTime/
    │   └── ConnectionTimeRepository.php
    ├── VoiceTime/
    │   └── VoiceTimeRepository.php
    ├── Recruitment/
    │   ├── ApplierRepository.php
    │   ├── InterviewRepository.php
    │   ├── PublicationRepository.php
    │   └── ProfileRepository.php
    ├── Organization/
    │   ├── AreaRepository.php
    │   ├── DivisionRepository.php
    │   └── TeamRepository.php
    ├── Communication/
    │   ├── MessageRepository.php
    │   └── NotificationRepository.php
    └── Shared/
        └── StoredProcedureExecutor.php  # se mantiene, aislado aquí
```

---

### 📂 `app/Rules/`

```
app/
└── Rules/
    └── ValidDate.php
```

---

### 📂 `app/Services/` — 🔁 REFACTORIZADO

> Ya **no hay servicios duplicados por rol**.
> Un solo Service por dominio recibe el contexto (rol, área, división) como **parámetro**.
> Los Services dependen de Repositories, no de Models.

```
app/
└── Services/
    ├── Auth/
    │   ├── AuthService.php                        # login, logout, permisos
    │   ├── PasswordResetService.php
    │   └── TokenService.php
    │
    ├── Attendance/                                # ⭐ ~28 services → 7
    │   ├── AttendanceDashboardService.php          # unifica los 4 dashboards por rol
    │   ├── AttendanceKpiService.php
    │   ├── AttendanceCalendarService.php
    │   ├── AttendanceRankingService.php
    │   ├── AttendanceReportService.php
    │   ├── AttendanceFilterService.php
    │   └── AbsenceService.php
    │
    ├── ConnectionTime/                            # ⭐ ~26 services → 7
    │   ├── ConnectionTimeDashboardService.php
    │   ├── ConnectionTimeKpiService.php
    │   ├── ConnectionTimeEvolutionService.php
    │   ├── ConnectionTimeRankingService.php
    │   ├── ConnectionTimeReportService.php
    │   ├── ConnectionTimeCrudService.php           # create, update, search, detail
    │   └── ConnectionTimeAgreementService.php
    │
    ├── VoiceTime/                                 # ⭐ ~31 services → 6
    │   ├── VoiceTimeDashboardService.php
    │   ├── VoiceTimeKpiService.php
    │   ├── VoiceTimeCalendarService.php
    │   ├── VoiceTimeEvolutionService.php
    │   ├── VoiceTimeRankingService.php
    │   └── VoiceTimeDetailService.php
    │
    ├── Recruitment/                               # ⭐ ~20 services → 9
    │   ├── ApplierService.php
    │   ├── InterviewService.php
    │   ├── EvaluationService.php
    │   ├── PublicationService.php
    │   ├── PlatformService.php
    │   ├── ProfileDescriptionService.php
    │   ├── WorkCredentialService.php
    │   ├── PersonnelRequirementService.php
    │   └── DiscService.php
    │
    ├── Organization/
    │   ├── AreaService.php                        # incluye AreaResolverService
    │   ├── DivisionService.php                    # incluye DivisionResolverService
    │   └── TeamService.php
    │
    ├── Communication/
    │   ├── MessageService.php
    │   ├── SmsService.php
    │   ├── TelegramService.php
    │   └── WhatsAppService.php
    │
    ├── Certificate/
    │   └── CertificateService.php
    │
    ├── Training/
    │   └── TrainingService.php
    │
    ├── Speech/
    │   └── SpeechService.php
    │
    ├── Users/
    │   ├── UserService.php
    │   └── RoleService.php
    │
    └── Shared/
        ├── ImageStorageService.php
        ├── ReportGeneratorService.php
        └── StoredProcedureService.php             # wrapper del StoredProcedureExecutor
```

---

### 📂 `app/Support/` — 🔁 REFACTORIZADO

> Los Traits de lógica se convierten en clases utilitarias inyectables.
> Solo se mantienen como Traits los de presentación HTTP.

```
app/
└── Support/
    ├── Helpers/                               # antes: vivían como Traits
    │   ├── DateHelper.php                     # antes: Traits/DateHelper.php
    │   ├── DataNormalizer.php                 # antes: Traits/DataNormalizer.php
    │   ├── EncryptHelper.php                  # antes: Traits/EncryptHelper.php
    │   └── DeviceIpResolver.php               # antes: Traits/DeviceIpResolver.php
    └── Traits/                                # solo los de presentación HTTP
        ├── ApiResponse.php
        ├── HttpCodesHelper.php
        ├── EnumOptions.php
        └── EnumValues.php
```

---

### 📂 Raíz del proyecto

```
.
├── artisan
├── bootstrap/
│   ├── app.php
│   └── cache/
│       ├── .gitignore
│       ├── packages.php
│       └── services.php
├── config/
│   ├── app.php            ├── hashing.php       ├── sanctum.php
│   ├── auth.php           ├── logging.php       ├── services.php
│   ├── broadcasting.php   ├── mail.php          ├── session.php
│   ├── cache.php          ├── permission.php    ├── telescope.php
│   ├── cors.php           ├── queue.php         └── view.php
│   ├── database.php       │
│   └── filesystems.php    │
├── database/
│   ├── factories/
│   │   ├── ApplierFactory.php
│   │   └── UserFactory.php
│   ├── migrations/
│   │   └── 2023_05_16_174058_create_permission_tables.php
│   └── seeders/
│       ├── ApplierSeeder.php    ├── DivisionSeeder.php   ├── RoleSeeder.php
│       ├── AreaSeeder.php       ├── PerfilSeeder.php     ├── TurnSeeder.php
│       ├── CountrySeeder.php    ├── PlatformSeeder.php   └── UserSeeder.php
│       └── DatabaseSeeder.php
├── docs/
│   ├── api.md
│   ├── architecture.md          # ⭐ NUEVO
│   ├── conventions.md
│   ├── estandares.md
│   ├── git-flow.md
│   ├── runbook.md
│   └── setup.md
├── routes/
│   ├── api.php                  # solo incluye los demás archivos de rutas
│   ├── web.php
│   ├── channels.php
│   ├── console.php
│   └── api/                     # ⭐ NUEVA: rutas divididas por dominio
│       ├── auth.php             ├── communication.php
│       ├── attendance.php       ├── certificate.php
│       ├── connection-time.php  ├── training.php
│       ├── voice-time.php       ├── speech.php
│       ├── recruitment.php      ├── users.php
│       ├── organization.php     ├── reports.php
│       │                        └── integrations.php
├── storage/
│   ├── app/public/
│   ├── framework/
│   │   ├── cache/
│   │   ├── sessions/
│   │   └── views/
│   └── logs/laravel.log
├── tests/                       # ⭐ NUEVA: organizada por capa
│   ├── Unit/
│   │   ├── Services/
│   │   │   ├── Attendance/
│   │   │   │   └── AttendanceDashboardServiceTest.php
│   │   │   ├── ConnectionTime/
│   │   │   │   └── ConnectionTimeDashboardServiceTest.php
│   │   │   └── Recruitment/
│   │   │       └── ApplierServiceTest.php
│   │   ├── Mappers/
│   │   │   └── AttendanceMapperTest.php
│   │   └── Helpers/
│   │       └── DateHelperTest.php
│   ├── Feature/
│   │   ├── Auth/
│   │   │   └── LoginTest.php
│   │   ├── Attendance/
│   │   │   └── AttendanceDashboardTest.php
│   │   └── Recruitment/
│   │       └── ApplierTest.php
│   └── Integration/
│       └── Repositories/
│           └── AttendanceRepositoryTest.php
├── .editorconfig
├── .env  /  .env.example
├── composer.json  /  composer.lock
├── docker-compose.yml
├── phpunit.xml
└── vite.config.js
```

---

## 📦 Resumen de Cambios por Carpeta

| Estado | Carpeta | Descripción |
|--------|---------|-------------|
| ⭐ **NUEVA** | `app/Contracts/` | Interfaces para Repositories y Services |
| ⭐ **NUEVA** | `app/Repositories/` | Capa de acceso a datos (desacopla Services de Eloquent) |
| ⭐ **NUEVA** | `app/Support/Helpers/` | Clases utilitarias (antes eran Traits) |
| ⭐ **NUEVA** | `routes/api/` | Rutas divididas por dominio |
| ⭐ **NUEVA** | `tests/` | Estructura de tests por capa (Unit / Feature / Integration) |
| 🔁 **REFACTORIZADO** | `app/DTOs/` | Unificados por dominio — elimina duplicados por rol |
| 🔁 **REFACTORIZADO** | `app/Services/` | ~80 services → ~35, unificados por dominio |
| 🔁 **REFACTORIZADO** | `app/Http/Controllers/` | Agrupados por dominio, delgados sin lógica |
| 🔁 **REFACTORIZADO** | `app/Http/Requests/` | Base requests reutilizables, agrupados por dominio |
| 🔁 **REFACTORIZADO** | `app/Mappers/` | 1 por dominio en vez de 1 por rol |
| 🔁 **REFACTORIZADO** | `app/Support/Traits/` | Solo Traits de presentación HTTP |
| ✅ **SIN CAMBIOS** | `app/Models/` | Se mantienen igual |
| ✅ **SIN CAMBIOS** | `app/Enums/` | Se mantienen (renombrado `AssistStatus`) |
| ✅ **SIN CAMBIOS** | `app/Exceptions/` | Se mantienen, reorganizados en subcarpetas |
| ✅ **SIN CAMBIOS** | `config/` | Sin cambios |
| ✅ **SIN CAMBIOS** | `database/` | Sin cambios |

---

## 🔀 Flujo de Datos

### Ejemplo: `GET /api/attendance/dashboard`

```
1.  Request llega    →  AttendanceDashboardController::index()
                                    ↓
2.  Validación       →  DashboardFilterRequest valida params (fecha, rol, área)
                                    ↓
3.  Delegación       →  AttendanceDashboardService::getSummary($filterDTO)
                                    ↓
4.  Contexto de rol  →  Service determina contexto y llama AttendanceRepository
                                    ↓
5.  Acceso a datos   →  Repository ejecuta Stored Procedure o Query Eloquent
                                    ↓
6.  Transformación   →  Resultado crudo → AttendanceSummaryDTO via AttendanceMapper
                                    ↓
7.  Respuesta        →  Service devuelve DTO → Controller retorna JsonResponse
```

---

## 🗺️ Mapeo: Original → Nuevo

### Controllers consolidados

| Original | Nuevo |
|----------|-------|
| `V1CallsController` | `Calls/CallController` |
| `V1CallsModelController` | `Calls/CallModelController` |
| `V1CvController` | `Cv/CvController` |
| `V1CvkeywordsController` | `Cv/CvKeywordsController` |
| `V1PersonalDataController` | `PersonalData/PersonalDataController` |
| `V1MaterialController` | `Material/MaterialController` |
| `V1TurnController` | `Turn/TurnController` |
| `V1StaffController` | `Staff/StaffController` |
| `V1EndContractController` | `Contract/EndContractController` |
| `V1GroupListController` | `GroupList/GroupListController` |
| `V1RouteController` | `Route/RouteController` |
| `V1RouteInputController` | `Route/RouteInputController` |
| `V1RouteOutputController` | `Route/RouteOutputController` |
| `V1RouteRoutinesController` | `Route/RouteRoutinesController` |
| `V1PermissionsController` | `Permissions/PermissionsController` |
| `V1PermissionToRoleController` | `Permissions/PermissionToRoleController` |
| `V1RustDeskController` | `Integrations/RustDeskController` |
| `NgrokIpController` | `Integrations/NgrokController` |
| `V1MonthlyRequirementController` | `Recruitment/PersonnelRequirementController` |
| `GenerateMessagesListController` | `Communication/MessageController` (consolidado) |
| `ColumNameSpResolverController` | ❌ eliminado — lógica a `StoredProcedureService` |
| `CollaboratorKPITempController` | `Dashboard/ConnectionTimeDashboardController` |
| `TestUseProcedureController` | ❌ eliminado (era solo para testing) |
| `IndexController` | ❌ eliminado — reemplazar por rutas directas |
| `Dashboard_recruiter/.../Publish...` | `Recruitment/PublicationController` |
| `WhatsAutoController` | `Communication/WhatsAppController` |
| `V1ProcessMessageController` | `Communication/BotController` |
| `V1SendMessageController` | `Communication/BotController` |
| `V1SystemViewController` | `Users/RoleController` (permisos de vista) |
| `V1ContinuationListController` | `Recruitment/ApplierController` (consolidado) |
| `V1FilterQuestionsController` | `Recruitment/EvaluationController` |
| `V1EvaluationModelController` | `Recruitment/EvaluationController` |
| `V1PractitionerController` | `Users/UserController` |
| `V1ProcessController` | `Recruitment/ApplierController` |
| `V1PostController` / `V1PostModel` | `Recruitment/PublicationController` |
| `V1InputController` / `V1OutputController` | `Route/RouteController` |
| `Docweb/CodigoController` | ❌ eliminado o mover a `Integrations/` |
| `FileToUrlController` | `Util/FileToUrlController` |
| `EquiposTrabajoController` | `Organization/TeamController` |
| `V1DashboardController` | `Dashboard/` (cubierto) |

---

### Dashboard Controllers — 12 → 3

```
DashboardColaboradorControllerV3      ┐
DashboardGerenciaControllerV3         ├──▶  Dashboard/AttendanceDashboardController
DashboardLiderAreaControllerV3        │     (rol resuelto por middleware o parámetro)
DashboardLiderDivisionControllerV3    ┘

DashboardColaboradorConControllerV3   ┐
DashboardGerenciaConControllerV3      ├──▶  Dashboard/ConnectionTimeDashboardController
DashboardLiderAreaConControllerV3     │
DashboardLiderDivisionConControllerV3 ┘

DashboardThColaboradorController      ┐
DashboardthGerenciaControllerV1       ├──▶  Dashboard/VoiceTimeDashboardController
DashboardLiderAreaThControllerV1      │
DashboardLiderDivisionControllerV1    ┘
```

---

### Requests consolidados

| Original | Nuevo |
|----------|-------|
| `AppNuevoApi/RangoFechas/RangoFechasRequest` | `Base/DateRangeRequest` |
| `AppNuevoApi/RangoFechas/RFAreaRequest` | `Filters/RangeWithAreaRequest` |
| `AppNuevoApi/RangoFechas/RFAyDivisionRequest` | `Filters/RangeWithDivisionRequest` |
| `AppNuevoApi/RangoFechas/RFOpcionalesRequest` | `Filters/RangeOptionalRequest` |
| `AppNuevoApi/RangoFechas/RFAySearchRequest` | `Filters/RangeWithSearchRequest` |
| `AppNuevoApi/RangoFechas/RFPractitionerRequest` | `Filters/PractitionerRangeRequest` |
| `AppNuevoApi/Puros/SoloIdRequest` | `Shared/SoloIdRequest` |
| `AppNuevoApi/Puros/SoloAreaRequest` | `Shared/SoloAreaRequest` |
| `AppNuevoApi/FechaEspecifica/*` | `Base/SpecificDateRequest` |
| `DashboardConGerenciaRequest` | `Dashboard/DashboardFilterRequest` |
| `DashboardConLiderDivisionRequest` | `Dashboard/DashboardFilterRequest` |
| `DashboardThGerenciaRequest` | `Dashboard/DashboardFilterRequest` |

---

### Services consolidados — ~80 → ~35

```
ColaboradorService/Asistencia/*  (6)    ┐
GerenciaService/Asistencia/*     (6)    ├──▶  Services/Attendance/  (7 services)
LiderAreaService/Asistencia/*    (6)    │     con parámetro RoleContext
LiderDivisionService/Asistencia/*(10)   ┘

ColaboradorService/TiempoConexion/* (8) ┐
GerenciaService/TiempoConexion/*    (6) ├──▶  Services/ConnectionTime/  (7 services)
LiderAreaService/TiempoConexion/*   (6) │
LiderDivisionService/TiempoConexion/(6) ┘

ColaboradorService/TiempoHablado/*  (6) ┐
GerenciaService/TiempoHablado/*     (7) ├──▶  Services/VoiceTime/  (6 services)
LiderAreaService/TiempoHablado/*    (8) │
LiderDivisionService/TiempoHablado/(10) ┘

LiderAreaService/Reclutamiento/*  (~20) ──▶  Services/Recruitment/  (9 services)
```

---

### Traits migrados a clases

| Trait original | Destino | Estado |
|----------------|---------|--------|
| `Traits/DataGetter.php` | Lógica distribuida a Services/Repositories | ❌ eliminado |
| `Traits/DataNormalizer.php` | `Support/Helpers/DataNormalizer.php` | 🔁 convertido |
| `Traits/DataValid.php` | Lógica movida a FormRequests | ❌ eliminado |
| `Traits/DateHelper.php` | `Support/Helpers/DateHelper.php` | 🔁 convertido |
| `Traits/DeviceIpResolver.php` | `Support/Helpers/DeviceIpResolver.php` | 🔁 convertido |
| `Traits/EncryptHelper.php` | `Support/Helpers/EncryptHelper.php` | 🔁 convertido |
| `Traits/getPractitioner.php` | Lógica movida a `UserRepository` | ❌ eliminado |
| `Traits/HttpResponseHelper.php` | Duplicaba `ApiResponse` Trait | ❌ eliminado |
| `Traits/ApiResponse.php` | `Support/Traits/ApiResponse.php` | ✅ se mantiene |
| `Traits/HttpCodesHelper.php` | `Support/Traits/HttpCodesHelper.php` | ✅ se mantiene |
| `Traits/EnumOptions.php` | `Support/Traits/EnumOptions.php` | ✅ se mantiene |
| `Traits/EnumValues.php` | `Support/Traits/EnumValues.php` | ✅ se mantiene |

---

### Mappers

| Mapper | Función |
|--------|---------|
| `AttendanceMapper.php` | Transforma SP results → DTOs de Asistencia |
| `ConnectionTimeMapper.php` | Transforma SP results → DTOs de TiempoConexion |
| `VoiceTimeMapper.php` | Transforma SP results → DTOs de TiempoHablado |
| `RecruitmentMapper.php` | Transforma SP results → DTOs de Reclutamiento |
| `OrganizationMapper.php` | Transforma modelos → DTOs de Organización |

---

## 📐 Convención de Nombres

| Clase | Patrón | Ejemplo |
|-------|--------|---------|
| **Controllers** | `NombreDominioController` | `AttendanceDashboardController` |
| **Services** | `NombreDominioAccionService` | `AttendanceDashboardService` |
| **Repositories** | `NombreDominioRepository` | `AttendanceRepository` |
| **Interfaces** | `NombreDominioRepositoryInterface` | `AttendanceRepositoryInterface` |
| **DTOs** | `NombreDominioConceptoDTO` | `AttendanceSummaryDTO` |
| **Requests** | `NombreDominioAccionRequest` | `DashboardFilterRequest` |
| **Mappers** | `NombreDominioMapper` | `AttendanceMapper` |
| **Helpers** | `NombreConceptoHelper` | `DateHelper` |

> ⚠️ Sin prefijo `V1` en ninguna clase. La versión del API se maneja en la **URL** (`/api/v1/`), no en el nombre de la clase.

---

## 🚨 Regla de Oro

> **Si el nombre de una clase incluye un ROL** (`Colaborador`, `Gerencia`, `LiderArea`, `LiderDivision`) **→ es código duplicado.**
>
> El rol debe ser un **parámetro**, no parte del nombre de la clase.

```php
// ❌ ANTES — 4 clases haciendo lo mismo
class DashboardColaboradorService   { ... }
class DashboardGerenciaService      { ... }
class DashboardLiderAreaService     { ... }
class DashboardLiderDivisionService { ... }

// ✅ DESPUÉS — 1 clase, el rol es un parámetro
class AttendanceDashboardService {
    public function getSummary(RoleContext $context): AttendanceSummaryDTO
    {
        // el contexto de rol determina el scope de los datos
    }
}
```

---

*Documentación generada como parte del proceso de refactorización arquitectural.*
