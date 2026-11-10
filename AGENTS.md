You are an expert in TypeScript, Angular, and scalable web application development. You write functional, maintainable, performant, and accessible code following Angular and TypeScript best practices.

## TypeScript Best Practices

- Use strict type checking
- Prefer type inference when the type is obvious
- Avoid the `any` type; use `unknown` when type is uncertain

## Angular Best Practices

- Always use standalone components over NgModules
- Must NOT set `standalone: true` inside Angular decorators. It's the default in Angular v20+.
- Do NOT set `changeDetection: ChangeDetectionStrategy.OnPush` explicitly. `OnPush` is the default in Angular v22+.
- Use signals for state management
- Implement lazy loading for feature routes
- Do NOT use the `@HostBinding` and `@HostListener` decorators. Put host bindings inside the `host` object of the `@Component` or `@Directive` decorator instead
- Use `NgOptimizedImage` for all static images.
  - `NgOptimizedImage` does not work for inline base64 images.

## Accessibility Requirements

- It MUST pass all AXE checks.
- It MUST follow all WCAG AA minimums, including focus management, color contrast, and ARIA attributes.

## Component File Format (required)

Every component uses the classic four-file format. NEVER put the template or styles inside the `.ts` file, regardless of component size.

```
user-card/
├── user-card.component.ts
├── user-card.component.html
├── user-card.component.css
└── user-card.component.spec.ts
```

```ts
import { Component, input, output } from '@angular/core';
import { User } from '../../../domain/models/user.model';

@Component({
  selector: 'app-user-card',
  imports: [],
  templateUrl: './user-card.component.html',
  styleUrl: './user-card.component.css',
})
export class UserCardComponent {
  user = input.required<User>();
  selected = output<string>();

  onSelect(): void {
    this.selected.emit(this.user().id);
  }
}
```

- Always use `templateUrl` and `styleUrl`. Do NOT use `template:` or `styles:`.
- Every component gets its own `.html`, `.css` and `.spec.ts` file, even if the CSS file starts empty.
- Use paths relative to the component TS file for `templateUrl` and `styleUrl`.
- Generate components with the CLI and never pass `--inline-template` or `--inline-style`:
  `ng generate component features/users/presentation/components/user-card`

### Components

- Keep components small and focused on a single responsibility
- Use `input()` and `output()` functions instead of decorators
- Use `model()` for two-way bound properties with `[(prop)]` syntax instead of pairing `input()` with `output()`
- Use `computed()` for derived state
- Use `linkedSignal()` for state derived from multiple reactive sources that must stay synchronized
- Always use external template and style files (see Component File Format). Do NOT use inline templates, even for small components
- Prefer Signal Forms (`@angular/forms/signals`) for new forms. They are stable in Angular v22+ and provide signal-based state, type-safe field access, and schema-based validation
- When not using Signal Forms, prefer Reactive forms instead of Template-driven ones
- Do NOT use `ngClass`, use `class` bindings instead
- Do NOT use `ngStyle`, use `style` bindings instead
- Split a component when its `.ts` file passes ~150 lines or its template ~100 lines

### Smart vs Presentational Components

- **Pages** (`presentation/pages/`) are smart: they inject facades, read state and handle routing. One page per route.
- **Components** (`presentation/components/`) are presentational: data comes in through `input()`, actions go out through `output()`. They do not inject facades, repositories or HTTP services.

## Clean Architecture

The app is split into four layers. Dependencies point inward only: presentation → application → domain. The data layer implements interfaces defined in the domain.

| Layer | Contains | May depend on | Must NOT |
| --- | --- | --- | --- |
| Domain | Models, repository abstractions (abstract classes), business rules | Nothing (plain TypeScript) | Import from `@angular/*` or any other layer |
| Application | Facades / use cases, feature state as signals | Domain | Use `HttpClient` or touch the DOM |
| Data | Repository implementations, API calls, DTOs, mappers | Domain, `HttpClient` | Be imported by components or pages |
| Presentation | Pages, components, feature routes | Application, domain models | Use `HttpClient` or repositories directly |

### How the layers connect

1. The domain defines the contract as an abstract class:

```ts
// features/users/domain/repositories/user.repository.ts
export abstract class UserRepository {
  abstract getAll(): Observable<User[]>;
  abstract getById(id: string): Observable<User>;
}
```

2. The data layer implements it and maps DTOs to domain models:

```ts
// features/users/data/repositories/user-http.repository.ts
@Injectable()
export class UserHttpRepository extends UserRepository {
  private readonly http = inject(HttpClient);

  getAll(): Observable<User[]> {
    return this.http
      .get<UserDto[]>(`${environment.apiUrl}/users`)
      .pipe(map(UserMapper.toDomainList));
  }
}
```

3. The feature routes bind the abstraction to the implementation:

```ts
// features/users/users.routes.ts
export const USERS_ROUTES: Routes = [
  {
    path: '',
    providers: [
      { provide: UserRepository, useClass: UserHttpRepository },
      UsersFacade,
    ],
    children: [
      { path: '', loadComponent: () => import('./presentation/pages/user-list/user-list.component').then(m => m.UserListComponent) },
    ],
  },
];
```

4. The application layer (facade) depends only on the abstraction and exposes state:

```ts
// features/users/application/users.facade.ts
@Injectable()
export class UsersFacade {
  private readonly repo = inject(UserRepository);
  readonly users = signal<User[]>([]);
  readonly loading = signal(false);
}
```

5. Pages inject the facade. Components only receive inputs and emit outputs.

- Never expose DTOs to the presentation layer. Map them to domain models in the data layer.
- Feature-scoped classes (facades, repository implementations) are provided in the feature's routes, not in root.

## Folder Structure

```
src/
├── app/
│   ├── core/                        # App-wide singletons, loaded once
│   │   ├── auth/                    # Auth service, auth guard
│   │   ├── interceptors/            # HTTP interceptors (auth token, errors)
│   │   ├── layout/                  # Shell, header, sidebar, footer
│   │   └── services/                # Global services (logging, notifications)
│   │
│   ├── shared/                      # Reusable, feature-agnostic building blocks
│   │   ├── components/              # Buttons, modals, tables, loaders
│   │   ├── directives/
│   │   ├── pipes/
│   │   ├── models/                  # Shared types (Paged<T>, ApiError)
│   │   └── utils/                   # Pure helper functions
│   │
│   ├── features/                    # One folder per business feature
│   │   └── users/
│   │       ├── domain/
│   │       │   ├── models/          # user.model.ts
│   │       │   └── repositories/    # user.repository.ts (abstract)
│   │       ├── application/
│   │       │   └── users.facade.ts
│   │       ├── data/
│   │       │   ├── dtos/            # user.dto.ts
│   │       │   ├── mappers/         # user.mapper.ts
│   │       │   └── repositories/    # user-http.repository.ts
│   │       ├── presentation/
│   │       │   ├── pages/
│   │       │   │   ├── user-list/
│   │       │   │   └── user-detail/
│   │       │   └── components/
│   │       │       └── user-card/
│   │       └── users.routes.ts
│   │
│   ├── app.component.ts / .html / .css
│   ├── app.config.ts
│   └── app.routes.ts                # Lazy-loads each feature's routes
│
├── environments/
│   ├── environment.ts
│   └── environment.development.ts
├── assets/
└── styles.css                       # Global styles and CSS variables only
```

- `core/` is used only by `app.config.ts`, `app.routes.ts` and the layout.
- Features never import from another feature's folder. Code needed by two or more features moves to `shared/` (UI) or `core/` (services).
- Every feature is lazy-loaded with `loadChildren` from `app.routes.ts`.
- Use path aliases from `tsconfig.json` (`@core/*`, `@shared/*`, `@features/*`, `@env/*`) instead of deep relative imports like `../../../../`.
- API base URLs live in `environments/`, never hard-coded.

## Naming Conventions

| Item | File name | Symbol |
| --- | --- | --- |
| Component | `user-card.component.ts` (+ `.html`, `.css`, `.spec.ts`) | `UserCardComponent` |
| Page | `user-list.component.ts` in `pages/` | `UserListComponent` |
| Domain model | `user.model.ts` | `User` |
| DTO | `user.dto.ts` | `UserDto` |
| Mapper | `user.mapper.ts` | `UserMapper` |
| Repository (abstract) | `user.repository.ts` | `UserRepository` |
| Repository (implementation) | `user-http.repository.ts` | `UserHttpRepository` |
| Facade | `users.facade.ts` | `UsersFacade` |
| Service | `notification.service.ts` | `NotificationService` |
| Guard | `auth.guard.ts` | `authGuard` (functional) |
| Interceptor | `auth.interceptor.ts` | `authInterceptor` (functional) |
| Routes | `users.routes.ts` | `USERS_ROUTES` |

- File and folder names use kebab-case.
- Component selectors use the `app-` prefix.
- One class, interface or component per file.

## State Management

- Use signals for local component state
- Use `computed()` for derived state
- Feature state lives in the feature's facade as signals; pages read it, components receive it through inputs
- Keep state transformations pure and predictable
- Do NOT use `mutate` on signals, use `update` or `set` instead

## Templates

- Keep templates simple and avoid complex logic
- Use native control flow (`@if`, `@for`, `@switch`) instead of `*ngIf`, `*ngFor`, `*ngSwitch`
- Always add `track` to `@for`
- Use the async pipe to handle observables
- Do not assume globals like (`new Date()`) are available.
- Handle loading, empty and error states in every page

## Styles

- Component styles stay in the component's `.css` file
- Global CSS variables and resets live in `src/styles.css`
- Do NOT use `::ng-deep`

## Services

- Design services around a single responsibility
- Use the `providedIn: 'root'` option for singleton services
- Prefer the `@Service` decorator over `@Injectable({providedIn: 'root'})` for new singleton services (Angular v22+)
- Root singletons belong in `core/`; feature-scoped services are provided in the feature's routes
- Use the `inject()` function instead of constructor injection

## Testing

- Every component, facade, mapper and repository has a `.spec.ts` beside it
- Test facades with a fake repository, not by mocking `HttpClient`
- Test presentational components through their inputs and outputs

## Before Finishing a Task

- [ ] No `template:` or `styles:` in any `@Component` decorator
- [ ] Every new component has its `.html`, `.css` and `.spec.ts` files
- [ ] New files are in the correct layer and folder
- [ ] No component or page imports `HttpClient` or a repository
- [ ] Domain files import nothing from `@angular/*`
- [ ] `ng lint` and `ng test` pass