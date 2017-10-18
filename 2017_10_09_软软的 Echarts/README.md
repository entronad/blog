在 web 开发中，特别是 dashboard 类型的SPA中，常常要求各组件都能对不同的屏幕分辨率进行缩放适配，常见的解决方案之一是在CSS中使用 rem 和 em 的长度单位结合媒体查询。

最近做到的一些数据可视化需求中，图表组件用到 Echarts，发现由于 Echarts 是基于绘制 Canvas 的方式生成图表，在配置项中，所有的距离和大小只能设置单位为 px，这就使得做屏幕适配十分不便，分辨率一变，不仅图表里的字体大小、距离与整体界面不协调，而且经常会发生字体重叠到一起。

后来想了一想， Echarts 每次重绘是在 JavaScript 中通过 setOption 方法传入配置的，可以利用 window 对象获取到窗口的尺寸，动态设置一个变量起到根元素 font-size 的作用，模拟出 rem 与媒体查询的效果。

首先为组件搞一个样式配置文件，或添加在已有的配置文件中：

```
// styleProps.js

const viewPort = window.screen.width;
let rootFontSize = 100;
if (viewPort <= 1480) rootFontSize = 71;
if (viewPort >= 1481 && viewPort <= 1760) rootFontSize = 83;
if (viewPort >= 2241 && viewPort <= 3200) rootFontSize = 130;
if (viewPort >= 3201) rootFontSize = 200;

export default {
  titleFontColor: 'rgba(255,255,255,0.97)',
  bodyFontColor: 'rgba(255,255,255,0.91)',
  titleFontSize: rootFontSize * 0.18,
  bodyFontSize: rootFontSize * 0.16,

  titleFontFamily: 'FangZhengZhengZhongHeiJT',
  bodyFontFamily: 'FangZhengZhengXianHeiJT',
};
```

为了解决 Chrome 浏览器下字体最小为 12px 的问题，习惯上会把根元素的 font-size 设为 100px ，这里也将 rootFontSize 设为 100 。

通过变量 viewPort 获取屏幕宽度，针对以下常见的屏幕分辨率，按区间进行适配：

```
1366*768
1600*900
1920*1080
2560*1400
3840*2160
```

取宽度的话，基本上全屏、投影拼接屏等情况也不会影响，当然了，因为是 dashboard 类型的的应用，就不考虑用户拉小窗体之类奇奇怪怪的情况了。

这个 styleProps.js 配置，与我们在整个应用的 CSS 中的设置是“同构”的：

```
html{
  font-size: 625%;
}
@media (max-width: 1480px) {
  html{
    font-size: 444%;
  }
}
@media (min-width: 1481px) and (max-width: 1760px) {
  html{
    font-size: 521%;
  }
}
@media (min-width: 2241px) and (max-width: 3200px) {
  html{
    font-size: 810%;
  }
}
@media (min-width: 3201px) {
  html{
    font-size: 1250%;
  }
}

body{
  font-size: 0.2rem;
  font-family: FangZhengZhengXianHeiJT;
  color: rgba(255,255,255,0.91);
}

h1 {
  font-size: 0.44rem;
  font-family: FangZhengZhengZhongHeiJT;
  color: rgba(255,255,255,0.97);
}
h2 {
  font-size: 0.32rem;
  font-family: FangZhengZhengZhongHeiJT;
  color: rgba(255,255,255,0.97);
}
h3 {
  font-size: 0.24rem;
  font-family: FangZhengZhengZhongHeiJT;
  color: rgba(255,255,255,0.97);
}
```

然后为曲线和柱状图表简单的封装了一个组件：

```
import React from 'react';
import echarts from 'echarts';
import styleProps from '../styleProps';

export default class HArrayChart extends React.Component {
  static defaultProps = {
    title: '',
    /**
     * dataItem {
     *  data: [],
     *  type: '',
     *  name: '',
     *  axis: 'left' | 'right',
     * }
     */
    dataList: [],
    /**
     * {
     *  name: '',
     *  data: []
     * }
     */
    index: {},
    /**
     * ['发电量(kWh)','功率(kW)']
     */
    axes: []
  }
  componentDidMount() {
    this.chart = echarts.init(this.chart);
    this.updateChart();
  }
  componentDidUpdate() {
    this.updateChart();
  }
  updateChart = () => {
    const getSerisOption = (type) => {
      switch (type) {
        case 'line':
          return {
            symbol: 'none'
          };
        default:
          break;
      }
      return null;
    };
    this.chart.setOption({
      title: {
        text: this.props.title,
        x: 'center',
        textStyle: {
          color: styleProps.titleFontColor,
          fontFamily: styleProps.titleFontFamily,
          fontSize: styleProps.titleFontSize
        }
      },
      grid: {
        top: (styleProps.bodyFontSize * 2.4) + styleProps.titleFontSize,
        right: (styleProps.bodyFontSize * 3.8) + 8,
        bottom: (styleProps.bodyFontSize * 4.2) + 8,
        left: (styleProps.bodyFontSize * 3.8) + 8
      },
      yAxis: this.props.axes.map(name => ({
        name,
        nameGap: styleProps.bodyFontSize * 0.8,
        type: 'value',
        nameTextStyle: {
          color: styleProps.bodyFontColor,
          fontFamily: styleProps.bodyFontFamily,
          fontSize: styleProps.bodyFontSize
        },
        axisLine: {
          lineStyle: {
            color: styleProps.bodyFontColor
          }
        },
        axisLabel: {
          color: styleProps.bodyFontColor,
          fontFamily: styleProps.bodyFontFamily,
          fontSize: styleProps.bodyFontSize
        }
      })),
      legend: {
        data: this.props.dataList.map(dataItem => dataItem.name),
        icon: 'bar',
        bottom: 0,
        textStyle: {
          color: styleProps.bodyFontColor,
          fontFamily: styleProps.bodyFontFamily,
          fontSize: styleProps.bodyFontSize
        }
      },
      xAxis: {
        data: this.props.index.data,
        type: 'category',
        name: this.props.index.name,
        nameLocation: 'center',
        nameGap: (styleProps.bodyFontSize * 1.2) + 8,
        nameTextStyle: {
          color: styleProps.bodyFontColor,
          fontFamily: styleProps.bodyFontFamily,
          fontSize: styleProps.bodyFontSize
        },
        axisLine: {
          lineStyle: {
            color: styleProps.bodyFontColor
          }
        },
        axisLabel: {
          color: styleProps.bodyFontColor,
          fontFamily: styleProps.bodyFontFamily,
          fontSize: styleProps.bodyFontSize
        }
      },
      series: this.props.dataList.map(dataItem => ({
        data: dataItem.data,
        type: dataItem.type || 'line',
        name: dataItem.name,
        animation: false,
        yAxisIndex: dataItem.axis === 'right' ? 1 : 0,
        ...getSerisOption(dataItem.type || 'line')
      }))
    });
  }
  render() {
    return <div className={this.props.className} ref={(div) => { this.chart = div; }} />;
  }
}
```

之前的 styleProps.js 配置文件通过 import 引入，相关的参数会在组件生成之前就计算好，这样足以应对屏幕分辨率的适配了，当然如果希望能够更动态的调整，也可以设置一个 state 监视视口宽度，在每次 componentWillUpdate 里重算参数。

为了方便不熟悉 Echarts 配置的使用者，这个组件只暴露了4个参数：

```
title: '',
dataList: [],
index: {},
axes: []
```

dataList 通过数组可传入多条数据 series ，每条可独立配置曲线或柱状类型，以及关联到左还是右坐标轴，每条数据 series 的长度需与横坐标的 index 长度对应。当然这样的配置使得自由度很小，可以给使用者也暴露个 option 配置项，覆盖默认的配置。

由于在styleProps.js 配置中对字体大小的设置，与我们在整个应用的 CSS 中的设置是“同构”的，所以通过 titleFontSize 和 bodyFontSize 就使得当自适应匹配屏幕分辨率变化时， Echarts 与整个界面的行为保持一致，字体大小、距离等变化协调。

图表中各个元素之间的距离，比如 Legend 和 xAxis 的距离、 grid 与四边的距离等，也用bodyFontSize的比例进行设置，这样当图表中字体变化后也不会发生相互覆盖的情况了：

```
yAxis: this.props.axes.map(name => ({
  nameGap: styleProps.bodyFontSize * 0.8,        
})),
```

由于项目中使用了 styled-components ，所以组件根元素要传入 className={this.props.className} 。

最后组件使用的时候大致是这个样子：

```
import styled from 'styled-components';
import { HArrayChart } from './hyphix';

const Chart = styled(HArrayChart)`
  width: 5rem;
  height: 3rem;
`;
const dataList = [
  {
    data: [2, 4, 5, 80, 22, 27, 39, 0, 12],
    name: '今日功率',
  },
  {
    data: [32, 33, 3, 8, 2, 27, 3, 3, 15],
    name: '昨日功率',
    type: 'bar'
  },
  {
    data: [837, 3099, 122, 3999, 100, 299, 3987, 3298, 1500],
    name: '昨日效率',
    axis: 'right'
  },
];
const index = { data: [1, 2, 3, 4, 5, 6, 7, 8, 9], name: '时间' };const Chart = styled(HArrayChart)`
  width: 500px;
  height: 300px;
`;
const dataList = [
  {
    data: [2, 4, 5, 80, 22, 27, 39, 0, 12],
    name: '今日功率',
  },
  {
    data: [32, 33, 3, 8, 2, 27, 3, 3, 15],
    name: '昨日功率',
    type: 'bar'
  },
  {
    data: [837, 3099, 122, 3999, 100, 299, 3987, 3298, 1500],
    name: '昨日效率',
    axis: 'right'
  },
];
const index = { data: [1, 2, 3, 4, 5, 6, 7, 8, 9], name: '时间' };

<Route path="/test" exact component={() => <Chart title={'今日发电功率曲线'} dataList={dataList} index={index} axes={['发电量(kWh)', '功率(kW)']} />} />
```

由于没有好看的数据，图就不上了，组件只是个大致的思路，要做的好看细节上还需要再完善。

