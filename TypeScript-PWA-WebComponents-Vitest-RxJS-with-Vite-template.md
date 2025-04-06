# Vanilla TypeScript + Web Components + PWA + Vitest + RxJS を Vite で構築する手順

以下コマンドにて、タイトル条件の開発環境を構築します。
```sh
npm install
```

> [!CAUTION]
> 以下は、このテンプレートの構築時の手順ですので、基本的には利用しません。

## 1. プロジェクトの初期化

```bash
# プロジェクトの作成
npm create vite@latest my-web-components-rxjs -- --template vanilla-ts
cd my-web-components-rxjs

# 必要なパッケージのインストール
npm install
```

## 2. 必要なパッケージのインストール

```bash
# PWA サポート
npm install -D vite-plugin-pwa

# RxJS
npm install rxjs

# Vitest と関連パッケージ
npm install -D vitest happy-dom @testing-library/dom
```

## 3. Vite の設定

`vite.config.ts` ファイルを作成します:

```typescript
import { defineConfig } from 'vite'
import { VitePWA } from 'vite-plugin-pwa'

export default defineConfig({
  plugins: [
    VitePWA({
      registerType: 'autoUpdate',
      includeAssets: ['favicon.svg', 'favicon.ico', 'robots.txt', 'apple-touch-icon.png'],
      manifest: {
        name: 'RxJS Web Components PWA',
        short_name: 'RxPWA',
        description: 'Web Components with RxJS Progressive Web App',
        theme_color: '#ffffff',
        icons: [
          {
            src: 'pwa-192x192.png',
            sizes: '192x192',
            type: 'image/png'
          },
          {
            src: 'pwa-512x512.png',
            sizes: '512x512',
            type: 'image/png'
          }
        ]
      }
    })
  ]
})
```

## 4. Vitest の設定

`vitest.config.ts` ファイルを作成します:

```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    environment: 'happy-dom',
    globals: true,
    include: ['**/*.{test,spec}.{ts,tsx}']
  }
})
```

## 5. package.json のスクリプト更新

package.json の scripts セクションを次のように更新します:

```json
"scripts": {
  "dev": "vite",
  "build": "tsc && vite build",
  "preview": "vite preview",
  "test": "vitest",
  "test:ui": "vitest --ui",
  "test:run": "vitest run",
  "test:coverage": "vitest run --coverage"
}
```

## 6. ベースとなる Web Components クラスの作成

`src/components/base/reactive-element.ts` ファイルを作成します:

```typescript
import { BehaviorSubject, Observable } from 'rxjs';

/**
 * RxJSを統合したベースWebComponentクラス
 */
export class ReactiveElement extends HTMLElement {
  private _state: Map<string, BehaviorSubject<any>> = new Map();
  
  /**
   * 監視する属性の配列
   */
  static get observedAttributes(): string[] {
    return [];
  }
  
  /**
   * コンポーネントがDOMに追加された時
   */
  connectedCallback(): void {
    this.render();
  }
  
  /**
   * コンポーネントがDOMから削除された時
   */
  disconnectedCallback(): void {
    // リソースの解放処理があればここで行う
  }
  
  /**
   * 属性が変更された時
   */
  attributeChangedCallback(name: string, oldValue: string, newValue: string): void {
    // 属性変更がステートにも影響する場合は、ステートを更新
    if (this._state.has(name)) {
      this.setState(name, newValue);
    } else {
      this.render();
    }
  }
  
  /**
   * ステートの初期化
   */
  protected initState<T>(key: string, initialValue: T): void {
    if (!this._state.has(key)) {
      this._state.set(key, new BehaviorSubject<T>(initialValue));
    }
  }
  
  /**
   * ステートの取得
   */
  protected getState<T>(key: string): T | undefined {
    return this._state.get(key)?.getValue();
  }
  
  /**
   * ステートの更新
   */
  protected setState<T>(key: string, value: T): void {
    if (this._state.has(key)) {
      this._state.get(key)?.next(value);
      this.render();
    }
  }
  
  /**
   * ステートの監視
   */
  protected observe<T>(key: string): Observable<T> {
    return this._state.get(key)?.asObservable() as Observable<T>;
  }
  
  /**
   * コンポーネントのレンダリング処理
   */
  protected render(): void {
    // 継承先でオーバーライド
  }
}
```

## 7. カウンターコンポーネントの実装

`src/components/counter/counter-element.ts` ファイルを作成します:

```typescript
// counter-element.ts の修正版
import { fromEvent } from 'rxjs';
import { ReactiveElement } from '../base/reactive-element';

export class CounterElement extends ReactiveElement {
  static get observedAttributes(): string[] {
    return ['initial'];
  }
  
  constructor() {
    super();
    // Shadow DOMの初期化
    this.attachShadow({ mode: 'open' });
    
    // ステートの初期化
    this.initState('count', 0);
  }
  
  attributeChangedCallback(name: string, oldValue: string, newValue: string): void {
    console.log(`name: ${name}, oldValue: ${oldValue}, newValue: ${newValue}`);
    if (name === 'initial' && newValue !== null) {
      // 初期値の属性が変更された場合、カウント値を更新
      this.setState('count', Number(newValue));
    }
    super.attributeChangedCallback(name, oldValue, newValue);
  }
  
  connectedCallback(): void {
    // 初期値の属性があれば取得
    const initialValue = this.getAttribute('initial');
    if (initialValue !== null) {
      this.setState('count', Number(initialValue));
    }
    
    super.connectedCallback();
    this.setupEventListeners();
  }
  
  /**
   * イベントリスナーの設定 - 再設定の問題を解決
   */
  private setupEventListeners(): void {
    // 古いイベントリスナーをクリア
    const incrementBtn = this.shadowRoot?.querySelector('.increment');
    const decrementBtn = this.shadowRoot?.querySelector('.decrement');
    const resetBtn = this.shadowRoot?.querySelector('.reset');
    
    // 一度だけイベントリスナーを設定するための属性を確認
    if (!this.hasAttribute('listeners-initialized')) {
      if (incrementBtn) {
        fromEvent(incrementBtn, 'click').subscribe(() => {
          const currentCount = this.getState<number>('count') || 0;
          this.setState('count', currentCount + 1);
        });
      }
      
      if (decrementBtn) {
        fromEvent(decrementBtn, 'click').subscribe(() => {
          const currentCount = this.getState<number>('count') || 0;
          this.setState('count', currentCount - 1);
        });
      }
      
      if (resetBtn) {
        fromEvent(resetBtn, 'click').subscribe(() => {
          this.setState('count', 0);
        });
      }
      
      // リスナー初期化済みのフラグを設定
      this.setAttribute('listeners-initialized', 'true');
    }
  }
  
  /**
   * ビューのレンダリング
   */
  protected render(): void {
    if (!this.shadowRoot) return;
    
    const count = this.getState<number>('count') || 0;
    
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: 'Arial', sans-serif;
          color: #333;
          margin: 20px;
          padding: 15px;
          border-radius: 8px;
          box-shadow: 0 2px 5px rgba(0,0,0,0.1);
          background-color: #f8f9fa;
        }
        
        .counter {
          text-align: center;
        }
        
        .value {
          font-size: 2rem;
          font-weight: bold;
          margin: 10px 0;
          color: ${count > 0 ? 'green' : count < 0 ? 'red' : 'black'};
        }
        
        .controls {
          display: flex;
          justify-content: center;
          gap: 10px;
          margin-top: 10px;
        }
        
        button {
          padding: 8px 16px;
          border: none;
          border-radius: 4px;
          cursor: pointer;
          font-weight: bold;
          transition: background-color 0.3s;
        }
        
        .increment {
          background-color: #28a745;
          color: white;
        }
        
        .decrement {
          background-color: #dc3545;
          color: white;
        }
        
        .reset {
          background-color: #6c757d;
          color: white;
        }
        
        button:hover {
          opacity: 0.9;
        }
      </style>
      
      <div class="counter">
        <h2>RxJS Counter</h2>
        <div class="value">${count}</div>
        <div class="controls">
          <button class="decrement">-</button>
          <button class="reset">Reset</button>
          <button class="increment">+</button>
        </div>
      </div>
    `;
    
    // イベントリスナーを再設定
    this.setupEventListeners();
  }
}

// カスタム要素の登録
customElements.define('rx-counter', CounterElement);
```

## 8. TodoListコンポーネントの実装

`src/components/todo/todo-element.ts` ファイルを作成します:

```typescript
import { fromEvent, merge } from 'rxjs';
import { ReactiveElement } from '../base/reactive-element';

interface TodoItem {
  id: number;
  text: string;
  completed: boolean;
}

export class TodoElement extends ReactiveElement {
  constructor() {
    super();
    // Shadow DOMの初期化
    this.attachShadow({ mode: 'open' });
    
    // ステートの初期化
    this.initState<TodoItem[]>('todos', []);
    this.initState<string>('newTodo', '');
  }
  
  connectedCallback(): void {
    super.connectedCallback();
  }
  
  /**
   * イベントリスナーの設定
   */
  private setupEventListeners(): void {
    const input = this.shadowRoot?.querySelector<HTMLInputElement>('.todo-input');
    const addButton = this.shadowRoot?.querySelector('.add-todo');
    
    if (input) {
      // 入力フィールドの変更を監視
      fromEvent(input, 'input').subscribe((event) => {
        const value = (event.target as HTMLInputElement).value;
        this.setState('newTodo', value);
      });
      
      // Enterキーでの追加
      fromEvent(input, 'keydown').subscribe((event: Event) => {
        const keyEvent = event as KeyboardEvent;
        if (keyEvent.key === 'Enter') {
          this.addTodo();
        }
      });
    }
    
    if (addButton) {
      // ボタンクリックでの追加
      fromEvent(addButton, 'click').subscribe(() => {
        this.addTodo();
      });
    }
    
    // 完了状態の切り替え
    const checkboxes = this.shadowRoot?.querySelectorAll<HTMLInputElement>('input[type="checkbox"]');
    if (checkboxes) {
      checkboxes.forEach(checkbox => {
        fromEvent(checkbox, 'change').subscribe(() => {
          const todoId = Number(checkbox.dataset.id);
          this.toggleTodoComplete(todoId);
        });
      });
    }
    
    // 削除ボタン
    const deleteButtons = this.shadowRoot?.querySelectorAll('.delete-todo');
    if (deleteButtons) {
      deleteButtons.forEach(button => {
        fromEvent(button, 'click').subscribe((event: Event) => {
          const todoId = Number((event.target as HTMLElement).dataset.id);
          this.deleteTodo(todoId);
        });
      });
    }
  }
  
  /**
   * 新しいTodoの追加
   */
  private addTodo(): void {
    const newTodoText = this.getState<string>('newTodo')?.trim();
    if (!newTodoText) return;
    
    const todos = [...(this.getState<TodoItem[]>('todos') || [])];
    const newTodo: TodoItem = {
      id: Date.now(),
      text: newTodoText,
      completed: false
    };
    
    this.setState('todos', [...todos, newTodo]);
    this.setState('newTodo', '');
  }
  
  /**
   * Todoの完了状態切り替え
   */
  private toggleTodoComplete(id: number): void {
    const todos = [...(this.getState<TodoItem[]>('todos') || [])];
    const updatedTodos = todos.map(todo => 
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    );
    
    this.setState('todos', updatedTodos);
    
    // カスタムイベントの発行
    this.dispatchEvent(new CustomEvent('todo-status-changed', {
      bubbles: true,
      detail: { id, completed: updatedTodos.find(t => t.id === id)?.completed }
    }));
  }
  
  /**
   * Todoの削除
   */
  private deleteTodo(id: number): void {
    const todos = [...(this.getState<TodoItem[]>('todos') || [])];
    const updatedTodos = todos.filter(todo => todo.id !== id);
    
    this.setState('todos', updatedTodos);
    
    // カスタムイベントの発行
    this.dispatchEvent(new CustomEvent('todo-deleted', {
      bubbles: true,
      detail: { id }
    }));
  }
  
  /**
   * ビューのレンダリング
   */
  protected render(): void {
    if (!this.shadowRoot) return;
    
    const todos = this.getState<TodoItem[]>('todos') || [];
    const newTodo = this.getState<string>('newTodo') || '';
    
    this.shadowRoot.innerHTML = `
      <style>
        :host {
          display: block;
          font-family: 'Arial', sans-serif;
          color: #333;
          margin: 20px;
          padding: 20px;
          border-radius: 8px;
          box-shadow: 0 2px 8px rgba(0,0,0,0.1);
          background-color: #ffffff;
        }
        
        .todo-app {
          max-width: 500px;
          margin: 0 auto;
        }
        
        h2 {
          text-align: center;
          color: #2c3e50;
        }
        
        .todo-input-container {
          display: flex;
          margin-bottom: 20px;
        }
        
        .todo-input {
          flex: 1;
          padding: 10px;
          border: 1px solid #ddd;
          border-radius: 4px 0 0 4px;
          font-size: 16px;
        }
        
        .add-todo {
          padding: 10px 15px;
          background-color: #3498db;
          color: white;
          border: none;
          border-radius: 0 4px 4px 0;
          cursor: pointer;
          font-weight: bold;
        }
        
        .todo-list {
          list-style-type: none;
          padding: 0;
        }
        
        .todo-item {
          display: flex;
          align-items: center;
          padding: 10px;
          border-bottom: 1px solid #eee;
        }
        
        .todo-item:last-child {
          border-bottom: none;
        }
        
        .todo-checkbox {
          margin-right: 10px;
        }
        
        .todo-text {
          flex: 1;
          font-size: 16px;
        }
        
        .completed {
          text-decoration: line-through;
          color: #7f8c8d;
        }
        
        .delete-todo {
          padding: 5px 10px;
          background-color: #e74c3c;
          color: white;
          border: none;
          border-radius: 4px;
          cursor: pointer;
        }
        
        .empty-state {
          text-align: center;
          padding: 20px;
          color: #7f8c8d;
        }
      </style>
      
      <div class="todo-app">
        <h2>RxJS Todo List</h2>
        
        <div class="todo-input-container">
          <input 
            type="text" 
            class="todo-input" 
            placeholder="Add a new task..." 
            value="${newTodo}"
          />
          <button class="add-todo">Add</button>
        </div>
        
        <ul class="todo-list">
          ${todos.length > 0 ? 
            todos.map(todo => `
              <li class="todo-item">
                <input 
                  type="checkbox" 
                  class="todo-checkbox" 
                  data-id="${todo.id}" 
                  ${todo.completed ? 'checked' : ''}
                />
                <span class="todo-text ${todo.completed ? 'completed' : ''}">
                  ${todo.text}
                </span>
                <button class="delete-todo" data-id="${todo.id}">Delete</button>
              </li>
            `).join('') : 
            '<div class="empty-state">No tasks yet. Add some!</div>'
          }
        </ul>
      </div>
    `;
    
    // イベントリスナーを再設定
    this.setupEventListeners();
  }
}

// カスタム要素の登録
customElements.define('rx-todo', TodoElement);
```

## 9. テスト実装

### カウンターコンポーネントのテスト

`src/components/counter/counter-element.test.ts` ファイルを作成します:

```typescript
import { describe, it, expect, beforeEach, afterEach } from 'vitest';
import './counter-element';

describe('CounterElement', () => {
  let element: HTMLElement;
  
  beforeEach(() => {
    element = document.createElement('rx-counter');
    document.body.appendChild(element);
  });
  
  afterEach(() => {
    document.body.removeChild(element);
  });
  
  it('renders with initial count of 0', () => {
    const shadowRoot = (element as any).shadowRoot;
    const valueEl = shadowRoot.querySelector('.value');
    expect(valueEl.textContent).toBe('0');
  });
  
  it('increments the counter when clicking the increment button', async () => {
    const shadowRoot = (element as any).shadowRoot;
    const incrementBtn = shadowRoot.querySelector('.increment');
    
    incrementBtn.click();
    
    // レンダリングが非同期の場合があるので少し待機
    await new Promise(resolve => setTimeout(resolve, 0));
    
    const valueEl = shadowRoot.querySelector('.value');
    expect(valueEl.textContent).toBe('1');
  });
  
  it('decrements the counter when clicking the decrement button', async () => {
    const shadowRoot = (element as any).shadowRoot;
    const decrementBtn = shadowRoot.querySelector('.decrement');
    
    decrementBtn.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    const valueEl = shadowRoot.querySelector('.value');
    expect(valueEl.textContent).toBe('-1');
  });
  
  it('resets the counter when clicking the reset button', async () => {
    const shadowRoot = (element as any).shadowRoot;
    const incrementBtn = shadowRoot.querySelector('.increment');
    const resetBtn = shadowRoot.querySelector('.reset');
    
    // まず値を変更
    incrementBtn.click();
    incrementBtn.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    // リセット
    resetBtn.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    const valueEl = shadowRoot.querySelector('.value');
    expect(valueEl.textContent).toBe('0');
  });
  
  it('renders with initial value from attribute', async () => {
    document.body.removeChild(element);
    
    // 初期値を指定
    element = document.createElement('rx-counter');
    element.setAttribute('initial', '10');
    document.body.appendChild(element);
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    const shadowRoot = (element as any).shadowRoot;
    const valueEl = shadowRoot.querySelector('.value');
    expect(valueEl.textContent).toBe('10');
  });
});
```

### TodoListコンポーネントのテスト

`src/components/todo/todo-element.test.ts` ファイルを作成します:

```typescript
import { describe, it, expect, beforeEach, afterEach, vi } from 'vitest';
import './todo-element';

describe('TodoElement', () => {
  let element: HTMLElement;
  
  beforeEach(() => {
    element = document.createElement('rx-todo');
    document.body.appendChild(element);
  });
  
  afterEach(() => {
    document.body.removeChild(element);
  });
  
  it('renders empty state initially', () => {
    const shadowRoot = (element as any).shadowRoot;
    const emptyState = shadowRoot.querySelector('.empty-state');
    expect(emptyState).not.toBeNull();
    expect(emptyState.textContent).toContain('No tasks yet');
  });
  
  it('adds a new todo when input and button are used', async () => {
    const shadowRoot = (element as any).shadowRoot;
    const input = shadowRoot.querySelector('.todo-input');
    const addButton = shadowRoot.querySelector('.add-todo');
    
    // 入力フィールドに値を設定
    input.value = 'Test Todo';
    input.dispatchEvent(new Event('input'));
    
    // 追加ボタンをクリック
    addButton.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    const todoItems = shadowRoot.querySelectorAll('.todo-item');
    expect(todoItems.length).toBe(1);
    
    const todoText = shadowRoot.querySelector('.todo-text');
    expect(todoText.textContent.trim()).toBe('Test Todo');
  });
  
  it('marks a todo as completed when checkbox is clicked', async () => {
    const shadowRoot = (element as any).shadowRoot;
    const input = shadowRoot.querySelector('.todo-input');
    const addButton = shadowRoot.querySelector('.add-todo');
    
    // Todo追加
    input.value = 'Complete me';
    input.dispatchEvent(new Event('input'));
    addButton.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    // チェックボックスをクリック
    const checkbox = shadowRoot.querySelector('.todo-checkbox');
    checkbox.checked = true;
    checkbox.dispatchEvent(new Event('change'));
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    const todoText = shadowRoot.querySelector('.todo-text');
    expect(todoText.classList.contains('completed')).toBe(true);
  });
  
  it('deletes a todo when delete button is clicked', async () => {
    const shadowRoot = (element as any).shadowRoot;
    const input = shadowRoot.querySelector('.todo-input');
    const addButton = shadowRoot.querySelector('.add-todo');
    
    // Todo追加
    input.value = 'Delete me';
    input.dispatchEvent(new Event('input'));
    addButton.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    // 削除前の確認
    let todoItems = shadowRoot.querySelectorAll('.todo-item');
    expect(todoItems.length).toBe(1);
    
    // 削除ボタンをクリック
    const deleteButton = shadowRoot.querySelector('.delete-todo');
    deleteButton.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    // 削除後の確認
    todoItems = shadowRoot.querySelectorAll('.todo-item');
    expect(todoItems.length).toBe(0);
    
    const emptyState = shadowRoot.querySelector('.empty-state');
    expect(emptyState).not.toBeNull();
  });
  
  it('dispatches custom event when todo status changes', async () => {
    const eventSpy = vi.fn();
    element.addEventListener('todo-status-changed', eventSpy);
    
    const shadowRoot = (element as any).shadowRoot;
    const input = shadowRoot.querySelector('.todo-input');
    const addButton = shadowRoot.querySelector('.add-todo');
    
    // Todo追加
    input.value = 'Event Test';
    input.dispatchEvent(new Event('input'));
    addButton.click();
    
    await new Promise(resolve => setTimeout(resolve, 0));
    
    // チェックボックスをクリック
    const checkbox = shadowRoot.querySelector('.todo-checkbox');
    checkbox.checked = true;
    checkbox.dispatchEvent(new Event('change'));
    
    expect(eventSpy).toHaveBeenCalled();
    expect(eventSpy.mock.calls[0][0].detail.completed).toBe(true);
  });
});
```

## 10. メインアプリケーション実装

`src/main.ts` ファイルを編集します:

```typescript
import './style.css';
import './components/counter/counter-element';
import './components/todo/todo-element';

document.querySelector<HTMLDivElement>('#app')!.innerHTML = `
  <div class="container">
    <h1>RxJS Web Components PWA</h1>
    
    <div class="component-container">
      <rx-counter initial="5"></rx-counter>
    </div>
    
    <div class="component-container">
      <rx-todo></rx-todo>
    </div>
  </div>
`;

// Todo関連のイベントをリッスン
document.addEventListener('todo-status-changed', (event: Event) => {
  const customEvent = event as CustomEvent;
  console.log('Todo status changed:', customEvent.detail);
});

document.addEventListener('todo-deleted', (event: Event) => {
  const customEvent = event as CustomEvent;
  console.log('Todo deleted:', customEvent.detail);
});

// PWA登録
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () => {
    console.log('Service Worker and PWA support detected');
  });
}
```

## 11. スタイルシートの編集

`src/style.css` ファイルを編集します:

```css
:root {
  font-family: Inter, system-ui, Avenir, Helvetica, Arial, sans-serif;
  line-height: 1.5;
  font-weight: 400;

  color-scheme: light dark;
  color: rgba(255, 255, 255, 0.87);
  background-color: #242424;

  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}

* {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}

body {
  margin: 0;
  display: flex;
  place-items: center;
  min-width: 320px;
  min-height: 100vh;
}

.container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 2rem;
}

h1 {
  font-size: 2.5rem;
  text-align: center;
  margin-bottom: 2rem;
  color: #3498db;
}

.component-container {
  margin-bottom: 2rem;
}

@media (prefers-color-scheme: light) {
  :root {
    color: #213547;
    background-color: #ffffff;
  }
}
```

## 12. PWA用のアセット追加

`public` ディレクトリに以下のファイルを追加します:

- favicon.ico
- pwa-192x192.png
- pwa-512x512.png
- robots.txt

## 13. インデックスHTMLファイルの更新

`index.html` ファイルを編集:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="Web Components with RxJS PWA Sample Application">
    <meta name="theme-color" content="#3498db">
    <title>RxJS Web Components PWA</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

## 14. プロジェクトの実行とビルド

```bash
# 開発サーバーの起動
npm run dev

# テストの実行
npm test

# ビルド
npm run build

# ビルド結果のプレビュー
npm run preview
```

## 15. 最終的なディレクトリ構造

```
my-web-components-rxjs/
├── public/
│   ├── favicon.ico
│   ├── pwa-192x192.png
│   ├── pwa-512x512.png
│   └── robots.txt
├── src/
│   ├── components/
│   │   ├── base/
│   │   │   └── reactive-element.ts
│   │   ├── counter/
│   │   │   ├── counter-element.ts
│   │   │   └── counter-element.test.ts
│   │   └── todo/
│   │       ├── todo-element.ts
│   │       └── todo-element.test.ts
│   ├── main.ts
│   └── style.css
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
└── vitest.config.ts
```
