---
theme: bricks
highlighter: shiki
lineNumbers: false
drawings:
  persist: false
transition: slide-left
mdc: true
fonts:
  sans: 'Kosugi Maru'
  serif: 'Kosugi Maru'
---

# TypeSpec を使い倒してる

---
---

# 自己紹介

<div class="flex flex-col m-8 h-64 justify-center">
<div class="flex gap-8">
<img src="https://github.com/euxn23.png" alt="euxn23" class="w-48 h-48 rounded-full">
<div class="flex flex-col">

# ユーン (@euxn23)

<div class="text-2xl">

- 東京から来ました2
- ドワンゴで教育事業をやっています
- いまは何も聞かないでくれ

</div>
</div>
</div>
</div>

---
layout: statement
---

# JS と言ったら OpenAPI Spec
# (誇大表現)

---
layout: statement
---

# TypeSpec を使い倒して
# OpenAPI Spec を書く例を紹介

---
layout: statement
---

# What is TypeSpec

---
class: content-center
---

# [https://typespec.io/](https://typespec.io/)

- TypeScript / C# っぽい風味に API Spec を記述できる DSL/コンパイラ実装
- プログラミング言語的な作法でコード分割や再利用が可能
- OpenAPI Spec だけでなく JSON Schema としてや protobuf としての出力も可能

---
layout: full
class: content-center px-64
---

```tsp
@route("/pets")
namespace Pets {
  op list(@query skip: int32, @query top: int32): {
  @body pets: Pet[];
  };
  op read(@path petId: int32): {
  @body pet: Pet;
  };
  @post
  op create(@body pet: Pet): {};
}
```

https://typespec.io/docs/getting-started/getting-started-http#request--response-bodies

---
class: content-center text-center
---

## いつもの PetStore を書いてみる

---
layout: full
class: py-0 px-64
---

```tsp
enum Versions {
  v1,
}

model Pet = {
  id: int32;
  category: {
    id: int32;
    name: string;
  };
  name: string;
  photoUrls: string[];
  tags: {
    id: int32;
    name: string;
  }[];
  status: "available" | "pending" | "sold";
};

@service({
  title: "PetStore",
})
@versioned(Versions)
namespace PetStore {
  @operationId("findPetById")
  @summary("find-pet-by-id")
  @get findPetById(@path id: int32): {
    @body pet: Pet;
  };
}
```

---
class: content-center text-center
---

# ファイル分割してみる

---
layout: full
class: content-center px-64
---

## こういうディレクトリ構造を考える

```
spec/
├── main.tsp
├── namespace.tsp
├── routes/
│   ├── pet.tsp
│   ├── store.tsp
│   ├── user.tsp
│   └── main.tsp
└── models/
    ├── api-response.tsp
    ├── category.tsp
    ├── pet.tsp
    ├── tag.tsp
    ├── order.tsp
    ├── user.tsp
    └── main.tsp
```

---
layout: full
class: content-center px-64
---

main.tsp

```tsp
import "./routes";
import "./namespace";
```

---
layout: full
class: content-center px-64
---

namespace を空で宣言する namespace.tsp を作る

(各 route が namespace に子を生やしていく)

```tsp
import "@typespec/versioning";

using TypeSpec.Versioning;

enum Versions {
  v1,
}

@service({
  title: "PetStore",
})
@versioned(Versions)
namespace PetStore {}
```

---
layout: full
class: content-center px-64
---


main.tsp がディレクトリのエントリーポイントとして解決されるので、

models/main.tsp に配下の import を書く

```tsp
import "./api-response.tsp";
import "./category.tsp";
import "./pet.tsp";
import "./tag.tsp";
import "./order.tsp";
import "./user.tsp";
```

---
layout: full
class: content-center px-64
---

models/pet.tsp

```tsp
// import されたファイルに記載されている model は
// global を汚染するので具体的な名前をつける
model PetStatus = "available" | "pending" | "sold";

model Pet = {
  id: int32;
  category: {
    id: int32;
    name: string;
  };
  name: string;
  photoUrls: string[];
  tags: {
    id: int32;
    name: string;
  }[];
  status: PetStatus;
};
```

---
layout: full
class: content-center px-64
---

同様に routes/main.tsp に配下の import を書く

```tsp
import "./pet.tsp";
import "./store.tsp";
import "./user.tsp";
```

---
layout: full
class: py-0 px-64
---

routes/pet.tsp

```tsp
@route("/pet")
namespace PetStore.Pet {
  @route("/")
  interface Root {
    @operationId("post-pet")
    @summary("Add a new pet to the store")
    @post
    post(
      @body pet: Pet
    ): {
      @statusCode
      statusCode: 200 | 405;
      @body pet: Pet;
    };
    
    @operationId("put-pet")
    @summary("Update an existing pet")
    @put
    put(
      @body pet: Pet
    ): {
      @statusCode
      statusCode: 200 | 400 | 404 | 405;
      @body pet: Pet;
    };
  }
```
---
layout: full
class: py-0 px-64
---

```tsp
  @route("/{id}")
  namespace Id {
    @route("/")
    interface Root {
      @operationId("get-pet-by-id")
      @summary("Find pet by ID")
      @get
      get(
        @doc("ID of pet to return")
        @path
        id: string
      ): {
        @statusCode
        statusCode: 200 | 400 | 404;

        @body pet: Pet;
      };
      
      @operationId("post-pet-by-id")
      @summary("Update a pet in the store with form data")
      @post
      post(
        @header
        `content-type`: "multipart/form-data",
        @path
        id: string
        @multipartBody body: {
          name: HttpPart<string>;
          status: HttpPart<string>;
  ...
```

---
class: content-center text-center
---

# いいかんじ

---
layout: full
class: content-center px-64
---

こんなヘルパーを用意してもいいかもしれない

```tsp
alias OmitID<T> = OmitProperties<T, "id">;
```

```tsp
model EphemeralPet = OmitID<Pet>
```

---
class: content-center
---

# 現代の当たり前を、OpenAPI Spec でも

- 冗長な重複記述は再利用で DRY に
- VSCode のプラグインがあるので補完が効く
- 構文エラーはコンパイラが落としてくれる

---
class: content-center text-center
---

# 話し切れなかったことや具体的なテクは
# 後日ドワンゴ教育サービス開発者ブログで！！

## [blog.nnn.dev](https://blog.nnn.dev)

---
layout: statement
---

# ありがとうございました
