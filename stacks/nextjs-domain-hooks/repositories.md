---
paths:
  - "**/repository/**"
  - "**/*.repository.ts"
  - "**/services/**"
---

# Next.js Domain Hooks: Repositories

Every HTTP call lives in `src/repository/`, one `<resource>.repository.ts` per API resource. No other folder imports `httpClient`.

## Shape

```ts
// src/repository/product.repository.ts
import { httpClient } from "@/config/axios-client";
import type { CreateProductDto } from "@/domain/product/dto/create-product.dto";
import type { ListProductsDto } from "@/domain/product/dto/list-products.dto";
import type { Product } from "@/domain/product/entities/product";
import type { Paginated } from "@/domain/shared/entities/pagination";
import { HttpMethods, Service } from "@/services";

export class ProductRepository extends Service {
  public static key = "@product";
  public static defaultPath = "/products";

  public static async getAllProducts(): Promise<Product[]> {
    const method = HttpMethods.GET;
    const path = Service.endpoint(ProductRepository.defaultPath, "");
    const { data } = await httpClient[method](path);
    return data;
  }

  public static async listProducts(query?: ListProductsDto): Promise<Paginated<Product>> {
    const method = HttpMethods.GET;
    const path = Service.endpoint(ProductRepository.defaultPath, "/search");
    const { data } = await httpClient[method](path, { params: query });
    return data;
  }

  public static async createProduct(dto: CreateProductDto): Promise<Product> {
    const method = HttpMethods.POST;
    const path = Service.endpoint(ProductRepository.defaultPath, "");
    const { data } = await httpClient[method](path, dto);
    return data;
  }

  public static async deleteProduct(id: number): Promise<void> {
    const method = HttpMethods.DELETE;
    const path = Service.endpoint(ProductRepository.defaultPath, `/${id}`);
    await httpClient[method](path);
  }
}
```

- `key` is the query-key prefix, in the form `@<resource>`. It must be unique across Repositories.
- `defaultPath` is the resource's base path in the API.
- Each method picks an `HttpMethods` value, builds the path with `Service.endpoint(defaultPath, suffix)`, calls `httpClient[method]`, and declares its return type.
- GET filters go in `{ params: query }`. Bodies go as the second argument.
- A route outside `defaultPath` uses `Service.endpoint("", "/verify-email")`, with a comment saying why.
- For multipart bodies, pass `Service.toFormData(dto)`. It skips `undefined` and `null` fields and expands a `FileList` into one field per file.

## Registry

`src/repository/index.ts` registers every Repository in one static class:

```ts
import { OrderRepository } from "./order.repository";
import { ProductRepository } from "./product.repository";

export class Repository {
  static order = OrderRepository;
  static product = ProductRepository;
}
```

Domain hooks always reach Repositories through this registry, imported from `@/repository`. A new Repository is registered here, under its resource name in camelCase.

## `src/services/`

This folder belongs to the Repository layer. It is not the core Service role.

| Export | Job |
|---|---|
| `Service` | Base class: `key`, `defaultPath`, `endpoint()`, `toFormData()` |
| `HttpMethods` | Enum of the axios method names |
| `Key(ctor, method)` | Builds a query key: `"<ctor.key>:<method>"` |

## Rules

- Methods are static, and a Repository holds no state. Biome's `noStaticOnlyClass` is off for this reason.
- Refer to the class by name (`ProductRepository.defaultPath`), never `this`, because domain hooks pass methods unbound as `queryFn` and `mutationFn`.
- Import from `domain/` only types, with `import type`: entities and DTOs.
- Return the typed `data` as the API sends it. Responses are not parsed at runtime. The entity interface is the contract.
- Make no business decisions, hold no cache, and import no React, TanStack Query or zustand.
- Never catch errors here. They reach the ViewModel unchanged.
