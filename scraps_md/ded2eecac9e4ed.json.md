---
title: "Zig 0.13.0で多相（ポリモーフィズム）"
closed: true
archived: false
created_at: "2025-01-18"
---

---

```
"author": "funatsufumiya",
"created_at": "2025-01-18",
"body_updated_at": "2025-01-18"
```

実験結果のGitリポジトリ。各スクラップに対応してlib.zig 〜 lib7.zig がある。
https://github.com/funatsufumiya/zig-polymorphism-study

---

```
"author": "funatsufumiya",
"created_at": "2025-01-18",
"body_updated_at": "2025-01-23"
```

今回取り扱うのはトレイト的なもので、いわゆるアドホック多相。`anytype`を使えば簡単に実装できるけど、できれば型名をコメント以外で明記したいところ。（Zigはいわば、コンパイル時ダックタイピングのようなことをする言語なので、Zig的にはこれで良いのかもしれないが。）

```zig
const std = @import(\"std\");
const testing = std.testing;

const Cat = struct {
    pub fn meow(_: *Cat) []const u8 {
        return \"meow\";
    }

    pub fn voice(self: *Cat) []const u8 {
        return self.meow();
    }
};

const Dog = struct {
    pub fn bow(_: *Dog) []const u8 {
        return \"bow wow\";
    }

    pub fn voice(self: *Dog) []const u8 {
        return self.bow();
    }
};

pub fn animalVoice(animal: anytype) void { // <- ここに型名を示したい。
    animal.voice();
}

test \"animal voice\" {
    var cat = Cat{};
    var dog = Dog{};

    try testing.expectEqualSlices(u8, \"meow\", cat.voice());
    try testing.expectEqualSlices(u8, \"bow wow\", dog.voice());
}
```

---

```
"author": "funatsufumiya",
"created_at": "2025-01-18"
```

次に`anyopaque`を使った方法。`anytype`を使うよりも意図は明確。ただ、共通の関数が多い時は辛そう。

```zig
const std = @import(\"std\");
const testing = std.testing;

const Animal = struct {
    voiceFn: *const fn (self: *anyopaque) []const u8,

    pub fn voice(self: *Animal) []const u8 {
        return self.voiceFn(self);
    }
};

const Cat = struct {
    pub fn meow(_: *Cat) []const u8 {
        return \"meow\";
    }

    pub fn asAnimal(_: *Cat) Animal {
        return .{
            .voiceFn = struct {
                fn voice(ptr: *anyopaque) []const u8 {
                    const self_ptr = @as(*Cat, @ptrCast(@alignCast(ptr)));
                    return self_ptr.meow();
                }
            }.voice,
        };
    }
};

const Dog = struct {
    pub fn bow(_: *Dog) []const u8 {
        return \"bow wow\";
    }

    pub fn asAnimal(_: *Dog) Animal {
        return .{
            .voiceFn = struct {
                fn voice(ptr: *anyopaque) []const u8 {
                    const self_ptr = @as(*Dog, @ptrCast(@alignCast(ptr)));
                    return self_ptr.bow();
                }
            }.voice,
        };
    }
};

pub fn animalVoice(animal: *Animal) []const u8 {
    return animal.voice();
}

test \"animal voice with interface\" {
    var cat = Cat{};
    var dog = Dog{};
    
    var cat_animal = cat.asAnimal();
    var dog_animal = dog.asAnimal();

    try testing.expectEqualSlices(u8, \"meow\", animalVoice(&cat_animal));
    try testing.expectEqualSlices(u8, \"bow wow\", animalVoice(&dog_animal));
}
``

---

```
"author": "funatsufumiya",
"created_at": "2025-01-18",
"body_updated_at": "2025-01-18"
```

インターフェースの方で、インスタンスおよびvtableを持つようにする方法。スッキリしてきたけど、`init`で関数ごとの実装が必要なのがやや冗長で、関数が多くなってくると辛そうだけど、とはいえ子クラスが増えても修正の必要はないので、今のところこれが一番スッキリか？（もっと良い方法がみつかったら追記したい。）

```zig
const std = @import(\"std\");
const testing = std.testing;

const Animal = struct {
    vtable: *const VTable,
    instance: *anyopaque,

    const VTable = struct {
        voiceFn: *const fn (*anyopaque) []const u8,
        nameFn: *const fn (*anyopaque) []const u8,
    };

    pub fn voice(self: *const Animal) []const u8 {
        return self.vtable.voiceFn(self.instance);
    }

    pub fn name(self: *const Animal) []const u8 {
        return self.vtable.nameFn(self.instance);
    }

    pub fn init(comptime T: type, instance: *T) Animal {
        const vtable = comptime VTable{
            .voiceFn = struct {
                fn func(ptr: *anyopaque) []const u8 {
                    const self = @as(*T, @ptrCast(@alignCast(ptr)));
                    return self.voice();
                }
            }.func,
            .nameFn = struct {
                fn func(ptr: *anyopaque) []const u8 {
                    const self = @as(*T, @ptrCast(@alignCast(ptr)));
                    return self.name();
                }
            }.func,
        };
        return .{
            .vtable = &vtable,
            .instance = instance,
        };
    }
};

const Cat = struct {
    name_str: []const u8,

    pub fn voice(_: *const Cat) []const u8 {
        return \"meow\";
    }

    pub fn name(self: *const Cat) []const u8 {
        return self.name_str;
    }

    pub fn asAnimal(self: *Cat) Animal {
        return Animal.init(Cat, self);
    }
};

const Dog = struct {
    name_str: []const u8,

    pub fn voice(_: *const Dog) []const u8 {
        return \"bow wow\";
    }

    pub fn name(self: *const Dog) []const u8 {
        return self.name_str;
    }

    pub fn asAnimal(self: *Dog) Animal {
        return Animal.init(Dog, self);
    }
};

pub fn animalVoice(animal: *const Animal) []const u8 {
    return animal.voice();
}

pub fn animalName(animal: *const Animal) []const u8 {
    return animal.name();
}

test \"animal voice and name with interface\" {
    var cat = Cat{ .name_str = \"Tama\" };
    var dog = Dog{ .name_str = \"Pochi\" };
    
    var cat_animal = cat.asAnimal();
    var dog_animal = dog.asAnimal();

    try testing.expectEqualSlices(u8, \"meow\", animalVoice(&cat_animal));
    try testing.expectEqualSlices(u8, \"bow wow\", animalVoice(&dog_animal));
    try testing.expectEqualSlices(u8, \"Tama\", animalName(&cat_animal));
    try testing.expectEqualSlices(u8, \"Pochi\", animalName(&dog_animal));
}
```

---

```
"author": "funatsufumiya",
"created_at": "2025-01-18",
"body_updated_at": "2025-01-18"
```

init内の、`const self = @as(*T, @ptrCast(@alignCast(ptr)));` を関数を使って共通化した例。
これで若干init書くのは楽になるかな？（[GitHubのREADME](https://github.com/funatsufumiya/zig-polymorphism-study)には、この**フルバージョン**を記載。）

```zig
// （Animal以外の定義は先に同じ。）

const Animal = struct {
    vtable: *const VTable,
    instance: *anyopaque,

    const VTable = struct {
        voiceFn: *const fn (*anyopaque) []const u8,
        nameFn: *const fn (*anyopaque) []const u8,
    };

    fn castTo(comptime T: type, ptr: *anyopaque) *T {
        return @as(*T, @ptrCast(@alignCast(ptr)));
    }

    pub fn voice(self: *const Animal) []const u8 {
        return self.vtable.voiceFn(self.instance);
    }

    pub fn name(self: *const Animal) []const u8 {
        return self.vtable.nameFn(self.instance);
    }

    pub fn init(comptime T: type, instance: *T) Animal {
        const vtable = comptime VTable{
            .voiceFn = struct {
                fn func(ptr: *anyopaque) []const u8 {
                    const self = Animal.castTo(T, ptr);
                    return self.voice();
                }
            }.func,
            .nameFn = struct {
                fn func(ptr: *anyopaque) []const u8 {
                    const self = Animal.castTo(T, ptr);
                    return self.name();
                }
            }.func,
        };
        return .{
            .vtable = &vtable,
            .instance = instance,
        };
    }
};
```

---

```
"author": "funatsufumiya",
"created_at": "2025-01-18",
"body_updated_at": "2025-01-18"
```

さらにさらに、init内のメソッド作成ごと共通化したもの。ただ、返り値の型の変更や引数の柔軟性などを考慮していないので、正直一つ前の実装くらいが素直で良い気がする。

```zig
// （Animal以外の定義は先に同じ。）

const Animal = struct {
    vtable: *const VTable,
    instance: *anyopaque,

    const VTable = struct {
        voiceFn: *const fn (*anyopaque) []const u8,
        nameFn: *const fn (*anyopaque) []const u8,
    };

    fn castTo(comptime T: type, ptr: *anyopaque) *T {
        return @as(*T, @ptrCast(@alignCast(ptr)));
    }

    fn makeMethodCaller(
        comptime T: type,
        comptime method: []const u8,
    ) fn (*anyopaque) []const u8 {
        return struct {
            fn caller(ptr: *anyopaque) []const u8 {
                const self = Animal.castTo(T, ptr);
                return @field(T, method)(self);
            }
        }.caller;
    }

    pub fn voice(self: *const Animal) []const u8 {
        return self.vtable.voiceFn(self.instance);
    }

    pub fn name(self: *const Animal) []const u8 {
        return self.vtable.nameFn(self.instance);
    }

    pub fn init(comptime T: type, instance: *T) Animal {
        const vtable = comptime VTable{
            .voiceFn = makeMethodCaller(T, \"voice\"),
            .nameFn = makeMethodCaller(T, \"name\"),
        };
        return .{
            .vtable = &vtable,
            .instance = instance,
        };
    }
};
```

---

```
"author": "funatsufumiya",
"created_at": "2025-01-18",
"body_updated_at": "2025-01-18"
```

Blueskyに書いた内容だけど、一応転記。

> これ書いてて思ったけど、zigの「見えないフローはない」というのと言語自体にデフォルトのポリモーフィズムがないことによって、自分がほしいポリモーフィズムが好きに作れるというのは、ある意味良いかもしれないという気がした。
> 
> もし A or B みたいな型が欲しい場合は、typeor(A, B) みたいな関数[^1]も自作できるし、そう考えるとcomptimeすごい。

これについてさらにコメントするとすれば、ポリモーフィズムというのはある意味でコードをわかりにくくしてしまうので、C言語のように素直に`abs` / `fabs` / `labs` のように別関数にしてしまう方が、状況によってはわかりやすいのかもしれない。（あるいは最初の例のようにストレートに`anytype`使ってしまう[^2]とか。）

[^1]: zigは型自体を値として返すことができ、型を値として引数に渡せることに注意。
[^2]: なんとなくRubyのコンパイル時verという気がして、自分は個人的には好き。動的な処理でありながらもコンパイラの静的解析の恩恵が受けれるという、何とも不思議な気分。（もはやcomptimeは静的解析の域を超えているが…。）

---

```
"author": "funatsufumiya",
"created_at": "2025-01-23",
"body_updated_at": "2025-01-23"
```

ちなみに今回の記事の範疇を超えるものの、zigで総称型のようなものを作るときは、わざわざ `const XXX = struct { }` とせずに、関数で直接、**無名構造体**を返すケースが多い。これは、zigでは型とstructは（comptimeには）ほぼ同列に扱われるためで、zigに慣れていないととても不思議に感じるし、最初、変数の型には何を指定すれば良いのか一瞬戸惑う。（この辺はなんとなくJavaScriptに似ているような気もする。）

（詳しく知りたい方は以下の記事などが詳しい。）
https://zenn.dev/drumato/books/learn-zig-to-be-a-beginner/viewer/code-reading-stdlib-arraylist

