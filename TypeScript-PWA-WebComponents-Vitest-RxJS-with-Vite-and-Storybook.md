# Vanilla TypeScript + Web Components + PWA + Vitest + RxJS + Storybook 構築ガイド

このガイドでは、モダンな Web フロントエンド開発環境を一から構築する手順を詳しく説明します。

## プロジェクト環境

- **Vanilla TypeScript**: 型安全な JavaScript 開発
- **Web Components**: 再利用可能なネイティブ UI コンポーネント
- **PWA**: Progressive Web App 対応
- **Vitest**: 高速なテストフレームワーク
- **RxJS**: リアクティブプログラミング
- **Storybook**: UI コンポーネント開発とドキュメント化
- **Vite**: 高速な開発サーバーとビルドツール

## 1. プロジェクトの初期化

```bash
# プロジェクトの作成
npm create vite@latest my-web-components-app -- --template vanilla-ts
cd my-web-components-app

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
        name: 'Web Components PWA App',
        short_name: 'WC-PWA',
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

## 5. package.json のスクリプト設定

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

## 6. ベース Web Component クラスの作成

ディレクトリ構造を作成します:

```bash
mkdir -p src/components/base
```

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

```bash
mkdir -p src/components/counter
```

`src/components/counter/counter-element.ts` ファイルを作成します:

```typescript
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
   * イベントリスナーの設定
   */
  private setupEventListeners(): void {
    // 既にイベントリスナーがセットアップされている場合は処理をスキップ
    if (this.hasAttribute('listeners-initialized')) {
      return;
    }
    
    const incrementBtn = this.shadowRoot?.querySelector('.increment');
    const decrementBtn = this.shadowRoot?.querySelector('.decrement');
    const resetBtn = this.shadowRoot?.querySelector('.reset');
    
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
    
    // イベントリスナー初期化済みフラグをセット
    this.setAttribute('listeners-initialized', 'true');
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

## 8. ToDo リストコンポーネントの実装

```bash
mkdir -p src/components/todo
```

`src/components/todo/todo-element.ts` ファイルを作成します:

```typescript
import { fromEvent } from 'rxjs';
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
    this.setupEventListeners();
  }
  
  /**
   * イベントリスナーの設定
   */
  private setupEventListeners(): void {
    // 完了状態の切り替えと削除ボタンのイベントリスナーを毎回設定し直す必要がある
    // これらは動的に追加される要素なので
    
    const input = this.shadowRoot?.querySelector<HTMLInputElement>('.todo-input');
    const addButton = this.shadowRoot?.querySelector('.add-todo');
    
    // 入力フィールドと追加ボタンは一度だけリスナーを設定
    if (!this.hasAttribute('listeners-initialized')) {
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
      
      // 静的なリスナーの初期化フラグをセット
      this.setAttribute('listeners-initialized', 'true');
    }
    
    // 以下の要素は動的に変わるため、毎回設定が必要
    
    // 完了状態の切り替え
    const checkboxes = this.shadowRoot?.querySelectorAll<HTMLInputElement>('input[type="checkbox"]');
    if (checkboxes) {
      checkboxes.forEach(checkbox => {
        // 既存のイベントリスナーを削除
        checkbox.removeEventListener('change', () => {});
        
        // 新しいイベントリスナーを追加
        checkbox.addEventListener('change', () => {
          const todoId = Number(checkbox.dataset.id);
          this.toggleTodoComplete(todoId);
        });
      });
    }
    
    // 削除ボタン
    const deleteButtons = this.shadowRoot?.querySelectorAll('.delete-todo');
    if (deleteButtons) {
      deleteButtons.forEach(button => {
        // 既存のイベントリスナーを削除
        button.removeEventListener('click', () => {});
        
        // 新しいイベントリスナーを追加
        button.addEventListener('click', (event: Event) => {
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

## 9. テストの実装

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

### ToDo リストコンポーネントのテスト

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

## 10. メインアプリケーションファイルの編集

`src/main.ts` ファイルを編集します:

```typescript
import './style.css';
import './components/counter/counter-element';
import './components/todo/todo-element';

document.querySelector<HTMLDivElement>('#app')!.innerHTML = `
  <div class="container">
    <h1>Web Components PWA</h1>
    
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
  width: 100%;
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
- favicon.svg
- pwa-192x192.png
- pwa-512x512.png
- robots.txt
- apple-touch-icon.png

簡単な `robots.txt` の例:

```
User-agent: *
Allow: /
```

## 13. インデックスHTMLファイルの更新

`index.html` ファイルを編集します:

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <link rel="icon" type="image/svg+xml" href="/favicon.svg" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <meta name="description" content="Web Components with RxJS PWA Sample Application">
    <meta name="theme-color" content="#3498db">
    <title>Web Components PWA</title>
  </head>
  <body>
    <div id="app"></div>
    <script type="module" src="/src/main.ts"></script>
  </body>
</html>
```

## 14. Storybook のインストールと設定

### 14.1 Storybook のインストール

```bash
# Storybook のインストール
npx storybook@latest init
```

このコマンドにより、Storybook に必要なパッケージがインストールされ、基本的な設定ファイルが生成されます。

### 14.2 Storybook の設定ファイルを編集

`.storybook/main.ts` ファイルを編集します:

```typescript
import type { StorybookConfig } from '@storybook/web-components-vite';

const config: StorybookConfig = {
  "stories": [
    "../src/**/*.mdx",
    "../src/**/*.stories.@(js|jsx|mjs|ts|tsx)"
  ],
  "addons": [
    "@storybook/addon-essentials",
    "@storybook/addon-links",
    "@storybook/addon-interactions",
    "@chromatic-com/storybook",
    "@storybook/experimental-addon-test"
  ],
  "framework": {
    "name": "@storybook/web-components-vite",
    "options": {}
  },
  "docs": {
    "autodocs": "true"  // すべてのストーリーに自動的にドキュメントを生成
  }
};
export default config;
```

### 14.3 プレビュー設定ファイルの作成

`.storybook/preview.ts` ファイルを編集します:

```typescript
import type { Preview } from "@storybook/web-components";

// グローバルスタイルのインポート
import '../src/style.css';

// Web Components の登録を確実に行うためにインポート
import '../src/components/counter/counter-element';
import '../src/components/todo/todo-element';

const preview: Preview = {
  parameters: {
    actions: { argTypesRegex: "^on[A-Z].*" },
    controls: {
      matchers: {
        color: /(background|color)$/i,
        date: /Date$/,
      },
    },
    // モバイルレスポンシブデザインのためのビューポート
    viewport: {
      viewports: {
        mobile: {
          name: 'Mobile',
          styles: {
            width: '375px',
            height: '667px',
          },
        },
        tablet: {
          name: 'Tablet',
          styles: {
            width: '768px',
            height: '1024px',
          },
        },
        desktop: {
          name: 'Desktop',
          styles: {
            width: '1200px',
            height: '800px',
          },
        },
      },
    },
  },
};

export default preview;
```

### 14.4 カスタムテーマの設定 (オプション)

`.storybook/manager.ts` ファイルを作成します:

```typescript
import { addons } from '@storybook/manager-api';
import { create } from '@storybook/theming/create';

const theme = create({
  base: 'light',
  brandTitle: 'Web Components PWA',
  brandUrl: 'https://example.com',
  brandTarget: '_self',
  colorPrimary: '#3498db',
  colorSecondary: '#2ecc71',
});

addons.setConfig({
  theme,
});
```

### 14.5 package.json にスクリプトを追加

```json
"scripts": {
  "storybook": "storybook dev -p 6006",
  "build-storybook": "storybook build"
}
```

## 15. コンポーネントのストーリー作成

### 15.1 カウンターコンポーネントのストーリー

`src/components/counter/counter-element.stories.ts` ファイルを作成します:

```typescript
import { StoryObj, Meta } from '@storybook/web-components';
import './counter-element';

const meta: Meta = {
  title: 'Components/Counter',
  component: 'rx-counter',
  render: (args) => {
    const element = document.createElement('rx-counter');
    if (args.initial !== undefined) {
      element.setAttribute('initial', args.initial.toString());
    }
    return element;
  },
  argTypes: {
    initial: {
      control: { type: 'number' },
      description: '初期カウント値',
      defaultValue: 0,
    },
  },
};

export default meta;

type Story = StoryObj;

export const Default: Story = {
  args: {
    initial: 0,
  },
};

export const StartWithPositiveValue: Story = {
  args: {
    initial: 10,
  },
  name: '正の値からスタート',
};

export const StartWithNegativeValue: Story = {
  args: {
    initial: -5,
  },
  name: '負の値からスタート',
};

export const NoInitialValue: Story = {
  name: '初期値なし',
};
```

### 15.2 ToDo リストコンポーネントのストーリー

`src/components/todo/todo-element.stories.ts` ファイルを作成します:

```typescript
import { StoryObj, Meta } from '@storybook/web-components';
import './todo-element';

const meta: Meta = {
  title: 'Components/Todo',
  component: 'rx-todo',
  render: () => {
    const element = document.createElement('rx-todo');
    return element;
  },
};

export default meta;

type Story = StoryObj;

export const Default: Story = {
  name: '空のTodoリスト',
};

// Todo項目を事前に入力した状態を作成するカスタムレンダラー
export const WithPrefilledTodos: Story = {
  name: 'タスク入力済み',
  render: () => {
    const element = document.createElement('rx-todo');
    
    // コンポーネントが接続された後にタスクを追加
    setTimeout(() => {
      // Shadow DOM内の要素にアクセス
      const shadowRoot = element.shadowRoot;
      const input = shadowRoot?.querySelector('.todo-input') as HTMLInputElement;
      const addButton = shadowRoot?.querySelector('.add-todo') as HTMLButtonElement;
      
      // サンプルTodoを追加
      const todos = ['Storybook の学習', 'Web Components の実装', 'RxJS でリアクティブ処理'];
      
      todos.forEach(todo => {
        input.value = todo;
        input.dispatchEvent(new Event('input'));
        addButton.click();
      });
      
      // 1つ目のタスクを完了状態に
      const checkbox = shadowRoot?.querySelector('.todo-checkbox') as HTMLInputElement;
      if (checkbox) {
        checkbox.checked = true;
        checkbox.dispatchEvent(new Event('change'));
      }
    }, 100);
    
    return element;
  }
};

// アクションを記録するストーリー
export const WithActions: Story = {
  name: 'イベント検証用',
  render: () => {
    const element = document.createElement('rx-todo');
    
    // カスタムイベントをリッスン
    element.addEventListener('todo-status-changed', (e) => {
      const event = new CustomEvent('storybook-action', {
        bubbles: true,
        detail: {
          action: 'todo-status-changed',
          data: (e as CustomEvent).detail
        }
      });
      element.dispatchEvent(event);
    });
    
    element.addEventListener('todo-deleted', (e) => {
      const event = new CustomEvent('storybook-action', {
        bubbles: true,
        detail: {
          action: 'todo-deleted',
          data: (e as CustomEvent).detail
        }
      });
      element.dispatchEvent(event);
    });
    
    return element;
  },
  parameters: {
    actions: {
      handles: ['storybook-action'],
    },
  },
};
```

## 16. ビルドと実行

### 16.1 開発サーバーの起動

```bash
# 通常の開発サーバー
npm run dev

# または Storybook
npm run storybook
```

### 16.2 テストの実行

```bash
# すべてのテストを実行
npm test

# または1回だけ実行
npm run test:run
```

### 16.3 本番ビルド

```bash
# アプリケーションのビルド
npm run build

# または Storybook のビルド
npm run build-storybook
```

### 16.4 ビルド結果のプレビュー

```bash
npm run preview
```

## 17. 最終的なディレクトリ構造

```
my-web-components-app/
├── .storybook/
│   ├── main.ts
│   ├── preview.ts
│   └── manager.ts
├── public/
│   ├── favicon.ico
│   ├── favicon.svg
│   ├── pwa-192x192.png
│   ├── pwa-512x512.png
│   ├── robots.txt
│   └── apple-touch-icon.png
├── src/
│   ├── components/
│   │   ├── base/
│   │   │   └── reactive-element.ts
│   │   ├── counter/
│   │   │   ├── counter-element.ts
│   │   │   ├── counter-element.test.ts
│   │   │   └── counter-element.stories.ts
│   │   └── todo/
│   │       ├── todo-element.ts
│   │       ├── todo-element.test.ts
│   │       └── todo-element.stories.ts
│   ├── main.ts
│   └── style.css
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
└── vitest.config.ts
```

## 18. 発展的なトピック

### 18.1 状態管理の改善

より複雑なアプリケーションでは、RxJSを使った中央集権的な状態管理を検討できます。例えば以下のようなサービスを実装できます：

```typescript
// src/services/store.ts
import { BehaviorSubject, Observable } from 'rxjs';
import { map } from 'rxjs/operators';

interface AppState {
  todos: TodoItem[];
  counter: number;
  // ...他の状態
}

export interface TodoItem {
  id: number;
  text: string;
  completed: boolean;
}

// 初期状態
const initialState: AppState = {
  todos: [],
  counter: 0
};

class Store {
  private state: BehaviorSubject<AppState>;
  
  constructor() {
    this.state = new BehaviorSubject<AppState>(initialState);
  }
  
  // 現在の状態を取得
  getState(): AppState {
    return this.state.getValue();
  }
  
  // 状態を監視
  select<T>(selector: (state: AppState) => T): Observable<T> {
    return this.state.pipe(
      map(state => selector(state))
    );
  }
  
  // 状態を更新
  setState(partialState: Partial<AppState>): void {
    this.state.next({
      ...this.getState(),
      ...partialState
    });
  }
  
  // Todo関連のアクション
  addTodo(text: string): void {
    const todos = [...this.getState().todos];
    const newTodo: TodoItem = {
      id: Date.now(),
      text,
      completed: false
    };
    
    this.setState({ todos: [...todos, newTodo] });
  }
  
  toggleTodo(id: number): void {
    const todos = [...this.getState().todos];
    const updatedTodos = todos.map(todo => 
      todo.id === id ? { ...todo, completed: !todo.completed } : todo
    );
    
    this.setState({ todos: updatedTodos });
  }
  
  deleteTodo(id: number): void {
    const todos = [...this.getState().todos];
    const updatedTodos = todos.filter(todo => todo.id !== id);
    
    this.setState({ todos: updatedTodos });
  }
  
  // カウンター関連のアクション
  incrementCounter(): void {
    this.setState({ counter: this.getState().counter + 1 });
  }
  
  decrementCounter(): void {
    this.setState({ counter: this.getState().counter - 1 });
  }
  
  resetCounter(): void {
    this.setState({ counter: 0 });
  }
}

// シングルトンインスタンスをエクスポート
export const store = new Store();
```

### 18.2 アクセシビリティの向上

Web Componentsでアクセシビリティを向上させるために、以下の点に注意します：

```typescript
// アクセシビリティを考慮したカウンターコンポーネントの例
protected render(): void {
  if (!this.shadowRoot) return;
  
  const count = this.getState<number>('count') || 0;
  
  this.shadowRoot.innerHTML = `
    <style>/* 省略 */</style>
    
    <div class="counter" role="region" aria-label="カウンター">
      <h2 id="counter-heading">RxJS Counter</h2>
      <div class="value" aria-live="polite" aria-atomic="true">${count}</div>
      <div class="controls">
        <button class="decrement" aria-label="カウントを減らす">-</button>
        <button class="reset" aria-label="カウントをリセット">Reset</button>
        <button class="increment" aria-label="カウントを増やす">+</button>
      </div>
    </div>
  `;
}
```

### 18.3 コンポーネント間通信

Web Componentsでのコンポーネント間通信には、カスタムイベントを使用します：

```typescript
// イベントを発行するコンポーネント
this.dispatchEvent(new CustomEvent('todo-added', {
  bubbles: true,  // DOMツリーを上に伝播
  composed: true, // Shadow DOM境界を越える
  detail: { id: newTodo.id, text: newTodo.text }
}));

// イベントをリッスンする側
document.addEventListener('todo-added', (e: Event) => {
  const detail = (e as CustomEvent).detail;
  console.log(`新しいTodo追加: ${detail.text}`);
});
```

## 19. デプロイ

ビルドされたアプリケーションは、静的ホスティングサービスにデプロイできます：

### GitHub Pages

```bash
# gh-pagesパッケージのインストール
npm install -D gh-pages

# package.jsonにデプロイスクリプトを追加
# "deploy": "npm run build && gh-pages -d dist"

# デプロイ実行
npm run deploy
```

### Netlify / Vercel

- GitHubリポジトリと連携
- ビルドコマンド: `npm run build`
- 公開ディレクトリ: `dist`

## 20. まとめ

この構成により、以下のメリットを持つモダンなフロントエンド開発環境が構築できます：

- **Web Components**: フレームワークに依存しない再利用可能なコンポーネント
- **TypeScript**: 静的型付けによる安全なコーディング
- **RxJS**: リアクティブプログラミングによる状態管理と非同期処理
- **Vitest**: 高速なテスト実行環境
- **PWA**: オフライン対応とネイティブアプリに近い体験
- **Storybook**: コンポーネントの開発・テスト・ドキュメント化
- **Vite**: 高速な開発サーバーとビルドツール

これらの技術スタックを組み合わせることで、モダンで保守性の高いWebアプリケーションを開発することができます。