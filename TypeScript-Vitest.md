# Vanilla TypeScriptとVitest
Vanilla TypeScriptとVitestの環境を構築するためのステップをご案内します。

## 環境構築手順

### 1. **プロジェクトの初期化**
```bash
mkdir ts-vitest-project
cd ts-vitest-project
npm init -y
```

### 2. **必要な依存関係のインストール**
```bash
npm install -D typescript vitest @vitest/ui ts-node
```

### 3. **TypeScriptの設定ファイル (tsconfig.json) の作成**
```bash
npx tsc --init
```

### 4. **tsconfig.jsonの編集**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "ESNext",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "strict": true,
    "outDir": "dist",
    "sourceMap": true,
    "declaration": true,
    "resolveJsonModule": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "**/*.test.ts"]
}
```

### 5. **Vitestの設定ファイル (vitest.config.ts) の作成**

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    environment: 'node',
    include: ['src/**/*.test.ts'],
    coverage: {
      reporter: ['text', 'json', 'html'],
    },
  },
});
```

### 6. **プロジェクトの構造を作成**
```bash
mkdir -p src/utils
```

### 7. **サンプルのユーティリティ関数とテストを作成**
#### src/utils/calculator.ts
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

#### src/utils/calculator.test.ts
```ts
import { describe, it, expect } from 'vitest';
import { Calculator } from './calculator';

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


### 8. **package.jsonにスクリプトを追加**
#### package.json
```json
{
  "name": "ts-vitest-project",
  "version": "1.0.0",
  "description": "Vanilla TypeScript with Vitest",
  "main": "dist/index.js",
  "type": "module",
  "scripts": {
    "build": "tsc",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest run --coverage"
  },
  "keywords": [
    "typescript",
    "vitest",
    "tdd"
  ],
  "author": "",
  "license": "ISC",
  "devDependencies": {
    "typescript": "^5.1.6",
    "vitest": "^0.34.6",
    "@vitest/ui": "^0.34.6",
    "ts-node": "^10.9.1"
  }
}
```

### 9. **メインのエントリーポイント作成**
#### src/index.ts
```ts
import { Calculator } from './utils/calculator';

// サンプル使用例
const calculator = new Calculator();
console.log(`1 + 2 = ${calculator.add(1, 2)}`);
console.log(`5 - 3 = ${calculator.subtract(5, 3)}`);
console.log(`4 * 6 = ${calculator.multiply(4, 6)}`);
console.log(`10 / 2 = ${calculator.divide(10, 2)}`);

// エクスポートもしておく
export { Calculator };
```


### 使用方法

1. **テストの実行**
```bash
npm test
```

2. **ウォッチモードでのテスト実行**
```bash
npm run test:watch
```

3. **UIモードでのテスト実行**
```bash
npm run test:ui
```

4. **カバレッジレポートの生成**
```bash
npm run test:coverage
```

5. **ビルド実行**
```bash
npm run build
```

### TDD開発の進め方

1. まず失敗するテストを書く（Red）
2. テストが通るように最小限のコードを実装する（Green）
3. コードをリファクタリングする（Refactor）

この環境では、`npm run test:watch`を実行しておくと、ファイルの変更を検知して自動的にテストが実行されるため、TDDのサイクルを効率的に回すことができます。

この設定はシンプルなTypeScriptプロジェクトでVitestを使うための基本的な環境です。必要に応じて、ESlintやPrettierなどのツールを追加して、コードの品質を向上させることも検討してみてください。

何か質問があればお気軽にどうぞ。