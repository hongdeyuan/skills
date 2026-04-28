# i18n 项目初始化指南

## 概述

为 React/TypeScript 项目搭建完整的 i18n 国际化基础设施。适用于使用 Vite + Module Federation 的微前端项目。

## 依赖安装

```bash
# 运行时依赖
pnpm add i18next react-i18next i18next-browser-languagedetector

# 开发依赖
pnpm add -D i18next-scanner eslint-plugin-i18next
```

## 目录结构

```
src/
├── i18n/
│   ├── index.ts           # 入口，导出 t, i18n, i18nService
│   ├── i18nService.ts     # i18n 服务（实例管理、语言切换）
│   ├── config.js          # i18next-scanner 配置
│   ├── README.md          # 使用文档
│   └── locale/
│       ├── en.json        # 英文翻译
│       └── zh.json        # 中文翻译（key=value=中文原文）
├── lib/
│   └── useLanguageChange.ts  # 语言切换 Hook
└── constants/
    └── index.ts           # COOKIE_LANG_KEY 等常量
```

## 关键文件模板

### i18nService.ts

核心 i18n 服务，负责：
- 管理 i18next 实例
- 语言检测（cookie → localStorage → navigator）
- 语言切换
- 命名空间管理

关键配置：
```typescript
{
    interpolation: { prefix: '{', suffix: '}', escapeValue: false },
    contextSeparator: '_::_',
    keySeparator: false,
    nsSeparator: false,
    supportedLngs: ['zh_CN', 'en_US'],
    fallbackLng: 'zh_CN',
}
```

### index.ts（Module Federation 场景）

```typescript
import localI18nService from './i18nService';

let i18nService = localI18nService;

// 尝试从宿主应用获取共享 i18n 实例
try {
    const hostService = require('host/i18nService').default;
    if (hostService) i18nService = hostService;
} catch (e) {
    // 独立运行模式，使用本地实例
}

const instance = i18nService.getOrCreateI18nInstance('app_i18n', {
    ns: 'app-main',
    resources: { en: enLocale, zh: zhLocale },
});

export const { t, i18n } = instance;
export { i18nService };
```

### useLanguageChange.ts

```typescript
import { useState, useCallback } from 'react';
import { i18nService } from '../i18n';

export function useLanguageChange() {
    const [locale, setLocale] = useState(i18nService.getCurrentLanguage());

    const changeLocale = useCallback((newLocale: string) => {
        i18nService.changeLanguage(newLocale);
        setLocale(newLocale);
        // 持久化到 cookie 和 localStorage
        document.cookie = `lang=${newLocale};path=/`;
        localStorage.setItem('locale', newLocale);
    }, []);

    return { locale, changeLocale };
}
```

### package.json 脚本

```json
{
    "scripts": {
        "i18n:scanner": "cd src && npx i18next-scanner --config i18n/config.js"
    }
}
```

## 使用方式

### 基础翻译

```typescript
import { t } from '@/i18n';

// 简单词条
t('创建')  // → "Create"

// 带插值
t('共 {count} 项', { count: 10 })  // → "Total 10 items"

// 带上下文
t('状态', { context: 'column_header' })
```

### React 组件中

```tsx
import { useTranslation } from 'react-i18next';

function MyComponent() {
    const { t } = useTranslation('app-main');
    return <h1>{t('首页')}</h1>;
}
```

### 包含 HTML 的翻译

```tsx
import { Trans } from 'react-i18next';

<Trans i18nKey="点击<0>链接</0>查看详情">
    Click <a href="/link">link</a> to view details
</Trans>
```

## Module Federation 注意事项

1. `i18next` 必须配置为 MF singleton 共享：
```typescript
// vite.config.ts
shared: {
    i18next: { singleton: true, requiredVersion: '^23.0.0' },
    'react-i18next': { singleton: true },
}
```

2. 实例名必须唯一（避免宿主和远程应用冲突）
3. 命名空间也必须唯一

## Scanner 运行时机

- 新增中文词条后运行
- CI/CD 中可作为 pre-commit hook
- 建议定期运行检查是否有遗漏
