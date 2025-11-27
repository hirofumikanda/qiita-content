---
title: MVCとMVVMアーキテクチャの違いを理解する
tags:
  - アーキテクチャ
  - MVC
  - MVVM
  - 設計パターン
  - ソフトウェア設計
private: false
updated_at: '2025-11-27T00:00:00+09:00'
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

# はじめに

ソフトウェア開発において、適切なアーキテクチャパターンを選択することは、保守性や拡張性の高いアプリケーションを構築する上で重要です。本記事では、代表的なアーキテクチャパターンであるMVCとMVVMの違いについて解説します。

## MVCアーキテクチャとは

MVC（Model-View-Controller）は、1970年代にSmalltalkで初めて導入された、歴史あるアーキテクチャパターンです。アプリケーションを3つの主要なコンポーネントに分割します。

### MVCの構成要素

#### Model（モデル）
- アプリケーションのデータとビジネスロジックを管理
- データベースとのやり取りやデータの検証を担当
- ViewやControllerから独立して動作

#### View（ビュー）
- ユーザーインターフェース（UI）を表示
- Modelのデータを視覚的に表現
- ユーザーからの入力を受け取る

#### Controller（コントローラー）
- ViewとModelの仲介役
- ユーザーの入力を処理し、Modelを更新
- Modelの変更に応じてViewを更新

### MVCのデータフロー

```
User Input → Controller → Model
                ↓           ↓
              View ←────────┘
```

1. ユーザーがViewで操作を行う
2. Controllerがユーザーのアクションを受け取る
3. ControllerがModelを更新
4. ModelがViewに通知（またはControllerがViewを更新）
5. Viewが更新されたデータを表示

### MVCの実装例（JavaScript）

```javascript
// Model
class TaskModel {
  constructor() {
    this.tasks = [];
    this.observers = [];
  }

  addTask(task) {
    this.tasks.push(task);
    this.notifyObservers();
  }

  getTasks() {
    return this.tasks;
  }

  notifyObservers() {
    this.observers.forEach(observer => observer.update());
  }

  subscribe(observer) {
    this.observers.push(observer);
  }
}

// View
class TaskView {
  constructor(model) {
    this.model = model;
    this.model.subscribe(this);
    this.taskListElement = document.getElementById('task-list');
  }

  render() {
    this.taskListElement.innerHTML = '';
    this.model.getTasks().forEach(task => {
      const li = document.createElement('li');
      li.textContent = task;
      this.taskListElement.appendChild(li);
    });
  }

  update() {
    this.render();
  }
}

// Controller
class TaskController {
  constructor(model, view) {
    this.model = model;
    this.view = view;
  }

  addTask(task) {
    if (task.trim()) {
      this.model.addTask(task);
    }
  }
}

// 使用例
const model = new TaskModel();
const view = new TaskView(model);
const controller = new TaskController(model, view);

// ユーザー操作
document.getElementById('add-button').addEventListener('click', () => {
  const input = document.getElementById('task-input');
  controller.addTask(input.value);
  input.value = '';
});
```

## MVVMアーキテクチャとは

MVVM（Model-View-ViewModel）は、2005年にMicrosoftのJohn GossmanによってWPFのために考案されたパターンです。MVCから進化し、データバインディングを活用したアーキテクチャです。

### MVVMの構成要素

#### Model（モデル）
- MVCと同様に、ビジネスロジックとデータを管理
- データソースとのやり取りを担当
- ViewやViewModelから独立

#### View（ビュー）
- UIの表示を担当
- ViewModelとデータバインディングで接続
- ロジックを持たず、表示のみに専念

#### ViewModel（ビューモデル）
- ViewとModelの仲介役
- Viewのための表示用データを保持
- Viewからの操作を処理し、Modelを更新
- データバインディングにより、Viewと自動的に同期

### MVVMのデータフロー

```
User Input → View ←→ ViewModel ←→ Model
             (Data Binding)
```

1. ユーザーがViewで操作を行う
2. データバインディングによりViewModelが自動的に更新
3. ViewModelがModelを更新
4. Modelの変更がViewModelに反映
5. データバインディングによりViewが自動的に更新

### MVVMの実装例（Vue.js）

```vue
<!-- View -->
<template>
  <div id="app">
    <h1>タスク管理</h1>
    <input 
      v-model="newTask" 
      @keyup.enter="addTask" 
      placeholder="新しいタスクを入力"
    />
    <button @click="addTask">追加</button>
    
    <ul>
      <li v-for="(task, index) in tasks" :key="index">
        {{ task }}
        <button @click="removeTask(index)">削除</button>
      </li>
    </ul>
    
    <p>タスク数: {{ taskCount }}</p>
  </div>
</template>

<script>
// Model
class TaskModel {
  constructor() {
    this.tasks = [];
  }

  addTask(task) {
    this.tasks.push(task);
    return this.tasks;
  }

  removeTask(index) {
    this.tasks.splice(index, 1);
    return this.tasks;
  }

  getTasks() {
    return this.tasks;
  }
}

// ViewModel
export default {
  name: 'App',
  data() {
    // ViewModelの状態
    return {
      taskModel: new TaskModel(),
      tasks: [],
      newTask: ''
    };
  },
  computed: {
    // ViewModelの算出プロパティ
    taskCount() {
      return this.tasks.length;
    }
  },
  methods: {
    // ViewModelのアクション
    addTask() {
      if (this.newTask.trim()) {
        this.tasks = this.taskModel.addTask(this.newTask);
        this.newTask = '';
      }
    },
    removeTask(index) {
      this.tasks = this.taskModel.removeTask(index);
    }
  }
};
</script>
```

## MVCとMVVMの主要な違い

### 1. データフローの違い

| 特徴 | MVC | MVVM |
|------|-----|------|
| データの流れ | 一方向または双方向（実装による） | 双方向データバインディング |
| 更新の仕組み | Controllerが明示的に更新 | データバインディングで自動更新 |
| 結合度 | ViewとControllerが密結合 | ViewとViewModelが疎結合 |

### 2. 責任の違い

**MVC:**
- Controllerがユーザー入力を処理
- ControllerがViewとModelの橋渡し
- Viewの更新をControllerが制御（または手動通知）

**MVVM:**
- ViewModelがViewのための状態を管理
- データバインディングにより自動同期
- ViewModelはViewの存在を知らない

### 3. テストのしやすさ

**MVC:**
- ControllerとViewが密結合のため、テストが複雑
- Viewをモック化する必要がある場合が多い

**MVVM:**
- ViewModelは独立しているため、単体テストが容易
- Viewのロジックがないため、UI層のテストが簡潔

### 4. 適用場面

**MVCが適している場合:**
- サーバーサイドのWebアプリケーション（Ruby on Rails、Django等）
- シンプルなWebアプリケーション
- 従来のWebフレームワーク

**MVVMが適している場合:**
- リッチなクライアントサイドアプリケーション
- データバインディングをサポートするフレームワーク（Vue.js、Angular、React with hooks等）
- デスクトップアプリケーション（WPF、Xamarin等）
- モバイルアプリケーション（SwiftUI、Jetpack Compose等）

## 実践的な比較例

### シナリオ: ユーザー入力でデータを更新する

#### MVCの場合

```javascript
// Controllerが明示的に処理
class UserController {
  constructor(model, view) {
    this.model = model;
    this.view = view;
    this.bindEvents();
  }

  bindEvents() {
    this.view.onNameChange((name) => {
      this.model.setName(name);
      this.view.updateDisplay(this.model.getName());
    });
  }
}
```

#### MVVMの場合

```vue
<!-- データバインディングで自動処理 -->
<template>
  <div>
    <input v-model="userName" />
    <p>Hello, {{ userName }}!</p>
  </div>
</template>

<script>
export default {
  data() {
    return {
      userName: ''
    };
  }
};
</script>
```

## まとめ

### MVCの特徴
- ✅ シンプルで理解しやすい
- ✅ サーバーサイドに適している
- ⚠️ ViewとControllerが密結合になりがち
- ⚠️ 手動でのView更新が必要

### MVVMの特徴
- ✅ データバインディングによる自動同期
- ✅ ViewModelの単体テストが容易
- ✅ Viewとビジネスロジックの完全な分離
- ⚠️ 学習コストがやや高い
- ⚠️ 小規模アプリには過剰な場合も

どちらのアーキテクチャも、それぞれの強みと適用場面があります。プロジェクトの要件、チームの経験、使用するフレームワークなどを考慮して、適切なパターンを選択することが重要です。

現代のフロントエンド開発では、データバインディングの利便性からMVVMやその派生パターンが主流となっていますが、サーバーサイドやシンプルなアプリケーションではMVCも依然として有効な選択肢です。

## 参考資料

- [Model-View-Controller - Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93controller)
- [Model-View-ViewModel - Wikipedia](https://en.wikipedia.org/wiki/Model%E2%80%93view%E2%80%93viewmodel)
- [Vue.js Documentation](https://vuejs.org/)
- [Introduction to the MVVM Pattern - Microsoft](https://learn.microsoft.com/en-us/xamarin/xamarin-forms/enterprise-application-patterns/mvvm)
