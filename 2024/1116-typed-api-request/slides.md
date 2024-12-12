---
theme: '@euxn23/slidev-theme-bricks'
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
mdc: true
fonts:
  sans: 'Kosugi Maru'
  serif: 'Kosugi Maru'
# for title page
layout: cover
---

# 型付きAPIリクエストを実現する
# いくつかの手法とその選択

<Footnote>

Presented by ユーン

@TSKaigi Kansai 2024

</Footnote>

---
layout: intro
---

<template v-slot:title>

# 自己紹介

</template>

<template v-slot:left>
  <div class="flex h-full content-center items-center">
    <img class="w-36 h-36 rounded-full" src="/images/icon.png" alt="euxn23 github icon" />
    <img class="w-36 h-36 rounded-full" src="/images/miyu.png" alt="euxn23 sns icon" />
  </div>
</template>

## ユーン (@euxn23)

- ドワンゴ教育事業のエンジニア
- ZEN ID という認証基盤を作っています
  - Web フロント
  - 前段バックエンド (TypeScript)
- RABBIT 小隊が好き

---
layout: center
---

# 2025年4月 ZEN大学 開学

---
layout: center
---

<img class="h-128" src="/images/zen-university.png" alt="ZEN大学">

---
layout: center
---

# このセッションでは、
# 安全な API リクエストを実現する
# いくつかの手法を紹介します

---
layout: side-by-side
---

<template v-slot:title>

## API 連携の安全性確保のための手法

</template>

<template v-slot:left>

## コードファーストでない

- API 仕様書 (NOT OpenAPI) を書く
- 結合テストを増やす
- 監視で異常系を発見する

</template>

## コードファーストな

- 実装や定義を共有する
- OpenAPI を実装と結びつける
- フレームワークの機能に乗る

---
layout: list
---

# コードファーストでない API 連携の課題

- API 仕様書<br>→ 仕様と実装の乖離
- 結合テスト<br>→ テストケースが多い
- 監視<br>→ 発見までにエラーは発生してしまう

---
layout: list
---

# コードファーストにするとどう解決されるか

- API仕様書<br>→ 仕様 (OpenAPI) と実装が関連付く、型により守られる
- 結合テスト<br>→ 型で表現できるレベルのテストケースは省略可能になる
- 監視<br>→ 型によりコーディングタイムでバグを発見しやすくなる

---
layout: center
---

## コードファーストなアプローチを考える

---
layout: side-by-side
---

<template v-slot:title>

## コードファーストなアプローチ

</template>

<template v-slot:left>

## TypeScript ファーストな手法

- 型定義の共有
- Zod スキーマの共有
- フレームワークの機能の利用
  - Hono RPC
  - tRPC

</template>

## 言語に依存しない手法

- サーバコードから OpenAPI を<br>自動生成
- OpenAPI からクライアントコードを自動生成

---
layout: side-by-side
---

<template v-slot:title>

## コードファーストなアプローチ

</template>

<template v-slot:left>

## TypeScript ファーストな手法

- 型定義の共有
- Zod スキーマの共有
- フレームワークの機能の利用
  - Hono RPC
  - tRPC

</template>

<Opacity opacity="40">

## 言語に依存しない手法

- サーバコードから OpenAPI を<br>自動生成
- OpenAPI からクライアントコードを自動生成

</Opacity>

---
layout: center
---

## 注意: ほとんどのケースで monorepo を前提とします

---
layout: code
---

<Title>

## 1. 型定義の共有
</Title>

サーバ側の型定義
```typescript
export type GetPetRequest = {
  param: {
    id: string
  }
};
export type GetPetResponse = Pet;
export type PostPetRequest = {
  body: Pet
};
export type PostPetRequest = never;
```

---
layout: code
---

<Title>

## 1. 型定義の共有
</Title>

クライアント側の実装
```typescript
import type { GetPetRequest, GetPetResponse, PostPetRequest, PostPetResponse } from '@/server';

export async function requestGetPet(req: GetPetRequest): Promise<GetPetResponse> {
  return fetch(`/api/pets/${req.param.id}`).then(res => res.json());
}

export async function requestPostPet(req: PostPetRequest): Promise<PostPetResponse> {
  return fetch(`/api/pets`, { method: 'POST', body: JSON.stringify(req.body) })
          .then(res => res.json());
}
```

```typescript
const pet = await requestGetPet({ param: { id: '1' } });
const newPet = {
  // ...
};
await requestPostPet({ body: newPet });
```


---
layout: side-by-side
---

<template v-slot:title>

## 1. 型定義の共有

</template>

<template v-slot:left>

## Pros

- 外部ライブラリや特別な設計に<br>依存しないので簡単に始めやすい

</template>

## Cons

- fetch の中は型がわからないまま<br>書くしかない
- API client のレイヤが別れている<br>必要がある

---
layout: code
---

<Title>

## 2. Zod スキーマの共有
</Title>

共通コード
```typescript
const PetTagSchema = z.object({
  id: z.number(),
  name: z.string(),
});

const PetSchema = z.object({
  id: z.string(),
  name: z.string(),
  tags: z.array(PetTagSchema),
});
```

<Footnote>

[https://github.com/colinhacks/zod](https://github.com/colinhacks/zod)

</Footnote>

---
layout: code
---

<Title>

## 2. Zod スキーマの共有
</Title>

クライアント側の実装
```typescript
async function postPet(pet: z.infer<typeof PetSchema>): Promise<void> {
  const result = PetSchema.safeParse(pet);
  if (!result.success) {
    throw new Error('Invalid pet');
  }
  await fetch('/api/pets', { method: 'POST', body: JSON.stringify(pet) });
}
```

---
layout: code
---

<Title>

## 2. Zod スキーマの共有

</Title>

サーバ側の実装(例: Express)
```typescript
app.post('/pets', async (req, res) => {
  const result = PetSchema.safeParse(req.body);
  if (!result.success) {
    res.status(400).json(result.error);
    return;
  }
  const pet = result.data;
  const inserted = await insertPet(pet)
  res.json(inserted);
  return;
});
```


---
layout: side-by-side
---

<template v-slot:title>

## 2. Zod スキーマの共有

</template>

<template v-slot:left>

## Pros

- バリデーションと型定義が一致する
- バリデーションロジックごと<br>サーバ・クライアントで共有できる

</template>

## Cons

- schema のプロパティ変更は infer した型の変数には波及しないため、type で引き回す方が良い
- 型定義と Zod の Schema の二重管理が発生するが、乖離を防ぐため次のような工夫が必要

---
layout: code
---

工夫の例

```typescript
export type PetTag = {
  id: number;
  name: string;
};

export type Pet = {
  id: string;
  name: string;
  tags: PetTag[];
};
```

```typescript
const PetTagSchema = z.object({
  id: z.number(),
  name: z.string(),
}) satisfies ZodType<PetTag>;

const PetSchema = z.object({
  id: z.string(),
  name: z.string(),
  tags: z.array(PetTagSchema),
}) satisfies ZodType<Pet>;
```

---
layout: code
---

<Title>

## 3. フレームワークの機能の利用 - Hono RPC
</Title>

```typescript
const route = app.post(
  '/posts',
  zValidator(
    'form',
    z.object({
      title: z.string(),
      body: z.string(),
    })
  ),
  (c) => {
    // ...
    return c.json({
      ok: true,
      message: 'Created!',
    }, 201)
  }
)

export type AppType = typeof route
```

<Footnote>

[https://hono.dev/docs/guides/rpc](https://hono.dev/docs/guides/rpc)

</Footnote>

---
layout: code
---

<Title>

## 3. フレームワークの機能の利用 - Hono RPC
</Title>

```typescript
import { AppType } from '.'
import { hc } from 'hono/client'

const client = hc<AppType>('http://localhost:8787/')

const res = await client.posts.$post({
  form: {
    title: 'Hello',
    body: 'Hono is a cool project',
  },
})

if (res.ok) {
  const data = await res.json()
  console.log(data.message)
}
```

---
layout: side-by-side
---

<template v-slot:title>

## TypeScript ファーストな手法
</template>

<template v-slot:left>

## Pros

- コードの共有から小さく始めやすい
- フレームワークの機能に乗れると<br>高速に開発ができる

</template>

## Cons

- TypeScript であることに強く依存<br>→**片方を別の言語で書き換えると、<br>&emsp;安全性は失われてしまう**

---
layout: center
---

## TypeScript に依存してしまっては、<br>柔軟さか安全さのどちらかが将来失われてしまう

---
layout: statement
---

## アバンパート終了

---
layout: center
---

## どうする！？ TypeScript と心中するか！？

---
layout: center
---

# 突如そこに OpenAPI が！

---
layout: statement
---

# OpenAPI ベースな<br>型付き API リクエスト

---
layout: side-by-side
---

<template v-slot:title>

## コードファーストなアプローチ

</template>

<template v-slot:left>

<Opacity opacity="40">

## TypeScript ファーストな手法

- 型定義の共有
- Zod スキーマの共有
- フレームワークの機能の利用
  - Hono RPC
  - tRPC

</Opacity>

</template>

## 言語に依存しない手法

- サーバコードから OpenAPI を<br>自動生成
- OpenAPI からクライアントコードを自動生成

---
layout: list
---

# OpenAPI のよいところ

- 言語非依存
- 仕様が標準化されている
- エコシステムが成熟しており、各言語にツールがある
- 既存の実装を損なわずに採用できるケースがままある
- etc...

---
layout: list
---


## なぜ gRPC や GraphQL ではないか

- gRPC
  - HTTP/2 が必要
  - プロトコルバッファの表現力の問題
  - Go まわり以外、エコシステムがあまり成熟していない
- GraphQL
  - そもそもが根本的に難しい
  - バックエンドの実装負担が大きい
  - 既存の実装を GraphQL 化するのは難しい

---
layout: list
---

## OpenAPIと実装を繋ぐ手法

- サーバコードから OpenAPI を生成
- OpenAPI から API クライアントを生成
- OpenAPI からサーバコードを生成

---
layout: center
---

## サーバコードから OpenAPI を自動生成

---
layout: list
---

<Title>

## サーバコードから OpenAPI を生成
</Title>

## サーバコードから OpenAPI を<br>生成する機能を持つフレームワーク

- NestJS
- Hono (@hono/zod-openapi)
- FastAPI (Python)
- etc...

---
layout: code
---

<Title>

## NestJS のサンプルコード
</Title>

```typescript
export class Cat {
  /**
   * The name of the Cat
   * @example Kitty
   */
  name: string;

  @ApiProperty({ example: 1, description: 'The age of the Cat' })
  age: number;

  @ApiProperty({
    example: 'Maine Coon',
    description: 'The breed of the Cat',
  })
  breed: string;
}
```

<Footnote>

[https://github.com/nestjs/nest/tree/master/sample/11-swagger](https://github.com/nestjs/nest/tree/master/sample/11-swagger)

</Footnote>

---
layout: code
---

```typescript
@ApiBearerAuth()
@ApiTags('cats')
@Controller('cats')
export class CatsController {
  constructor(private readonly catsService: CatsService) {}

  @Post()
  @ApiOperation({ summary: 'Create cat' })
  @ApiResponse({ status: 403, description: 'Forbidden.' })
  async create(@Body() createCatDto: CreateCatDto): Promise<Cat> {
    return this.catsService.create(createCatDto);
  }

  @Get(':id')
  @ApiResponse({
    status: 200,
    description: 'The found record',
    type: Cat,
  })
  findOne(@Param('id') id: string): Cat {
    return this.catsService.findOne(+id);
  }
}
```

---
layout: code
---

<Title>

## @hono/zod-openapi のサンプルコード
</Title>

```typescript
const route = createRoute({
  method: 'get',
  path: '/users/{id}',
  request: {
    params: ParamsSchema,
  },
  responses: {
    200: {
      content: {
        'application/json': {
          schema: UserSchema,
        },
      },
      description: 'Retrieve the user',
    },
  },
})
```

<Footnote>

[https://hono.dev/examples/zod-openapi](https://hono.dev/examples/zod-openapi)

</Footnote>

---
layout: code
---

```typescript
const app = new OpenAPIHono()

app.openapi(route, (c) => {
  const { id } = c.req.valid('param')
  return c.json({
    id,
    age: 20,
    name: 'Ultra-man',
  })
})

// The OpenAPI documentation will be available at /doc
app.doc('/doc', {
  openapi: '3.0.0',
  info: {
    version: '1.0.0',
    title: 'My API',
  },
})
```

---
layout: center
---

## FastAPI の例は省略 (Pythonなので)

---
layout: side-by-side
---

<template v-slot:title>

# サーバ実装から OpenAPI を生成することの是非
</template>

<template v-slot:left>

## Pros

- 仕様と実装が一致する
- 常に最新になる
- OpenAPI を手書きしなくてよい

</template>

## Cons

- 実装を変更しないと OpenAPI を変更できない
  - まだ実装されていないが変更予定のものを OpenAPI にしにくい
  - OpenAPI を中心として議論しにくい

---
layout: center
---

## 誰がOpenAPIを書くか (=API仕様を決めるか)<br>に応じて選択しましょう
## → みんなで書くなら実装ベースじゃない方が<br>議論もしやすい

---
layout: center
---

## OpenAPI から API クライアントを生成

---
layout: list
---

## OpenAPI から API クライアントを<br>生成するライブラリ

- openapi-fetch
- orval
- etc...

---
layout: list
---

# openapi-typescript / openapi-fetch

- openapi-typescript は OpenAPI から TS コードを生成
- openapi-fetch はこれを元に API Client コードを生成

<Footnote>

[https://openapi-ts.dev/](https://openapi-ts.dev/)

[https://openapi-ts.dev/openapi-fetch/](https://openapi-ts.dev/openapi-fetch/)

</Footnote>

---
layout: code
---

<Title>

## openapi-typescript のサンプルコード
</Title>

```typescript
import { paths, components } from "./path/to/my/schema"; // <- generated by openapi-typescript

// Schema Obj
type MyType = components["schemas"]["MyType"];

// Path params
type EndpointParams = paths["/my/endpoint"]["parameters"];

// Response obj
type SuccessResponse = paths["/my/endpoint"]["get"]["responses"][200]["content"]["application/json"]["schema"];
type ErrorResponse = paths["/my/endpoint"]["get"]["responses"][500]["content"]["application/json"]["schema"];
```

<Footnote>

[https://openapi-ts.dev/introduction](https://openapi-ts.dev/introduction)

</Footnote>

---
layout: code
---

<Title>

## openapi-fetch のサンプルコード
</Title>

```typescript
import createClient from "openapi-fetch";
import type { paths } from "./my-openapi-3-schema"; // generated by openapi-typescript

const client = createClient<paths>({ baseUrl: "https://myapi.dev/v1/" });

const {
  data, // only present if 2XX response
  error, // only present if 4XX or 5XX response
} = await client.GET("/blogposts/{post_id}", {
  params: {
    path: { post_id: "123" },
  },
});

await client.PUT("/blogposts", {
  body: {
    title: "My New Post",
  },
});
```

<Footnote>

[https://openapi-ts.dev/openapi-fetch/](https://openapi-ts.dev/openapi-fetch/)

</Footnote>

---
layout: list
---

# orval

- Axios ベースの API Client コードを生成
- TanStack Query や SWR、Zod との連携もある

<Footnote>

[https://orval.dev/](https://orval.dev/)

</Footnote>

---
layout: code
---

<Title>

## orval のサンプルコード
</Title>

```typescript
import type { CreatePetsBody } from '../model';
import { customInstance } from '../mutator/custom-instance';

export const createPets = (
        createPetsBody: CreatePetsBody,
        version: number = 1,
) => {
  return customInstance<void>({
    url: `/v${version}/pets`,
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    data: createPetsBody,
  });
}; 
```

<Footnote>

[https://github.com/anymaniax/orval/blob/master/samples/react-app](https://github.com/anymaniax/orval/blob/master/samples/react-app)

</Footnote>

---
layout: side-by-side
---

<template v-slot:title>

## OpenAPI から API クライアントを生成
</template>

<template v-slot:left>

## Pros

- 実装が変わった際に型エラーで<br>検知できる
- エンドポイントのパスや<br>パスパラメータも<br>補完・チェックできる

</template>

## Cons

- OpenAPI を実装から生成していない場合、実装との乖離は起こりうる

---
layout: center
---

## OpenAPI からサーバコードを生成

---
layout: list
---

<Title>

## OpenAPI からサーバコードを生成
</Title>

## 残念ながら、TS ではメジャーな(十分枯れた)ものはない

- 例えば golang なら oapi-codegen や ogen などがある
- ogen は interface を生成、それに合わせて実装する
- レイヤが別れるので後入れは難しい、開発初期なら検討の余地あり

---
layout: center
---

## Q. ところで OpenAPI は yaml を手書きするの？
## A. OpenAPI を書くための次世代 DSL、TypeSpec がある

---
layout: list
---

# TypeSpec

- TypeScript / C# 風味の DSL で OpenAPI Schema を書けるライブラリ
- LanguageServer を提供されており、VSCode 向けにプラグインを提供
- コンパイル時に valid な記法か確認されるので安心

<Footnote>

[https://typespec.io/](https://typespec.io/)

</Footnote>

---
layout: code
---

```tsp
import "@typespec/http";

using TypeSpec.Http;

model PetTag {
  id: number;
  name: string;
};

model Pet {
  id: string;
  name: string;
  tags: PetTag[];
}

@route("/pets/{id}")
interface Stores {
  @opetationId("get-pet")
  @summary("Get a pet by ID")
  @get
  get(@path id: string): {
    @statusCode
    statusCode: 200;
    @body Pet;
  };
}
```

---
layout: list
---

## yaml 手書きと比べて

- 複雑でない module システムを備え、ファイルの分割や再利用が容易
- デフォルト値が設定されているものは省略可能であり、文量が少ない
- monorepoにも対応、別パッケージの定義を参照することも可能

---
layout: list
---

# まとめ

- サーバとクライアントの TS コード共有は楽だが、<br>結合レイヤの TS 依存はリスクである
- 結合レイヤはエコシステムが充実しており、<br>言語非依存の OpenAPI を用いるのが良いのではないか
- OpenAPI により仕様と実装を近づけることで<br>コード面の安全性や開発効率の向上が期待できる

---
layout: list
---

<Title>

## 想定質問
</Title>

## Q. OpenAPI とバックエンドの乖離はどうするの？<br>A. テストケースを書いて頑張ろう(ここは課題です)

<div>

結局バックエンドの実装と仕様を一致させるのが一番大変ですよね

でも、乖離しうる箇所を減らせているだけでも大きな進歩です

テスト用のクライアントコードを OpenAPI から生成するなどして頑張りましょう

</div>

---
layout: center
---

# Any question?

---
layout: center
---

## ドワンゴのスポンサーブースにいます、遊びに来てね
## 質問やディスカッションも大歓迎

<img src="/images/hiring.png" alt="ドワンゴ教育事業はTSエンジニアを採用しています" class="h-96">

---
layout: statement
---

# ありがとうございました

---
layout: list
---

## Appendix: 過去に話した関連する登壇資料

- [NestJS アプリケーションから Swagger を自動生成する](https://speakerdeck.com/euxn23/nestjs-meetup-tokyo-01)
- [Powerfully Typed TypeScript](https://speakerdeck.com/euxn23/powerfully-typed-typescript)
- [TypeSpec を使い倒してる](https://speakerdeck.com/euxn23/use-up-typespec)

---
layout: list
---

## Appendix: 関連する有益な参考資料

- [WebフロントエンドにおけるGraphQL（あるいはバックエンドのAPI）との向き合い方](https://speakerdeck.com/izumin5210/number-241106-plk-frontend)
- [見よ、これがHonoのRPCだ](https://zenn.dev/yusukebe/articles/a00721f8b3b92e)
- [Hono × Zod-OpenAPIで快適API開発](https://zenn.dev/slowhand/articles/b7872e09b84e15)
- [最近のGoのOpenAPI Generatorの推しはogen](https://blog.p1ass.com/posts/ogen/)

