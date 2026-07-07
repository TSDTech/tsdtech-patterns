# TSDTech — Code Patterns

Template completo para criar features seguindo DDD Anêmico. Cada camada com seu padrão específico. Use como referência ao implementar.

---

## Índice

- [1. Visão Geral do Fluxo](#1-visão-geral-do-fluxo)
- [2. Entity](#2-entity)
- [3. DAO (Firestore)](#3-dao-firestore)
- [4. Repository (PostgreSQL/Knex)](#4-repository-postgresqlknex)
- [5. Request DTO](#5-request-dto)
- [6. Response DTO](#6-response-dto)
- [7. Service](#7-service)
- [8. Controller](#8-controller)
- [9. Module](#9-module)
- [10. Feature Completa (Exemplo)](#10-feature-completa-exemplo)

---

## 1. Visão Geral do Fluxo

```
HTTP Request
  → Controller (valida DTO, delega)
  → Service (regras de négocio, chama DAOs/externals)
  → DAO/Repository (CRUD no banco)
  → Entity (modelo de domínio)
  → Service (monta response DTO)
  → Controller (retorna response)
```

### Regras de Ouro

| Camada | Pode | NÃO Pode |
|--------|------|----------|
| Controller | @Body, @Param, @Query, Guards, Swagger | Regra de négocio, acesso a DAO |
| Service | Tudo de negócio, chamar DAOs/externals | @Body, @Param, decorators HTTP |
| DAO/Repository | CRUD, queries, filtros | Lógica de negócio, validacão |
| Entity | @JsonObject, @JsonProperty, constructor | DAO, Swagger, persistência |
| DTO | class-validator, @ApiProperty | @JsonObject, entity inheritance |

---

## 2. Entity

### Base — EntityImmutable / EntityMutable

```typescript
// src/core/common-models/immutable.entity.ts
@JsonObject()
export abstract class EntityImmutable {
  @JsonProperty()
  private id: string;

  @JsonProperty({
    beforeDeserialize: (p) => new Date(p).toISOString(),
    beforeSerialize: (p) => p?.getTime(),
  })
  private createdAtUtc: Date;

  constructor(params: { id?: string; createdAtUtc?: Date }) {
    if (!params) return;
    this.setId(params?.id);
    this.createdAtUtc = params?.createdAtUtc ?? new Date();
  }

  public setId(id: string) {
    this.id = id ?? crypto.randomUUID();
  }

  public getId(): string { return this.id; }
  public getCreatedAtUtc(): Date { return this.createdAtUtc; }
}
```

```typescript
// src/core/common-models/mutable.entity.ts
@JsonObject()
export class EntityMutable extends EntityImmutable {
  @JsonProperty({
    beforeDeserialize: (p) => new Date(p).toISOString(),
    beforeSerialize: (p) => p?.getTime(),
  })
  private updatedAtUtc?: Date;

  constructor(params: { id?: string; createdAtUtc?: Date; updatedAtUtc?: Date }) {
    super(params);
    if (!params) return;
    this.setUpdatedAtUtc(params?.updatedAtUtc);
  }

  public getUpdatedAtUtc(): Date { return this.updatedAtUtc; }
  public setUpdatedAtUtc(d: Date) { this.updatedAtUtc = d; }
  public update() { this.setUpdatedAtUtc(new Date()); }
}
```

### Firestore Entity

```typescript
// src/modules/api/<feature>/entities/<entidade>.entity.ts
import { JsonObject, JsonProperty } from 'typescript-json-serializer';
import { EntityMutable } from 'src/core/common-models/mutable.entity';
import Decimal from 'decimal.js';
import { DecimalJsonSerializer } from 'src/core/decorators/decimal.decorator';

@JsonObject()
export class MinhaEntidade extends EntityMutable {
  @JsonProperty()
  private nome: string;

  @DecimalJsonSerializer()
  private valor: Decimal;

  @JsonProperty({ name: 'status_atual' })
  private statusAtual: string;

  @JsonProperty()
  private idempotencyKey: string;

  constructor(p: {
    id?: string;
    nome: string;
    valor: Decimal;
    statusAtual: string;
    idempotencyKey: string;
    createdAtUtc?: Date;
    updatedAtUtc?: Date;
  }) {
    super(p);
    if (!p) return;
    this.nome = p.nome;
    this.valor = p.valor;
    this.statusAtual = p.statusAtual;
    this.idempotencyKey = p.idempotencyKey;
  }

  // ── Getters ──
  getNome(): string { return this.nome; }
  getValor(): Decimal { return this.valor; }
  getStatusAtual(): string { return this.statusAtual; }
  getIdempotencyKey(): string { return this.idempotencyKey; }

  // ── Setters ──
  setStatusAtual(s: string) { this.statusAtual = s; }
}
```

### Regras da Entity

- `@JsonObject()` na classe + `@JsonProperty()` em cada campo **privado**
- Campos privados com getters públicos (`getNome()`, `getValor()`)
- Valores financeiros: `@DecimalJsonSerializer()` com tipo `Decimal`
- Construtor recebe objeto tipado com todos os campos
- `super(p)` no início + `if (!p) return;` (obrigatório para serializacão)
- Timestamps serializados como Unix timestamp (`getTime()`) no Firestore
- NUNCA colocar Swagger, DAO, ou lógica de negócio aqui

---

## 3. DAO (Firestore)

### BaseDao (v2 — padrão atual)

```typescript
// src/core/base/base.dao.ts
export abstract class BaseDao<T extends { getId(): string }> {
  protected abstract collectionName: string;

  constructor(
    @Inject(Firestore) protected readonly firestore: Firestore,
    private readonly type: Type<T>,
    private readonly readOnly: boolean = false,
  ) {}
}
```

### Métodos do BaseDao

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| **findAll** | `({ filterIdsAction?, queryFilterAction?, filters?, pagination? })` | Busca com filtro duplo (Firestore + in-memory) + paginação cursor |
| **create** | `({ entities: T[], transaction?: WriteBatch })` | **Sempre batch** — usa `batch.set()` |
| **update** | `({ entities: T[], transaction?: WriteBatch })` | **Sempre batch** — usa `batch.update()` |
| **delete** | `({ ids: string[], transaction?: WriteBatch })` | **Sempre batch** — usa `batch.delete()` |

### Como obter por ID

**Não existe `findOne(id)`.** Use `findAll` com `filters.ids`:

```typescript
const { items } = await this.dao.findAll({ filters: { ids: [id] } });
const entity = items[0]; // pode ser undefined
```

### Paginação

Cursor-based via `lastEvaluatedId`:

```typescript
const result = await this.dao.findAll({
  filters: { ... },
  pagination: new PaginationDto({ lastEvaluatedId: 'abc', limit: 20 }),
});
// result.items → T[]
// result.nextEvaluatedId → string | undefined (para próxima página)
```

### Como implementar um DAO

```typescript
// src/modules/api/<feature>/daos/<entidade>.dao.ts
import { Injectable } from '@nestjs/common';
import { BaseDao } from 'src/core/base/base.dao';
import { Firestore } from '@google-cloud/firestore';
import { PaginatedListDto } from 'src/core/dtos/paginated-list.dto';
import { PaginationDto } from 'src/core/dtos/pagination.dto';
import { FilterMinhaEntidadeDto } from '../dto/filter-minha-entidade.dto';
import { MinhaEntidade } from '../entities/minha-entidade.entity';

@Injectable()
export class MinhaEntidadeDao extends BaseDao<MinhaEntidade> {
  protected collectionName = 'minhas_entidades';

  constructor(firestore: Firestore) {
    super(firestore, MinhaEntidade);
  }

  async findAll(params: {
    filters?: FilterMinhaEntidadeDto;
    pagination?: PaginationDto;
  }): Promise<PaginatedListDto<MinhaEntidade>> {
    return super.findAll({
      ...params,

      // Filtro in-memory (pós-query) — para campos que não podem ir no where()
      filterIdsAction: (entity) => {
        if (params.filters?.organizationIds &&
            !params.filters.organizationIds.includes(entity.getOrganizationId())) {
          return false;
        }
        return true;
      },

      // Filtro Firestore (pré-query) — usa where() do Firestore
      queryFilterAction: (query) => {
        if (params.filters?.organizationIds) {
          query = query.where('organizationId', 'in', params.filters.organizationIds);
        }
        if (params.filters?.statuses) {
          query = query.where('status', 'in', params.filters.statuses);
        }
        return query;
      },
    });
  }
}
```

### Dual Filter: queryFilterAction vs filterIdsAction

| | queryFilterAction | filterIdsAction |
|--|-------------------|-----------------|
| **Quando** | Filters.ids não foi passado | Sempre roda (também com ids) |
| **Onde** | Firestore `where()` (server-side) | In-memory (client-side) |
| **Pra que** | Filtrar no banco (performance) | Pós-filtro para campos não-indexados |
| **Exemplo** | `query.where('status', 'in', statuses)` | `entity.getNome().includes(searchTerm)` |

### Batch + Transaction

**Todas as operacões de escrita usam `WriteBatch`.** O `transaction` é opcional — quando passado, o batch não é committado automaticamente, permitindo compor transacões multi-DAO:

```typescript
// Auto-commit (caso mais comum)
await this.dao.create({ entities: [entity] });

// Transacão compartilhada entre DAOs
const batch = firestore.batch();
await this.daoA.create({ entities: [a], transaction: batch });
await this.daoB.create({ entities: [b], transaction: batch });
await batch.commit();
```

---

## 4. Repository (PostgreSQL/Knex)

```typescript
// src/modules/api/<feature>/repositories/<entidade>.repository.ts
import { Injectable } from '@nestjs/common';
import { Knex } from 'knex';
import { BaseRepository } from 'src/core/services/base.repository';
import { MinhaEntidade } from '../entities/minha-entidade.entity';

export interface MinhaEntidadeFilter {
  ids?: string[];
  organizationId?: string;
  status?: string;
}

@Injectable()
export class MinhaEntidadeRepository extends BaseRepository<
  MinhaEntidade,
  MinhaEntidadeFilter
> {
  protected tableName = 'minhas_entidades';

  constructor(protected readonly knex: Knex) {
    super(knex, MinhaEntidade);
  }

  findBy(query: Knex.QueryBuilder, filters?: MinhaEntidadeFilter): void {
    if (filters?.ids?.length) {
      query.whereIn('id', filters.ids);
    }
    if (filters?.organizationId) {
      query.where('organization_id', filters.organizationId);
    }
    if (filters?.status) {
      query.where('status', filters.status);
    }
  }
}
```

---

## 5. Request DTO

```typescript
// src/modules/api/<feature>/dto/create-<entidade>.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import {
  IsNotEmpty,
  IsString,
  IsNumber,
  IsOptional,
  IsEnum,
  MaxLength,
  ArrayMaxSize,
} from 'class-validator';

export class CreateMinhaEntidadeDto {
  @ApiProperty({ description: 'Nome da entidade', example: 'João Silva' })
  @IsNotEmpty()
  @IsString()
  @MaxLength(200)
  nome: string;

  @ApiProperty({ description: 'Valor em centavos', example: 1500 })
  @IsNotEmpty()
  @IsNumber()
  valor: number; // ← centavos! Não Decimal

  @ApiProperty({ description: 'Status', enum: ['ACTIVE', 'INACTIVE'], example: 'ACTIVE' })
  @IsOptional()
  @IsEnum(['ACTIVE', 'INACTIVE'])
  status?: string;
}
```

```typescript
// src/modules/api/<feature>/dto/filter-<entidade>.dto.ts
import { ApiProperty } from '@nestjs/swagger';
import { IsOptional, IsArray, IsString } from 'class-validator';

export class FilterMinhaEntidadeDto {
  @ApiProperty({ required: false })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  ids?: string[];

  @ApiProperty({ required: false })
  @IsOptional()
  @IsArray()
  @IsString({ each: true })
  organizationIds?: string[];
}
```

---

## 6. Response DTO

```typescript
// src/modules/api/<feature>/dto/<entidade>-response.dto.ts
import { ApiProperty } from '@nestjs/swagger';

export class MinhaEntidadeResponseDto {
  @ApiProperty() id: string;
  @ApiProperty() nome: string;
  @ApiProperty({ description: 'Valor em centavos', example: 1500 }) valor: number;
  @ApiProperty({ description: 'Data de criacão' }) createdAt: Date;

  static fromEntity(entity: MinhaEntidade): MinhaEntidadeResponseDto {
    const dto = new MinhaEntidadeResponseDto();
    dto.id = entity.getId();
    dto.nome = entity.getNome();
    dto.valor = entity.getValor().times(100).toNumber(); // Decimal → centavos
    dto.createdAt = entity.getCreatedAtUtc();
    return dto;
  }
}
```

---

## 7. Service

### CREATE

```typescript
async create(dto: CreateMinhaEntidadeDto): Promise<MinhaEntidadeResponseDto> {
  await this.validateCreate(dto);

  const valorDecimal = new Decimal(dto.valor).div(100);

  const entity = new MinhaEntidade({
    nome: dto.nome,
    valor: valorDecimal,
    statusAtual: dto.status ?? 'ACTIVE',
    idempotencyKey: crypto.randomUUID(),
  });

  const [created] = await this.dao.create({ entities: [entity] });
  return MinhaEntidadeResponseDto.fromEntity(created);
}
```

### FIND BY ID (via findAll com filters.ids)

```typescript
async findById(id: string): Promise<MinhaEntidadeResponseDto> {
  const { items } = await this.dao.findAll({ filters: { ids: [id] } });
  if (!items.length) {
    throw HTTPError.NOT_FOUND({
      message: 'Entidade não encontrada',
      title: 'Not Found',
    });
  }
  return MinhaEntidadeResponseDto.fromEntity(items[0]);
}
```

### UPDATE

```typescript
async update(id: string, dto: Partial<CreateMinhaEntidadeDto>): Promise<MinhaEntidadeResponseDto> {
  const { items } = await this.dao.findAll({ filters: { ids: [id] } });
  if (!items.length) {
    throw HTTPError.NOT_FOUND({
      message: 'Entidade não encontrada',
      title: 'Not Found',
    });
  }

  const entity = items[0];
  if (dto.nome !== undefined) entity.setNome(dto.nome);
  if (dto.valor !== undefined) entity.setValor(new Decimal(dto.valor).div(100));

  const [updated] = await this.dao.update({ entities: [entity] });
  return MinhaEntidadeResponseDto.fromEntity(updated);
}
```

### Procedural Mutation (find → modify → update)

```typescript
async desativar(id: string): Promise<void> {
  const { items: [entity] } = await this.dao.findAll({ filters: { ids: [id] } });
  if (!entity) throw HTTPError.NOT_FOUND({ message: 'Não encontrada', title: 'Not Found' });

  entity.setStatusAtual('INACTIVE');
  await this.dao.update({ entities: [entity] });
}
```

### DELETE

```typescript
async delete(id: string): Promise<void> {
  const { items } = await this.dao.findAll({ filters: { ids: [id] } });
  if (!items.length) throw HTTPError.NOT_FOUND({ message: 'Não encontrada', title: 'Not Found' });

  await this.dao.delete({ ids: [id] });
}
```

### CREATE com batch compartilhado

```typescript
// Quando precisa atomicidade entre múltiplos DAOs
async createComTransacao(dto: CreateDto): Promise<ResponseDto> {
  const batch = this.firestore.batch(); // ou injetar WriteBatch

  const entityA = new EntidadeA({ ... });
  const entityB = new EntidadeB({ ... });

  await this.daoA.create({ entities: [entityA], transaction: batch });
  await this.daoB.create({ entities: [entityB], transaction: batch });

  await batch.commit(); // commit manual
  // ...
}
```

---

## 8. Controller

```typescript
// src/modules/api/<feature>/controllers/<feature>.controller.ts
import { Body, Controller, Delete, Get, Param, Post, Put, Query } from '@nestjs/common';
import { ApiResponse, ApiTags } from '@nestjs/swagger';
import { PaginatedListDto } from 'src/core/dtos/paginated-list.dto';
import { PaginationDto } from 'src/core/dtos/pagination.dto';
import { PinbankProtected } from 'src/core/guards/pinbank-protected/pinbank-protected.decorator';

import { MinhaEntidadeService } from '../<feature>.service';
import { CreateMinhaEntidadeDto } from '../dto/create-minha-entidade.dto';
import { FilterMinhaEntidadeDto } from '../dto/filter-minha-entidade.dto';
import { MinhaEntidadeResponseDto } from '../dto/minha-entidade-response.dto';

@ApiTags('MinhaEntidade')
@Controller('MinhaEntidade')
export class MinhaEntidadeController {
  constructor(private readonly service: MinhaEntidadeService) {}

  @Post('Create')
  @PinbankProtected({ encrypted: false })
  @ApiResponse({ type: MinhaEntidadeResponseDto, status: 201 })
  async create(@Body() dto: CreateMinhaEntidadeDto): Promise<MinhaEntidadeResponseDto> {
    return this.service.create(dto);
  }

  @Get('Get/:id')
  @PinbankProtected({ encrypted: false })
  @ApiResponse({ type: MinhaEntidadeResponseDto })
  async get(@Param('id') id: string): Promise<MinhaEntidadeResponseDto> {
    return this.service.findById(id);
  }

  @Get('List')
  @PinbankProtected({ encrypted: false })
  @ApiResponse({ type: PaginatedListDto<MinhaEntidadeResponseDto> })
  async list(
    @Query() filters?: FilterMinhaEntidadeDto,
    @Query() pagination?: PaginationDto,
  ): Promise<PaginatedListDto<MinhaEntidadeResponseDto>> {
    return this.service.findAll(filters, pagination);
  }

  @Put('Update/:id')
  @PinbankProtected({ encrypted: false })
  @ApiResponse({ type: MinhaEntidadeResponseDto })
  async update(
    @Param('id') id: string,
    @Body() dto: CreateMinhaEntidadeDto,
  ): Promise<MinhaEntidadeResponseDto> {
    return this.service.update(id, dto);
  }

  @Delete('Delete/:id')
  @PinbankProtected({ encrypted: false })
  @ApiResponse({ status: 204 })
  async delete(@Param('id') id: string): Promise<void> {
    return this.service.delete(id);
  }
}
```

### Health Controller

```typescript
@Controller()
export class HealthController {
  @Get('health')
  health() { return { status: 'ok', timestamp: new Date().toISOString() }; }

  @Get('health/ready')
  ready() { return { status: 'ready' }; }
}
```

---

## 9. Module

```typescript
// src/modules/api/<feature>/<feature>.module.ts
import { Module } from '@nestjs/common';

@Module({
  imports: [],
  controllers: [MinhaEntidadeController],
  providers: [MinhaEntidadeService, MinhaEntidadeDao],
  exports: [MinhaEntidadeService, MinhaEntidadeDao],
})
export class MinhaEntidadeModule {}
```

---

## 10. Feature Completa (Exemplo)

```
src/modules/api/produto/
├── controllers/
│   └── produto.controller.ts
├── daos/
│   └── produto.dao.ts
├── dto/
│   ├── create-produto.dto.ts
│   ├── filter-produto.dto.ts
│   └── produto-response.dto.ts
├── entities/
│   └── produto.entity.ts
├── produto.service.ts
└── produto.module.ts
```

### Fluxo de Implementacão

1. **Entity** — estender EntityMutable, campos privados com getters, @DecimalJsonSerializer
2. **DAO** — estender BaseDao, implementar findAll com filterIdsAction + queryFilterAction
3. **DTOs** — CreateDto (class-validator), FilterDto (ids, organizationIds), ResponseDto (fromEntity)
4. **Service** — regras de négocio, validacão, conversão centavos↔Decimal, CRUD via DAO
5. **Controller** — thin, delegacão, @ApiTags, @ApiResponse, @PinbankProtected
6. **Module** — wiring, exports

### Checklist de Feature

- [ ] Entity com campos privados + getters públicos
- [ ] Entity usa `super(p) + if (!p) return` no constructor
- [ ] Valores financeiros com @DecimalJsonSerializer + Decimal
- [ ] DAO com findAll sobrescrito (filterIdsAction + queryFilterAction)
- [ ] DAO sem lógica de négocio
- [ ] Request DTO com class-validator + @ApiProperty (centavos como number)
- [ ] Response DTO com factory fromEntity() (Decimal → centavos)
- [ ] Service usa `findAll({ filters: { ids } })` para buscar por ID
- [ ] Service valida com HTTPError estático
- [ ] Create/Update/Delete usam batch (com ou sem transaction)
- [ ] Controller thin (só delega), sem lógica
- [ ] @ApiTags + @ApiResponse em todos endpoints
- [ ] Guard correto (health sem auth)
- [ ] Module com exports dos providers
