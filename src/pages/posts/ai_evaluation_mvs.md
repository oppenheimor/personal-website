---
title: AI Code 评测 - MostVolatileStock 模块
date: 2025-03-22T00:00:00.000+00:00
lang: zh
duration: 20min
author: 沈佳棋
---

实际需求验证 AI Code 能力。

## 一、需求信息

**需求类型：新的业务模块**

<img src="/ai_evaluation_mvs/1.png" />

<img src="/ai_evaluation_mvs/2.png" />

## 二、需求验证点

| 验证点 | 需求是否涉及 |
| --- | --- |
| UI 效果还原 | ✅ |
| 交互效果还原 | ✅ |
| 响应式设计 | ✅ |
| 复用已有组件 | ✅ |
| Mock 数据 | ✅ |
| 取数 | ✅ |
| 多语言 | ✅ |
| 埋点 | ✅ |
| SSR | ✅ |
| SEO | ✅ |
| 性能 | ✅ |
| APM | ❌ |
| 代码规范性 | ✅ |

## 三、验证细节

### 3.1 UI 效果还原

```js
// prompt
帮我基于上传的 UI 稿生成 MostVolatileStocks 组件；
组件放在 apps/web/home/components 目录下，并在 apps/web/home/components/StockSection/index.tsx 中进行引入
```

**生成代码**：

```tsx
// apps/web/home/components/StockSection/index.tsx

import React from 'react';
import Link from 'next/link';
import Image from 'next/image';

interface StockItem {
  id: string;
  name: string;
  symbol: string;
  logo: string;
  price: number;
  change: number;
  marketStatus: 'open' | 'closed';
  sparkline?: string;
}

interface MostVolatileStocksProps {
  stocks: StockItem[];
}

const MostVolatileStocks: React.FC<MostVolatileStocksProps> = ({ stocks }) => {
  return (
    <section className="w-full max-w-7xl mx-auto px-4 py-8">
      <header className="flex items-center justify-between mb-8">
        <h2 className="text-2xl md:text-[28px] font-bold leading-[34px]">
          Most volatile stocks
        </h2>
        <Link
          href="/stocks/volatile"
          className="inline-flex items-center"
          aria-label="View all volatile stocks"
        >
          <span className="w-4 h-4 bg-black mask-chevron-right" />
        </Link>
      </header>

      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4 md:gap-8">
        {stocks?.map((stock) => (
          <article
            key={stock.id}
            className="flex flex-col gap-2"
          >
            <Link
              href={`/stocks/${stock.symbol.toLowerCase()}`}
              className="p-4 rounded-lg hover:bg-gray-50 transition-colors"
            >
              <div className="flex items-center gap-3">
                <div className="relative w-9 h-9 rounded-full overflow-hidden">
                  <Image
                    src={stock.logo}
                    alt={`${stock.name} logo`}
                    fill
                    className="object-cover"
                  />
                </div>

                <div className="flex-1">
                  <div className="flex items-center gap-1">
                    <span className="text-base font-medium line-clamp-1">
                      {stock.name}
                    </span>
                    <span className={`w-[18px] h-[18px] rounded flex items-center justify-center ${
                      stock.marketStatus === 'open' ? 'bg-green-50' : 'bg-gray-50'
                    }`}>
                      <span className={`w-2.5 h-1 rounded ${
                        stock.marketStatus === 'open' ? 'bg-green-600' : 'bg-gray-400'
                      }`} />
                    </span>
                  </div>

                  <div className="mt-1.5 inline-flex px-2 py-1 bg-black/5 rounded-md">
                    <span className="text-xs">{stock.symbol}</span>
                  </div>
                </div>

                {stock.sparkline && (
                  <div className="w-16 h-10">
                    <Image
                      src={stock.sparkline}
                      alt="Price trend"
                      width={64}
                      height={40}
                    />
                  </div>
                )}

                <div className="flex flex-col items-end gap-1.5">
                  <div className="flex items-baseline gap-0.5">
                    <span className="text-base font-medium">
                      {new Intl.NumberFormat('en-US', {
                        minimumFractionDigits: 2,
                        maximumFractionDigits: 2
                      }).format(stock.price)}
                    </span>
                    <span className="text-xs text-gray-500">USD</span>
                  </div>
                  <span className={`text-base ${
                    stock.change >= 0 ? 'text-[#009b67]' : 'text-red-500'
                  }`}>
                    {stock.change >= 0 ? '+' : ''}{stock.change}%
                  </span>
                </div>
              </div>
            </Link>
            <hr className="border-gray-100" />
          </article>
        ))}
      </div>
    </section>
  );
};

export default MostVolatileStocks;
```

**效果**：

<img src="/ai_evaluation_mvs/3.png" />

**问题**:

1. 布局错误。组件根节点处设置了 tailwind 类："px-4 py-8 mx-auto w-full max-w-7xl"，应该去掉。组件的边距应该由组装的容器层来设置；
2. Mock 的图片、图标都是不存在的地址，展示不出来；
3. 从 tsx 来看结构拆分的问题：

<img src="/ai_evaluation_mvs/3.png" />

<img src="/ai_evaluation_mvs/4.png" />

<img src="/ai_evaluation_mvs/5.png" />

<img src="/ai_evaluation_mvs/6.png" />

**思考**:

1. 架构师的角色：将页面/业务模块拆解成一个个块(block)，将这些块分成两部分：可以复用组件的 / 需要新开发的（方案：结合本地(项目) + 云端(例如物料平台）的信息匹配可以直接复用的组件）
这波分析完之后再得到一个合适的数据结构。
2. 按开发工程师的习惯，到这里就该停一停，进行一波调整了，否则后面的需求(其它验证点)再往上堆就越来越不可控了。

**prompt 调整**：

```tsx
// prompt
帮我基于上传的 UI 稿生成 MostVolatileStocks 组件；
组件放在 apps/web/home/components 目录下，并在 apps/web/home/components/StockSection/index.tsx 中进行引入

要求：
1. 组件根节点不要设置跟布局相关的样式，例如内边距、外边距、最大宽等；
2. 帮我 Mock 数据，其中涉及到图片的值请使用真实 url 地址，这里你可以使用：https://cdn.ainvest.com/icon/us/etf/SQQQ.png；
3. 我会给你组件的名称和路径，请你仔细分析组件的 api 并根据你对需求的理解在合适的地方正确使用：
[
  { componentName: "TitleJump", "componentPath": "apps/web/home/components/Common/TitleJump.tsx" },
  { componentName: "MarketSymbolLogo", "componentPath": "packages/business/src/MarketSymbolLogo/index.tsx" },
  { componentName: "MarketStatusIcon", "componentPath": "packages/business/src/MarketStatusIcon/index.tsx" },
  { componentName: "ChartSpark", "componentPath": "packages/business/src/ChartSpark/index.tsx" },
  { componentName: "FormatNumber", "componentPath": "apps/web/home/components/Common/FormatNumber.tsx" }
]
```

**问题**：

1. 组件路径导入错误，需要二次修正：

<img src="/ai_evaluation_mvs/7.png" />

2. Mock 数据偷懒，需要二次修正：

<img src="/ai_evaluation_mvs/8.png" />

3. 布局理解错误（原本应该是按行分组，它这里变成了按列分组，还专门写了个逻辑处理这个）:

<img src="/ai_evaluation_mvs/9.png" />

<img src="/ai_evaluation_mvs/10.png" />

4. 对组件传参的理解有问题（MarketSymbolLogo 组件没有 url 属性，FormatNumber 应该传 numValue 属性，而不是 value），看来还是需要把具体入参信息给模型：

<img src="/ai_evaluation_mvs/11.png" />

**prompt 调整：**

```tsx
// prompt
帮我基于上传的 UI 稿生成 MostVolatileStocks 组件；
组件放在 apps/web/home/components 目录下，并在 apps/web/home/components/StockSection/index.tsx 中进行引入

要求：
1. 组件根节点不要设置跟布局相关的样式，例如内边距、外边距、最大宽等；
2. 组件里的子项目按行排列；
3. 帮我 Mock 数据，其中涉及到图片的值请使用真实 url 地址，这里你可以使用：https://cdn.ainvest.com/icon/us/etf/SQQQ.png，帮我 mock 10 条数据；
4. 我会给你组件的名称和路径以及 api，请根据你对需求的理解在合适的地方正确使用：
[
  {"componentName":"TitleJump","componentPath":"apps/web/home/components/Common/TitleJump.tsx","props":"interface TitleJumpProps{/**标题文本@default'' /title?:string;/**点击标题时的回调函数@default()=>{} /onTitleClick?:()=>void;/**标题大小，影响字体大小lg:36px md:28px sm:20px@default'md' /size?:'lg'|'md'|'sm';/ 是否显示右侧箭头图标@default true /icon?:boolean;/**链接地址@default'#' /hrefString?:string;}"},
  {"componentName":"MarketSymbolLogo","componentPath":"packages/business/src/MarketSymbolLogo/index.tsx","props":"interface MarketSymbolLogoProps{/**股票/ETF/加密货币的代码@default'' /symbol?:string;/ 资产类型@defaultM_Market_Type_Enum.Stock /marketType?:M_Market_Type_Enum;/**图标尺寸@default'36px' /size?:string;}/* 市场类型枚举 /enum M_Market_Type_Enum{Stock='stock',ETF='etf',Crypto='crypto',Block='block'}"},
  {"componentName":"MarketStatusIcon","componentPath":"packages/business/src/MarketStatusIcon/index.tsx","props":"interface MarketStatusIconProps{/* 市场状态可选值:-TradeClass.PreMarket:盘前交易-TradeClass.PostMarket:盘后交易-TradeClass.Closed:已收盘-'holiday':假期休市@defaultundefined /status:string;}/* 返回值类型 /interface StatusInfo{/* 提示文本 /tooltip:string;/ 状态图标 /icon:React.ReactNode;}/ 内部使用的TradeClass类型通过动态导入获取 /interface TradeClass{PreMarket:string;PostMarket:string;Closed:string;}"},
  {"componentName":"ChartSpark","componentPath":"packages/business/src/ChartSpark/index.tsx","props":"interface ChartSparkProps{/ 股票代码信息 /codeInfo:{/**股票代码@default'' /symbol:string;/**市场代码@default'' /market:string;};/**图表宽度，支持tailwind类名@default'w-[64px]' /width?:string;}/ 图表样式配置类型 /interface ChartStyles{grid:{show:boolean;};candle:{type:'area';area:{lineSize:number;lineColor:string;backgroundColor:Array<{offset:number;color:string;}>;};priceMark:{pre:{show:boolean;}};tooltip:{showRule:'none';}};xAxis:{show:boolean;splitNum:number;};yAxis:{show:boolean;inside:boolean;preInCenter:boolean;};crosshair:{show:boolean;};}/ 图表初始化参数类型 /interface ChartParams{id:string;code:string;market:string;data_class:'trend';trade_class:'intraday';trade_date:number;}"},
  {"componentName":"FormatNumber","componentPath":"apps/web/home/components/Common/FormatNumber.tsx","props":"interface FormatNumberProps{/ 需要格式化的数值@default0 /numValue?:number;/**颜色显示类型-'text':文字颜色-'background':背景色@default'text' /colorType?:'text'|'background';/**字体大小，支持tailwind类名@default'text-[16px]' /fontSize?:string;/**字体粗细，支持tailwind类名@default'font-semibold' /fontWeight?:string;}/* CSS变量类型 /type PriceColorVars={'--atom-color-price-even':string;'--atom-color-price-up':string;'--atom-color-price-down':string;}"}
]
```

**问题：**

1. 大忌：mock 数据和视图层强耦合！而且这里在强行用 JS 构建一个 4 * 3 的布局结构，后面就没法儿做响应式了。（模型发挥不稳定 - Claude3.5-latest）

<img src="/ai_evaluation_mvs/12.png" />

2. 这里在调用组件进行传参的时候，把参数给写死了

<img src="/ai_evaluation_mvs/13.png" />

3. 建议：调用组件进行传参的时候参数如果有默认值的话是不用传的：

<img src="/ai_evaluation_mvs/14.png" />

**prompt 调整：**

```tsx
// prompt
帮我基于上传的 UI 稿生成 MostVolatileStocks 组件；
组件放在 apps/web/home/components 目录下，并在 apps/web/home/components/StockSection/index.tsx 中进行引入

要求：
1. 组件根节点不要设置跟布局相关的样式，例如内边距、外边距、最大宽等；
2. 布局：组件里的子项目按行排列；
3. 响应式：组件需要考虑响应式，当屏幕宽度 >= 1920 时，列表展示 4 列；在 1440 ~ 1920 区间内，展示 3 列，在 1024 ~ 1440 区间内，展示 2 列；< 1024 展示 1 列；
4. 帮我 Mock 数据，其中涉及到图片的值请使用真实 url 地址，这里你可以使用：https://cdn.ainvest.com/icon/us/etf/SQQQ.png，帮我 mock 10 条数据；
5. 我会给你组件的名称和路径以及 api，请根据你对需求的理解在合适的地方正确使用：
[
  {"componentName":"TitleJump","componentPath":"apps/web/home/components/Common/TitleJump.tsx","props":"interface TitleJumpProps{/**标题文本@default'' /title?:string;/**点击标题时的回调函数@default()=>{} /onTitleClick?:()=>void;/**标题大小，影响字体大小lg:36px md:28px sm:20px@default'md' /size?:'lg'|'md'|'sm';/ 是否显示右侧箭头图标@default true /icon?:boolean;/**链接地址@default'#' /hrefString?:string;}"},
  {"componentName":"MarketSymbolLogo","componentPath":"packages/business/src/MarketSymbolLogo/index.tsx","props":"interface MarketSymbolLogoProps{/**股票/ETF/加密货币的代码@default'' /symbol?:string;/ 资产类型@defaultM_Market_Type_Enum.Stock /marketType?:M_Market_Type_Enum;/**图标尺寸@default'36px' /size?:string;}/* 市场类型枚举 /enum M_Market_Type_Enum{Stock='stock',ETF='etf',Crypto='crypto',Block='block'}"},
  {"componentName":"MarketStatusIcon","componentPath":"packages/business/src/MarketStatusIcon/index.tsx","props":"/**MarketStatusIcon 组件的属性类型*/interface MarketStatusIconProps{/**市场交易状态@type TradeClass|'holiday'@required*/status:TradeClass|'holiday';}/**交易状态枚举*/export declare enum TradeClass{/**盘前交易*/PreMarket="pre_market",/**盘中交易*/Intraday="intraday",/**盘后交易*/PostMarket="post_market",/**已收盘*/Closed="closed"}"},
  {"componentName":"ChartSpark","componentPath":"packages/business/src/ChartSpark/index.tsx","props":"/**ChartSpark 组件的属性类型*/interface ChartSparkProps{/**股票代码信息@example{symbol:"NVDA",market:"185"}//英伟达{symbol:"AAPL",market:"185"}//苹果{symbol:"TSLA",market:"185"}//特斯拉{symbol:"GOOG",market:"185"}//谷歌{symbol:"BTCUSD",market:"UDC"}//比特币@required*/codeInfo:{/**股票代码@default''*/symbol:string;/**市场代码@default''*/market:string;};/**图表宽度，支持tailwind类名@default'w-[64px]'*/width?:string;}"},
  {"componentName":"FormatNumber","componentPath":"apps/web/home/components/Common/FormatNumber.tsx","props":"interface FormatNumberProps{/ 需要格式化的数值@default0 /numValue?:number;/**颜色显示类型-'text':文字颜色-'background':背景色@default'text' /colorType?:'text'|'background';/**字体大小，支持tailwind类名@default'text-[16px]' /fontSize?:string;/**字体粗细，支持tailwind类名@default'font-semibold' /fontWeight?:string;}/* CSS变量类型 /type PriceColorVars={'--atom-color-price-even':string;'--atom-color-price-up':string;'--atom-color-price-down':string;}"}
]
6. 代码规范：请注意按高标准的代码规范进行编写，注意代码的分层、扩展性和可维护性，例如不要将 mock 数据和视图层偶合在一起等；
```

**问题:**

1. 从其他文件里导了个不能在这个场景里使用的 ts 类型 IndexSortItem（应该是从其它文件里学过来的），用了这个类型约束 props 之后，会全面报 ts 类型错误。-- 人工修正

<img src="/ai_evaluation_mvs/15.png" />

2. 文件结尾给我生成了一段注释来解释功能，关键是这是个 python 注释，在 js 里面会报错。（模型稳定性）

<img src="/ai_evaluation_mvs/16.png" />

3. Mock 数据多样性。太单一了。拿 TSLA 的数据循环了 10 遍，这块跟没给它提供我司的数据样本也有关系。

<img src="/ai_evaluation_mvs/17.png" />

4. 使用 ChartSpark 火花图的组件会报错。因为 HxKLine 里面用了 window 变量，SSR 阶段是不能使用的，这种场景是需要特殊处理的，处理方法：组件使用 dynamic 进行动态引入，组件外层使用 `<NovaMounted>` 标签进行包裹，
NovaMounted 是 ainvest 官网项目里封装的一个方法，用于让组件在 CSR 阶段执行。需要让模型知道这个规则和用法，特别是要让模型知道什么情况下要按这种方式去处理。

<img src="/ai_evaluation_mvs/18.png" />

处理方法:

<img src="/ai_evaluation_mvs/19.png" />

<img src="/ai_evaluation_mvs/20.png" />

**可圈可点的地方**：

1. 分层良好，拆分了列表的容器和 item:

<img src="/ai_evaluation_mvs/21.png" />

2. 在合适的地方正确使用了组件，并且传参正确:

<img src="/ai_evaluation_mvs/22.png" />

3. 响应式表现正常，效果：

<img src="/ai_evaluation_mvs/23.png" />

<img src="/ai_evaluation_mvs/24.png" />

<img src="/ai_evaluation_mvs/25.png" />

<img src="/ai_evaluation_mvs/26.png" />

截至目前为止的代码(经过一点点人工微调： SSR 特殊处理、包路径导入调整、ts 类型调整)：

```tsx
import React from 'react';

import TitleJump from '../Common/TitleJump';
import { MarketSymbolLogo } from '@ainvest-nova/business/MarketSymbolLogo';
import { MarketStatusIcon } from '@ainvest-nova/business/MarketStatusIcon';
import FormatNumber from '../Common/FormatNumber';
import { M_Market_Type_Enum } from '@hx-klineview/models'
import { NovaMounted } from '@ainvest-nova/layout/WebsiteLayout/ClientInitializer';
import dynamic from 'next/dynamic';

const ChartSpark = dynamic(() => import('@ainvest-nova/business/ChartSpark'), {
  ssr: false
});

interface StockItem {
  id: string;
  symbol: string;
  market: string;
  name: string;
  price: number;
  changePercent: number;
  marketType: M_Market_Type_Enum;
  status: string;
  logoUrl?: string;
}

interface StockItemProps {
  item: StockItem;
}

interface MostVolatileStocksProps {
  data?: StockItem[];
}

// Mock data
const mockData: StockItem[] = Array(10).fill(null).map((_, index) => ({
  id: `${index}`,
  symbol: 'TSLA',
  market: '185',
  name: 'Tesla Inc.',
  price: 244.14 + index,
  changePercent: 8.03 + (index * 0.5),
  marketType: M_Market_Type_Enum.Stock,
  status: 'pre_market',
  logoUrl: 'https://cdn.ainvest.com/icon/us/etf/SQQQ.png'
}));

const StockItem: React.FC<StockItemProps> = ({ item }) => {
  return (
    <div className="py-2">
      <div className="flex gap-3 items-center">
        <div className="flex gap-2 items-center">
          <MarketSymbolLogo
            symbol={item.symbol}
            marketType={item.marketType}
            size="36px"
          />
          <div className="flex flex-col gap-1.5">
            <div className="flex gap-1 items-center">
              <span className="text-base leading-snug">{item.name}</span>
              <MarketStatusIcon status={ item.status } />
            </div>
            <div className="px-2 py-1 rounded-md bg-black/5">
              <span className="text-xs">{item.symbol}</span>
            </div>
          </div>
        </div>

        <div className="flex flex-1 justify-center items-center">
          <NovaMounted>
            <ChartSpark
              codeInfo={{
                symbol: item.symbol,
                market: item.market
              }}
              width="w-16"
            />
          </NovaMounted>
        </div>

        <div className="flex flex-col items-end gap-1.5">
          <div className="flex items-end gap-0.5">
            <span className="text-base leading-snug">
              {item.price.toFixed(2)}
            </span>
            <span className="text-[11px] leading-[18px]">USD</span>
          </div>
          <FormatNumber
            numValue={item.changePercent}
            fontSize="text-base"
            fontWeight="font-normal"
          />
        </div>
      </div>
    </div>
  );
};

const MostVolatileStocks: React.FC<MostVolatileStocksProps> = ({
  data = mockData
}) => {
  return (
    <div>
      <div className="mb-8">
        <TitleJump
          title="Most volatile stocks"
          size="md"
          icon
          onTitleClick={() => {}}
        />
      </div>

      <div className="grid gap-4 divide-y divide-[#f2f2f2]
        grid-cols-1
        lg:grid-cols-2
        xl:grid-cols-3
        2xl:grid-cols-4"
      >
        {data.map((item) => (
          <StockItem key={item.id} item={item} />
        ))}
      </div>
    </div>
  );
};

export default MostVolatileStocks;
```

终于把大致的布局给调出来了！！（涉及到 UI 效果还原、响应式设计、 使用已有组件、Mock 数据、代码规范性、 SSR）
用时（8h 左右）

**心得**：

1. 开发这样一个复杂的业务模块，按前端工程师的习惯，更偏向渐进式开发。先把地基打好（上述的一些工作），然后再往上继续垒业务逻辑，所以上面那部分代码是不希望再有什么大改动的。如果模型不稳定的话，再重新生成一遍的时候会带来很多的不确定性，有些生成效果甚至会倒退，带来额外的调整的工作量。

一次生成面临的问题：从  figma 解析 → 拆解组件 → 生成布局 → 响应式处理 → 使用组件 → 填充组件没有覆盖的 UI 细节 → 定义页面数据结构，生成 mock 数据 → 处理 SSR → 处理取数、业务逻辑 → 处理埋点、多语言 → ...，同时还需要关注代码的质量，中间会有很多的信息损耗和不稳定因素的存在。

2. 前端用于驱动页面展示的数据模型构建很关键（后端数据 + 控制视图状态的变量）

接下来需要处理的：交互（悬浮效果、跳转）、取数（接口取数、行情取数）、业务逻辑、多语言、埋点、SEO、性能问题处理（每支股票都有火花图，直接展示出来会卡顿，需要考虑监听该模块滚动到可视区域再获取行情数据）

### 3.2 逻辑处理

取数 & 业务逻辑（前端实际在做这块需求的时候找产品、后端对需求，理解需求耗时 1d，熟悉封装好的取数工具+梳理代码编写逻辑 1.5d，总用时 2.5d）：

```tsx
// 这块代码展示的是线上人肉写的真实代码

1. 首页会在入口处批量调用各个模块的接口，接口地址：//extquote.ainvest.com/index_api/v1/sort。不同的模块传参不同，MostVolatileStocks 模块：
传参（DataAPI 格式）：{
        symbols: {
          type: 'block_id',
          values: 'C191',
        },
        indexes: [
          {
            index_id: '55',
            index_attr: {
              trade_class: 'intraday',
            },
          },
          {
            index_id: '10',
            index_attr: {
              trade_class: 'intraday',
            },
          },
          {
            index_id: '199112',
            index_attr: {
              trade_class: 'intraday',
            },
          },
        ],
        sort: {
          sort_index: {
            index_id: 'price_amplitude_ratio_pct',
          },
          sort_begin: 0,
          sort_count: 16,
          sort_direct: 'descend',
        },
      }
返回的数据格式：
{
  "status_code": 0,
  "status_msg": "Success.",
  "data": {
    "data": [
      {
        "code": "ANTE",
        "market": "186",
        "values": [
          { "index_id": "price_amplitude_ratio_pct", "value": 99.422179, "index_attr": { "value_type": "ratio" } },
          { "index_id": "199112", "value": 97.93459552, "index_attr": { "trade_class": "intraday", "value_type": "ratio" } },
          { "index_id": "55", "value": "AirNet Technology", "index_attr": { "trade_class": "intraday" } },
          { "index_id": "10", "value": 1.61, "index_attr": { "trade_class": "intraday", "value_type": "price" } }
        ]
      },
      // .......
    ],
    "summary": { "total_count": 6494 }
  }
}
拿到的这个数据作为 MostVolatileStocks 模块的参数。

2. 拿到原始后端数据之后前端需要处理一道，把数据处理成适合前端展示的格式（扁平化）：

const cards = mostVolatileStocksData.map(item => ({
    symbol: item.code,
    market: item.market,
    companyName: item.values.find(v => v.index_id === '55')?.value as string,
    type: 'stock',
    price: item.values.find(v => v.index_id === '10')?.value,
    change: item.values.find(v => v.index_id === '199112')?.value,
    status: ''
  }));
遍历这个数组，把每一项的数据传给子组件（StockItem）：
<StockListItem
   key={stock.symbol}
   symbol={stock.symbol}
   companyName={stock.companyName}
   price={stock.price}
   priceChange={stock.change}
   market={stock.market}
   type={stock.type}
   status={stock.status}
   from='stockhighvolatility'
/>

3. 每个股票项（StockListItem）内部需要进行取数处理
订阅数据和订阅事件总线监听的方法需要一起执行，取消订阅数据和取消订阅事件总线监听的方法需要一起执行

// 订阅数据
const subscribeData = () => {
  hkv.getDataStore().addSubscribeCodeInfo({
    subCodeInfo: [{
      codeName: symbol,
      period: '',
      symbol: symbol,
      market: market,
      marketType: type,
      showCode: symbol,
      marketStatus: TradeClass.All
    }]
  });
};

// 取消订阅数据
const unsubscribeData = () => {
  hkv.getDataStore().removeSubscribeCodeInfo({
    codeName: symbol,
    symbol: symbol,
    market: market,
    period: '',
    marketType: type,
    showCode: symbol,
    marketStatus: TradeClass.All
  });
};

// 订阅事件总线监听
const subscribeEvent = () => {
  // 添加市场数据更新订阅
  window.__AINVEST_NOVA__.EventBus.on(Event_Quote_OnUpdateMarketData.key, handleMarketData);
  // 添加市场状态变化订阅
  window.__AINVEST_NOVA__.EventBus.on(Event_Quote_OnChangeMarketStatus.key, handleMarketStatus);
};

// 取消订阅事件总线监听
const unsubscribeEvent = () => {
  window.__AINVEST_NOVA__.EventBus.off(Event_Quote_OnUpdateMarketData.key, handleMarketData);
  window.__AINVEST_NOVA__.EventBus.off(Event_Quote_OnChangeMarketStatus.key, handleMarketStatus);
};

// 处理市场数据更新
// 几个状态：1. 股票价格、股票涨跌幅 2. "盘前盘后" 状态是否展示 3. "盘前盘后" 文本的值 4. "盘前盘后" 具体的数值
// 规则：1. 股票价格、涨跌幅在盘中才更新，否则默认用前面 DataAPI 里的值做兜底 2. "盘前盘后" 状态在盘中的时候不展示，其它情况皆展示 3. 当 "盘前盘后" 状态展示的时候，更新文本 "pre" "post"
// 4. 盘前或盘后的时候使用推送的数据，收盘的时候再请求一次快照，使用快照数据

const handleMarketData = (data: Event_Quote_OnUpdateMarketData.Payload) => {
  // 找到订阅的这支股票
  const newCodeInfo = data.dataList.find((item: any) => item.codeInfo.symbol === symbol && item.codeInfo.market === market);
  // 区分了几种状态：盘前、盘后、盘中、收盘
  // 如果是盘中则更新价格和涨跌幅
  // 如果为非盘中阶段则展示涨跌幅下面的字段，盘前展示盘前涨跌幅，盘后和非交易阶段展示盘后涨跌幅，盘中不展示这个字段
  if (newCodeInfo) {
    if (marketStatus === TradeClass.Intraday) {
      setNewPrice(Number(newCodeInfo.marketData.latestPrice));
      setNewPriceChange(Number(newCodeInfo.marketData.priceChangeRate));
      // 盘前 盘后不展示
      setPreOrPostStatus(false);
    }
    if (marketStatus === TradeClass.PreMarket) {
      // 展示盘前
      setPreOrPostPriceChange(Number(newCodeInfo.marketData.priceChangeRate));
      setPreOrPostText(langObj['AinvestWeb_Homepage_mainpage_target_pre'][language]);
      setPreOrPostStatus(true);
    }
    if (marketStatus === TradeClass.PostMarket) {
      // 展示盘后
      setPreOrPostPriceChange(Number(newCodeInfo.marketData.priceChangeRate));
      setPreOrPostText(langObj['AinvestWeb_Homepage_mainpage_target_post'][language]);
      setPreOrPostStatus(true);
    }
  }
};

// 处理市场状态变化
const handleMarketStatus = (data: Event_Quote_OnChangeMarketStatus.Payload) => {
  if (data.codeInfo.symbol === symbol && data.codeInfo.market === market && data.marketStatus !== marketStatus) {
    setMarketStatus(data.marketStatus);
    if (data.marketStatus === TradeClass.Closed) {
      //展示收盘 请求一次行情快照
      hkv.getDataStore().core.queryMarketSnapshot(
        [{
          symbol: symbol,
          market: market,
          marketId: 0,
          marketStatus: TradeClass.PostMarket,
          marketType: type
        }],
        hxModel.M_MARKET_QUERY_BASE_HQ_DATA_FIELDS,
        false
      ).then((res: any) => {
        const postCodeInfo = res.dataList.find((item: any) => item.codeInfo.symbol === symbol && item.codeInfo.market === market);
        if (postCodeInfo) {
          setPreOrPostPriceChange(Number(postCodeInfo.marketData.priceChangeRate));
          setPreOrPostText(langObj['AinvestWeb_Homepage_mainpage_target_post'][language]);
          setPreOrPostStatus(true);
        }
      });
    }
  }
};
```

**多语言、埋点、SEO 处理**。

**prompt：**

```tsx
帮我基于上传的 UI 稿生成 MostVolatileStocks 组件；
组件放在 apps/web/home/components 目录下，并在 apps/web/home/components/StockSection/index.tsx 中进行引入

要求：
1. 组件根节点不要设置跟布局相关的样式，例如内边距、外边距、最大宽等；
2. 布局：组件里的子项目按行排列；
3. 响应式：组件需要考虑响应式，当屏幕宽度 >= 1920 时，列表展示 4 列；在 1440 ~ 1920 区间内，展示 3 列，在 1024 ~ 1440 区间内，展示 2 列；< 1024 展示 1 列；
4. 帮我 Mock 数据，其中涉及到图片的值请使用真实 url 地址，这里你可以使用：https://cdn.ainvest.com/icon/us/etf/SQQQ.png，帮我 mock 10 条数据；
5. 我会给你组件的名称和路径以及 api，请根据你对需求的理解在合适的地方正确使用：
[
  {"componentName":"TitleJump","componentPath":"apps/web/home/components/Common/TitleJump.tsx","props":"interface TitleJumpProps{/**标题文本@default'' /title?:string;/**点击标题时的回调函数@default()=>{} /onTitleClick?:()=>void;/**标题大小，影响字体大小lg:36px md:28px sm:20px@default'md' /size?:'lg'|'md'|'sm';/ 是否显示右侧箭头图标@default true /icon?:boolean;/**链接地址@default'#' /hrefString?:string;}"},
  {"componentName":"MarketSymbolLogo","componentPath":"packages/business/src/MarketSymbolLogo/index.tsx","props":"interface MarketSymbolLogoProps{/**股票/ETF/加密货币的代码@default'' /symbol?:string;/ 资产类型@defaultM_Market_Type_Enum.Stock /marketType?:M_Market_Type_Enum;/**图标尺寸@default'36px' /size?:string;}/* 市场类型枚举 /enum M_Market_Type_Enum{Stock='stock',ETF='etf',Crypto='crypto',Block='block'}"},
  {"componentName":"MarketStatusIcon","componentPath":"packages/business/src/MarketStatusIcon/index.tsx","props":"/**MarketStatusIcon 组件的属性类型*/interface MarketStatusIconProps{/**市场交易状态@type TradeClass|'holiday'@required*/status:TradeClass|'holiday';}/**交易状态枚举*/export declare enum TradeClass{/**盘前交易*/PreMarket="pre_market",/**盘中交易*/Intraday="intraday",/**盘后交易*/PostMarket="post_market",/**已收盘*/Closed="closed"}"},
  {"componentName":"ChartSpark","componentPath":"packages/business/src/ChartSpark/index.tsx","props":"/**ChartSpark 组件的属性类型*/interface ChartSparkProps{/**股票代码信息@example{symbol:"NVDA",market:"185"}//英伟达{symbol:"AAPL",market:"185"}//苹果{symbol:"TSLA",market:"185"}//特斯拉{symbol:"GOOG",market:"185"}//谷歌{symbol:"BTCUSD",market:"UDC"}//比特币@required*/codeInfo:{/**股票代码@default''*/symbol:string;/**市场代码@default''*/market:string;};/**图表宽度，支持tailwind类名@default'w-[64px]'*/width?:string;}"},
  {"componentName":"FormatNumber","componentPath":"apps/web/home/components/Common/FormatNumber.tsx","props":"interface FormatNumberProps{/ 需要格式化的数值@default0 /numValue?:number;/**颜色显示类型-'text':文字颜色-'background':背景色@default'text' /colorType?:'text'|'background';/**字体大小，支持tailwind类名@default'text-[16px]' /fontSize?:string;/**字体粗细，支持tailwind类名@default'font-semibold' /fontWeight?:string;}/* CSS变量类型 /type PriceColorVars={'--atom-color-price-even':string;'--atom-color-price-up':string;'--atom-color-price-down':string;}"}
]
6. 代码规范：请注意按高标准的代码规范进行编写，注意代码的分层、扩展性和可维护性，例如不要将 mock 数据和视图层偶合在一起等；
7. 多语言处理：
    7.1 多语言所在的配置文件是 apps/web/home/models/langConfig.json；
    7.2 该模块大标题的多语言 Key 是 AinvestWeb_Homepage_mainpage_stock_volatile
    7.3 多语言请使用 @hx-klineview/core 库中的 hkv 方法，你可以参考其他模块的实现，例如：apps/web/home/components/MarketSummary/index.tsx
8. SEO 处理：请遵循 SEO 规范。所有的跳转交互元素均用 a 标签包裹（href属性值为绝对路径），页面模块导航大标题 h1 标签，二级标题 h2 标签，卡片（例如资讯卡片）中的标题用 h3 标签
9. 需要配置埋点：
     9.1 发送埋点的方法使用 window.__AINVEST_NOVA__.Lib.weblog?.report 方法
     9.2 参考 apps/web/home/components/AimeSection/index.tsx
10. 需要考虑性能，由于火花图非常消耗性能，当这个模块滚动到页面视图中央的时候再渲染火花图；
11. 交互效果：鼠标悬浮在每个股票项上的时候展示悬浮态样式；
```

## 四、问题清单

| 具体问题 | 问题分类 |
| --- | --- |
| 布局错误。组件根节点处设置了 tailwind 类：px-4 py-8 mx-auto w-full max-w-7xl，应该去掉。组件的边距应该由组装的容器层来设置； | 样式布局 |
| Mock 的图片、图标都是不存在的地址，展示不出来； | Mock 数据 |
| 结构拆分问题（不使用已有组件，一些封装在组件内部的隐藏逻辑模型是不知道的）| 结构拆分，使用组件 |
| 组件路径导入错误，需要二次修正 | 导入路径 |
| Mock 数据偷懒，需要二次修正 | Mock 数据 |
| 布局理解错误（原本应该是按行分组，它这里变成了按列分组，还专门写了个逻辑处理这个） | 样式布局 |
| 对组件传参的理解有问题（MarketSymbolLogo 组件没有 url 属性，FormatNumber 应该传 numValue 属性，而不是 value），看来还是需要把具体入参信息给模型 | 组件使用 |
| mock 数据和视图层强耦合！而且这里在强行用 JS 构建一个 4 * 3 的布局结构，后面就没法儿做响应式了。（模型发挥不稳定 - Claude3.5-latest） | 代码生成质量 |
| 这里在调用组件进行传参的时候，把参数给写死了 | 代码生成质量 |
| 建议：调用组件进行传参的时候参数如果有默认值的话是不用传的 | 组件使用 |
| 从其他文件里导了个不能在这个场景里使用的 ts 类型 IndexSortItem（应该是从其它文件里学过来的），用了这个类型约束 props 之后，会全面报 ts 类型错误。-- 人工修正 | 代码生成质量 |
| 文件结尾给我生成了一段注释来解释功能，关键是这是个 python 注释，在 js 里面会报错。（模型稳定性） | 代码生成质量 |
| Mock 数据多样性。太单一了。拿 TSLA 的数据循环了 10 遍 ，这块跟没给它提供我司的数据样本也有关系。 | Mock 数据 |
| 使用 ChartSpark 火花图的组件会报错。因为 HxKLine 里面用了 window 变量，SSR 阶段是不能使用的，这种场景是需要特殊处理的，处理方法：组件使用 dynamic 进行动态引入，组件外层使用 `<NovaMounted>` 标签进行包裹，
| NovaMounted 是 ainvest 官网项目里封装的一个方法，用于让组件在 CSR 阶段执行。需要让模型知道这个规则和用法，特别是要让模型知道什么情况下要按这种方式去处理。 | SSR |
| 标签的样式还是有点问题（受布局影响，被撑开了） | 样式布局 |
| 目前文档分散/没有文档：组件、工具方法（多语言方法、埋点方法、数值处理方法...）、SEO规范、SSR 规范 | - |
| 复杂的逻辑表述（需求是什么？使用什么工具？怎么更新工具和状态）前提是你已经知道有那些用代码表述的状态，如果一切皆黑盒的话，代码生成更不可控。 | - |
