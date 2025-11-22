---
title: JavaScriptのプロトタイプベースOOPについて
tags:
  - JavaScript
  - オブジェクト指向
  - プロトタイプ
  - prototype
private: false
updated_at: '2025-11-22T11:36:43+09:00'
id: c898d191dd012a22b631
organization_url_name: null
slide: false
ignorePublish: false
---
# はじめに

JavaScriptは**プロトタイプベース**のオブジェクト指向プログラミング言語です。これは、JavaやC++のような**クラスベース**のオブジェクト指向とは異なるアプローチを取っています。この記事では、JavaScriptのプロトタイプベースOOPの仕組みと特徴を解説します。

## クラスベース vs プロトタイプベース

### クラスベースOOP
- オブジェクトはクラスから生成される
- クラスは設計図として機能
- 継承はクラス間で行われる

### プロトタイプベースOOP
- オブジェクトから直接オブジェクトを生成できる
- クラスの概念は存在しない(ES6のclassは糖衣構文)
- 継承はオブジェクト間で行われる

## プロトタイプチェーン

JavaScriptでは、すべてのオブジェクトは内部的に`prototype`というプロパティを持っています。これは他のオブジェクトへの参照で、プロトタイプチェーンを形成します。

```javascript
const animal = {
  eat: function() {
    console.log('食べています');
  }
};

const dog = Object.create(animal);
dog.bark = function() {
  console.log('ワンワン!');
};

dog.eat();  // '食べています' - animalから継承
dog.bark(); // 'ワンワン!' - dog自身のメソッド
```

## コンストラクタ関数とプロトタイプ

従来のJavaScriptでは、コンストラクタ関数とプロトタイプを使ってオブジェクトを作成します。

```javascript
function Person(name, age) {
  this.name = name;
  this.age = age;
}

// プロトタイプにメソッドを追加
Person.prototype.greet = function() {
  console.log(`こんにちは、${this.name}です。${this.age}歳です。`);
};

const taro = new Person('太郎', 25);
const hanako = new Person('花子', 30);

taro.greet();   // 'こんにちは、太郎です。25歳です。'
hanako.greet(); // 'こんにちは、花子です。30歳です。'

console.log(taro.greet === hanako.greet); // true - 同じメソッドを共有
```

### なぜプロトタイプにメソッドを定義するのか?

```javascript
// ❌ 非効率的な方法
function Person(name) {
  this.name = name;
  this.greet = function() {
    console.log(`Hello, ${this.name}`);
  };
}

// ✅ 効率的な方法
function Person(name) {
  this.name = name;
}
Person.prototype.greet = function() {
  console.log(`Hello, ${this.name}`);
};
```

プロトタイプにメソッドを定義すると、すべてのインスタンスで同じメソッドを共有できるため、メモリ効率が向上します。

## プロトタイプ継承

プロトタイプチェーンを使って継承を実現できます。

```javascript
function Animal(name) {
  this.name = name;
}

Animal.prototype.eat = function() {
  console.log(`${this.name}は食べています`);
};

function Dog(name, breed) {
  Animal.call(this, name); // 親コンストラクタを呼び出し
  this.breed = breed;
}

// プロトタイプチェーンを設定
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;

Dog.prototype.bark = function() {
  console.log(`${this.name}はワンワン鳴いています`);
};

const pochi = new Dog('ポチ', '柴犬');
pochi.eat();  // 'ポチは食べています'
pochi.bark(); // 'ポチはワンワン鳴いています'
```

## ES6のクラス構文

ES6で導入された`class`構文は、内部的にはプロトタイプベースの仕組みを使っています。

```javascript
class Person {
  constructor(name, age) {
    this.name = name;
    this.age = age;
  }

  greet() {
    console.log(`こんにちは、${this.name}です。${this.age}歳です。`);
  }
}

class Student extends Person {
  constructor(name, age, grade) {
    super(name, age);
    this.grade = grade;
  }

  study() {
    console.log(`${this.name}は${this.grade}年生で勉強中です`);
  }
}

const student = new Student('次郎', 20, 2);
student.greet(); // 'こんにちは、次郎です。20歳です。'
student.study(); // '次郎は2年生で勉強中です'

// 内部的にはプロトタイプを使用
console.log(typeof Student); // 'function'
console.log(Student.prototype.study); // [Function: study]
```

## プロトタイプの確認方法

```javascript
const obj = {};

// プロトタイプの取得
console.log(Object.getPrototypeOf(obj)); // Object.prototype
console.log(obj.__proto__); // 非推奨だが使用可能

// プロトタイプチェーンの確認
console.log(obj instanceof Object); // true

// プロパティの所有確認
console.log(obj.hasOwnProperty('toString')); // false
console.log(Object.prototype.hasOwnProperty('toString')); // true
```

## プロトタイプの動的な変更

JavaScriptのプロトタイプは動的に変更できます。

```javascript
function Cat(name) {
  this.name = name;
}

const tama = new Cat('タマ');

// 後からプロトタイプにメソッドを追加
Cat.prototype.meow = function() {
  console.log(`${this.name}はニャーと鳴きます`);
};

tama.meow(); // 'タマはニャーと鳴きます' - 既存のインスタンスでも使える!
```

## プロトタイプベースOOPの利点

1. **柔軟性**: オブジェクトを動的に拡張できる
2. **メモリ効率**: メソッドをプロトタイプで共有できる
3. **動的な継承**: 実行時にプロトタイプチェーンを変更できる

## デメリット

1. **理解の難しさ**: クラスベースOOPに慣れた開発者にとって、プロトタイプチェーンの概念は直感的でない場合がある
2. **予期しない動作**: プロトタイプの動的な変更が既存のすべてのインスタンスに影響するため、意図しない副作用が発生する可能性がある

## まとめ

JavaScriptのプロトタイプベースOOPは、クラスベースOOPとは異なるアプローチですが、非常に強力で柔軟な仕組みです。ES6のclass構文を使っても、内部的にはプロトタイプが使われているため、プロトタイプの理解はJavaScript開発者にとって重要です。

プロトタイプチェーンを理解することで、JavaScriptの継承、メソッドの共有、そしてオブジェクト間の関係性をより深く理解できるようになります。

## 参考リンク

- [MDN - 継承とプロトタイプチェーン](https://developer.mozilla.org/ja/docs/Web/JavaScript/Inheritance_and_the_prototype_chain)
- [MDN - Object.prototype](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Global_Objects/Object/prototype)

