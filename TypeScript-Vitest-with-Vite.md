# Vanilla TypeScript + Vitest with Vite 環境構築

Vanilla TypeScript + Vitest + Viteの環境を構築します。Viteを使うことで、より高速な開発環境が実現できます。

## 環境構築手順

### 1. **Viteを使ってプロジェクトを初期化**
```bash
npm create vite@latest ts-vite-vitest -- --template vanilla-ts
cd ts-vite-vitest
```

### 2. **Vitestをインストール**
```bash
npm install -D vitest @vitest/ui @vitest/coverage-v8 jsdom 
```

### 3. **vite.config.tsの作成/編集**
#### vite.config.ts

```ts
/// <reference types="vitest" />
import { defineConfig } from 'vite';

export default defineConfig({
  plugins: [],
  server: {
    open: true
  },
  build: {
    outDir: 'dist',
    sourcemap: true
  },
  test: {
    globals: true,
    environment: 'jsdom',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html'],
    },
  }
});
```

### 4. **tsconfig.jsonの調整**

#### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "useDefineForClassFields": true,
    "module": "ESNext",
    "lib": ["ES2020", "DOM", "DOM.Iterable"],
    "skipLibCheck": true,

    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "resolveJsonModule": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,

    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    "noUncheckedSideEffectImports": true,

    /* Testing */
    "types": ["vitest/globals"]
  },
  // "include": ["src"]
  "include": ["src/**/*.ts", "tests/**/*.ts"],
}

```


### 5. **package.jsonのスクリプト部分を修正**

#### package.json
```json
{
  "name": "ts-vite-vitest",
  "private": true,
  "version": "0.0.0",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc && vite build",
    "preview": "vite preview",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage"
  },
  "devDependencies": {
    "@vitest/coverage-v8": "^3.1.1",
    "@vitest/ui": "^3.1.1",
    "jsdom": "^26.0.0",
    "typescript": "~5.7.2",
    "vite": "^6.2.0",
    "vitest": "^3.1.1"
  }
}

```


### 6. **ディレクトリ構造を作成**
```bash
mkdir -p src/utils tests
```

### 7. **サンプルのユーティリティクラスを作成**

#### src/tuils/calculator.ts
```ts
/**
 * 簡単な計算機能を提供するユーティリティ
 */
export class Calculator {
  /**
   * 2つの数値を加算します
   * @param a 最初の数値
   * @param b 2番目の数値
   * @returns 合計値
   */
  add(a: number, b: number): number {
    return a + b;
  }

  /**
   * 2つの数値を減算します
   * @param a 最初の数値
   * @param b 2番目の数値
   * @returns 差分値
   */
  subtract(a: number, b: number): number {
    return a - b;
  }

  /**
   * 2つの数値を乗算します
   * @param a 最初の数値
   * @param b 2番目の数値
   * @returns 積の値
   */
  multiply(a: number, b: number): number {
    return a * b;
  }

  /**
   * 2つの数値を除算します
   * @param a 最初の数値
   * @param b 2番目の数値
   * @returns 商の値
   * @throws 0で割ろうとした場合にエラーをスローします
   */
  divide(a: number, b: number): number {
    if (b === 0) {
      throw new Error('0で割ることはできません');
    }
    return a / b;
  }
}
```

### 8. **テストファイルを作成**

#### tests/calculator.test.ts

```ts
import { describe, it, expect, beforeEach } from 'vitest';
import { Calculator } from '../src/utils/calculator';

describe('Calculator', () => {
  let calculator: Calculator;

  beforeEach(() => {
    calculator = new Calculator();
  });

  describe('add', () => {
    it('正の数を加算できる', () => {
      expect(calculator.add(1, 2)).toBe(3);
    });

    it('負の数を加算できる', () => {
      expect(calculator.add(1, -2)).toBe(-1);
    });
  });

  describe('subtract', () => {
    it('正の数を減算できる', () => {
      expect(calculator.subtract(5, 2)).toBe(3);
    });

    it('結果が負になる減算ができる', () => {
      expect(calculator.subtract(2, 5)).toBe(-3);
    });
  });

  describe('multiply', () => {
    it('正の数を乗算できる', () => {
      expect(calculator.multiply(3, 4)).toBe(12);
    });

    it('負の数を乗算できる', () => {
      expect(calculator.multiply(3, -4)).toBe(-12);
    });

    it('0との乗算ができる', () => {
      expect(calculator.multiply(3, 0)).toBe(0);
    });
  });

  describe('divide', () => {
    it('正の数で除算できる', () => {
      expect(calculator.divide(10, 2)).toBe(5);
    });

    it('負の数で除算できる', () => {
      expect(calculator.divide(10, -2)).toBe(-5);
    });

    it('小数点の結果になる除算ができる', () => {
      expect(calculator.divide(5, 2)).toBe(2.5);
    });

    it('0で割るとエラーがスローされる', () => {
      expect(() => calculator.divide(10, 0)).toThrow('0で割ることはできません');
    });
  });
});
```

### 9. **メインファイルを編集**

#### src/main.ts
```ts
import './style.css';
import { Calculator } from './utils/calculator';

document.querySelector<HTMLDivElement>('#app')!.innerHTML = `
  <div>
    <h1>TypeScript + Vite + Vitest</h1>
    <div class="card">
      <div id="calculator">
        <h2>計算機デモ</h2>
        <div class="form">
          <input type="number" id="num1" value="10" />
          <select id="operation">
            <option value="add">+</option>
            <option value="subtract">-</option>
            <option value="multiply">×</option>
            <option value="divide">÷</option>
          </select>
          <input type="number" id="num2" value="5" />
          <button id="calculate">=</button>
          <span id="result">?</span>
        </div>
      </div>
    </div>
  </div>
`;

// 計算機のインスタンスを作成
const calculator = new Calculator();

// DOM要素の参照を取得
const num1Input = document.querySelector<HTMLInputElement>('#num1')!;
const num2Input = document.querySelector<HTMLInputElement>('#num2')!;
const operationSelect = document.querySelector<HTMLSelectElement>('#operation')!;
const calculateButton = document.querySelector<HTMLButtonElement>('#calculate')!;
const resultSpan = document.querySelector<HTMLSpanElement>('#result')!;

// 計算実行ハンドラー
calculateButton.addEventListener('click', () => {
  try {
    const a = parseFloat(num1Input.value);
    const b = parseFloat(num2Input.value);
    const operation = operationSelect.value;
    
    let result: number;
    
    switch (operation) {
      case 'add':
        result = calculator.add(a, b);
        break;
      case 'subtract':
        result = calculator.subtract(a, b);
        break;
      case 'multiply':
        result = calculator.multiply(a, b);
        break;
      case 'divide':
        result = calculator.divide(a, b);
        break;
      default:
        throw new Error('不明な操作です');
    }
    
    resultSpan.textContent = result.toString();
  } catch (error) {
    if (error instanceof Error) {
      resultSpan.textContent = `エラー: ${error.message}`;
    } else {
      resultSpan.textContent = '不明なエラーが発生しました';
    }
  }
});
```


### 10. **CSSを修正**

#### src/style.css
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
  -webkit-text-size-adjust: 100%;
}

a {
  font-weight: 500;
  color: #646cff;
  text-decoration: inherit;
}
a:hover {
  color: #535bf2;
}

body {
  margin: 0;
  display: flex;
  place-items: center;
  min-width: 320px;
  min-height: 100vh;
}

h1 {
  font-size: 3.2em;
  line-height: 1.1;
}

#app {
  max-width: 1280px;
  margin: 0 auto;
  padding: 2rem;
  text-align: center;
}

.card {
  padding: 2em;
}

.form {
  display: flex;
  gap: 10px;
  align-items: center;
  justify-content: center;
  margin-top: 20px;
}

input, select, button {
  padding: 8px;
  font-size: 16px;
  border-radius: 4px;
  border: 1px solid #333;
}

button {
  cursor: pointer;
  background-color: #646cff;
  color: white;
  border: none;
}

button:hover {
  background-color: #535bf2;
}

#result {
  font-size: 24px;
  font-weight: bold;
  padding: 0 10px;
}

@media (prefers-color-scheme: light) {
  :root {
    color: #213547;
    background-color: #ffffff;
  }
  a:hover {
    color: #747bff;
  }
  button {
    background-color: #f9f9f9;
  }
}
```


## 使用方法

### 1. **開発サーバーの起動**
```bash
npm run dev
```

### 2. **テストの実行**
```bash
npm test
```

### 3. **ウォッチモードでのテスト実行**
```bash
npm run test:watch
```

### 4. **UIモードでのテスト実行**
```bash
npm run test:ui
```

### 5. **カバレッジレポートの生成**
```bash
npm run test:coverage
```

### 6. **ビルド実行**
```bash
npm run build
```

### 7. **ビルド結果のプレビュー**
```bash
npm run preview
```

## TDDの開発サイクル

この環境でTDDを実践する際の推奨フローは以下のとおりです：

1. **Red**: テストディレクトリ（tests/）に新しい機能の失敗するテストを書く
2. **Green**: src/ディレクトリ内に実装を書いて、テストが通るようにする
3. **Refactor**: コードがきれいになるようリファクタリングする

`npm run test:watch`コマンドを実行しておくと、ファイルの変更を検知して自動的にテストが実行されるため、TDDサイクルを効率的に回すことができます。

また、Viteを使うことで開発サーバーが高速に起動し、変更がリアルタイムでブラウザに反映されるため、UIの確認も同時に行うことができます。

この環境はTypeScript、Vite、Vitestを組み合わせた基本的な構成です。プロジェクトの要件に応じて、ESLint、Prettier、その他のライブラリや設定を追加してカスタマイズすることが可能です。