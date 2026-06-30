# 天气可视化仪表盘 Axios + ECharts5.x
## 项目简介
本项目基于原生JS、Axios、ECharts5、Open-Meteo免费气象API实现多图表联动天气仪表盘，完成链式异步请求、响应式布局、全局Loading、异常捕获等全部任务要求。

## 技术栈
- Axios：处理HTTP链式异步请求，全局拦截统一管理Loading状态
- ECharts 5.x：折线图、柱状图、雷达风向玫瑰图，多图表connect联动
- Open-Meteo API：免费地理编码+7天逐时气象数据，无需密钥
- 原生CSS Grid：响应式布局，适配手机/平板/桌面端

## 核心功能（全部满足任务硬性指标）
1. Axios链式异步流程
   - async/await 串行调用【城市→经纬度】→【经纬度→气象数据】接口
   - axios拦截器统一显示/隐藏Loading遮罩
   - try/catch捕获网络错误、城市不存在等异常，友好提示
2. ECharts多图表联动
   - 温度折线图、降雨量柱状图、风向玫瑰雷达图三种图表
   - echarts.connect()实现多图共享dataZoom，hover同步高亮
3. 响应式仪表盘
   - CSS Grid自动分栏，移动端单列、桌面双列
   - window.resize监听自动重绘图表
4. 完整交互
   - 输入城市回车/按钮查询、切换城市实时刷新图表
   - 网络/参数错误统一提示框


## 运行方式
1. 将index.html直接使用浏览器打开（无需后端，纯前端项目）
2. 默认加载北京7天天气，输入任意国内城市即可切换数据
3. 缩放浏览器窗口，图表自动适配尺寸

## 文件说明
- index.html：整合HTML结构、CSS样式、全部JS业务逻辑
- README.md：项目说明文档（本文件）
<!DOCTYPE html>
<html lang="zh-CN">
<head>
  <meta charset="UTF-8" />
  <!-- 响应式适配核心标签 -->
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>天气可视化仪表盘</title>
  <!-- CDN引入Axios & ECharts 5.x -->
   <!-- 提供异步网络请求能力 -->
  <script src="https://cdn.jsdelivr.net/npm/axios/dist/axios.min.js"></script>
   <!-- 图表绘图核心库，多图联动、缩放 -->
  <script src="https://cdn.jsdelivr.net/npm/echarts@5.4.3/dist/echarts.min.js"></script>
  <style>
    /* 全局样式 */
    * {
      margin: 0;
      padding: 0;
      box-sizing: border-box;
      font-family: "Microsoft YaHei", sans-serif;
    }
    body {
      background-color: #f5f7fa;
      padding: 16px;
    }
    .container {
      max-width: 1400px;
      margin: 0 auto;
    }
    /* 搜索栏样式 */
    .search-bar {
      display: flex;
      gap: 10px;
      margin-bottom: 20px;
      align-items: center;
    }
    #cityInput {
      flex: 1;
      max-width: 400px;
      padding: 10px 14px;
      border: 1px solid #ddd;
      border-radius: 6px;
      font-size: 16px;
    }
    #searchBtn {
      padding: 10px 24px;
      background: #1677ff;
      color: white;
      border: none;
      border-radius: 6px;
      cursor: pointer;
      font-size: 16px;
    }
    #searchBtn:disabled {
      background: #8cb8f7;
      cursor: not-allowed;
    }
    .loading {
      color: #1677ff;
      font-size: 16px;
      margin-left: 12px;
      display: none;
    }
    .error-tip {
      color: #f5222d;
      margin: 8px 0 16px;
      display: none;
    }
    /* CSS Grid 响应式图表布局 */
    .chart-wrap {
      display: grid;
      grid-template-columns: 1fr 1fr;
      grid-template-rows: auto auto;
      gap: 16px;
    }
    .chart-item {
      background: #fff;
      border-radius: 10px;
      padding: 12px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.06);
      min-height: 420px;
    }
    /* 风向玫瑰图单独占整行 */
    .chart-wind {
      grid-column: 1 / -1;
    }
    /* 移动端适配 */
    @media screen and (max-width: 768px) {
      .chart-wrap {
        grid-template-columns: 1fr;
      }
      .search-bar {
        flex-wrap: wrap;
      }
      #cityInput {
        width: 100%;
        max-width: unset;
      }
    }
  </style>
</head>
<body>
<div class="container">
  <!-- 搜索区域 -->
  <div class="search-bar">
    <input id="cityInput" placeholder="请输入城市名称，例如：北京、上海" value="北京"/>
    <button id="searchBtn">查询天气</button>
    <span class="loading" id="loadingTip">加载中，请稍候...</span>
  </div>
  <div class="error-tip" id="errorTip"></div>

  <!-- 图表容器 -->
  <div class="chart-wrap">
    <div class="chart-item">
      <div id="tempChart" style="width:100%;height:100%;"></div>
    </div>
    <div class="chart-item">
      <div id="rainChart" style="width:100%;height:100%;"></div>
    </div>
    <div class="chart-item chart-wind">
      <div id="windChart" style="width:100%;height:100%;"></div>
    </div>
  </div>
</div>

<script>
  //全局变量
  let tempChart, rainChart, windChart;
  const loadingTip = document.getElementById('loadingTip');
  const errorTip = document.getElementById('errorTip');
  const searchBtn = document.getElementById('searchBtn');
  const cityInput = document.getElementById('cityInput');

  //Axios 请求拦截器：统一处理 Loading
  // 注册请求拦截
  axios.interceptors.request.use(config => {
  // 显示页面加载文字
    loadingTip.style.display = 'inline';
  // 查询按钮禁用
    searchBtn.disabled = true;
  // 隐藏错误提示，清空旧报错
    errorTip.style.display = 'none';
  // 请求全部配置
    return config;
  });
  // 成功回调
  axios.interceptors.response.use(res => {
  // 隐藏加载文字，加载结束
    loadingTip.style.display = 'none';
  // 解锁查询按钮，允许用户再次点击查询
    searchBtn.disabled = false;
  // 返回完整响应对象传递给代码
    return res;
  // 回调失败
  }, err => {
  // 请求失败，关闭加载提示
    loadingTip.style.display = 'none';
  // 解锁按钮，重新查询
    searchBtn.disabled = false;
  // 网络异常提示
    showError('网络请求失败，请检查网络后重试');
  // 将错误抛出，捕获错误
    return Promise.reject(err);
  });

  //工具函数
  /**
   * 展示错误提示
   * @param {String} msg 错误文案
   */
  function showError(msg) {
    errorTip.innerText = msg;
    errorTip.style.display = 'block';
  }

  /**
   * 初始化三个ECharts实例
   */
  function initCharts() {
    tempChart = echarts.init(document.getElementById('tempChart'));
    rainChart = echarts.init(document.getElementById('rainChart'));
    windChart = echarts.init(document.getElementById('windChart'));
    // 多图表联动：connect共享dataZoom与hover交互
    echarts.connect([tempChart, rainChart, windChart]);
  }

  /**
   * 窗口自适应，图表重绘
   */
  function bindResize() {
    window.addEventListener('resize', () => {
      tempChart.resize();
      rainChart.resize();
      windChart.resize();
    })
  }

  /**
   * 将0-360°风向转换为16方位名称，统计频次（风向玫瑰图数据处理）
   * @param {Number[]} windDirList 原始风向数组
   * @returns {Object} 方位-频次映射
   */
  function calcWindRoseData(windDirList) {
    // 16个风向方位
    const dirNames = ['北','北东北','东北','东东北','东','东东南','东南','南东南','南','南西南','西南','西西南','西','西西北','西北','北西北'];
    const dirCount = new Array(16).fill(0);
    const unit = 22.5; // 360/16，每个方位区间22.5度
    // 统计出现次数
    windDirList.forEach(deg => {
      let idx = Math.floor((deg + unit / 2) / unit) % 16;
      dirCount[idx]++;
    })
    // 组装ECharts需要的数据格式
    return dirNames.map((name, i) => ({ name, value: dirCount[i] }))
  }

  //Axios 链式请求核心逻辑
  /**
   * 链式请求：城市名→经纬度→7天小时级天气数据
   * @param {String} cityName 城市名称
   */
  async function fetchWeatherData(cityName) {
    try {
      // 第一步：调用地理编码API，获取城市经纬度
      const geoRes = await axios.get('https://geocoding-api.open-meteo.com/v1/search', {
        params: {
          name: cityName,
          count: 1,
          language: 'zh',
          format: 'json'
        }
      })
      // 无返回城市数据
      const geoList = geoRes.data.results;
      if (!geoList || geoList.length === 0) {
        showError(`未找到城市【${cityName}】，请更换名称重试`);
        return;
      }
      const { latitude, longitude } = geoList[0];

      // 第二步：根据经纬度请求气象API
      const weatherRes = await axios.get('https://api.open-meteo.com/v1/forecast', {
        params: {
          latitude,
          longitude,
          hourly: 'temperature_2m,precipitation,wind_speed_10m,wind_direction_10m',
          timezone: 'auto',
          forecast_days: 7
        }
      })
      const hourlyData = weatherRes.data.hourly;
      renderAllCharts(hourlyData);
    } catch (err) {
      console.error('请求异常：', err);
    }
  }

  //ECharts 渲染函数
  /**
   * 渲染全部三张图表
   * @param {Object} data 原始hourly气象数据
   */
  function renderAllCharts(data) {
    const { time, temperature_2m, precipitation, wind_direction_10m } = data;
    // 1. 温度折线图
    tempChart.setOption({
      title: { text: '7天逐小时温度变化(℃)' },
      tooltip: { trigger: 'axis' },
      dataZoom: [{ type: 'slider', xAxisIndex: 0 }],
      xAxis: { type: 'category', data: time },
      yAxis: { name: '温度 ℃' },
      series: [{
        name: '气温',
        type: 'line',
        data: temperature_2m,
        smooth: true,
        itemStyle: { color: '#f5a623' }
      }]
    })

    // 2. 降雨量柱状图
    rainChart.setOption({
      title: { text: '每小时降雨量(mm)' },
      tooltip: { trigger: 'axis' },
      dataZoom: [{ type: 'slider', xAxisIndex: 0 }],
      xAxis: { type: 'category', data: time },
      yAxis: { name: '降雨量 mm' },
      series: [{
        name: '降雨',
        type: 'bar',
        data: precipitation,
        itemStyle: { color: '#1890ff' }
      }]
    })

    // 3. 风向玫瑰图（极坐标）
    const windRoseData = calcWindRoseData(wind_direction_10m);
    windChart.setOption({
      title: { text: '风向频率玫瑰图(16方位)' },
      tooltip: { trigger: 'item' },
      polar: {},
      radiusAxis: {},
      angleAxis: { type: 'category', data: windRoseData.map(d => d.name) },
      series: [{
        type: 'bar',
        data: windRoseData,
        coordinateSystem: 'polar',
        barWidth: '40%',
        itemStyle: { color: '#52c41a' }
      }]
    })
  }

  //页面初始化与事件绑定
  function init() {
    initCharts();
    bindResize();
    // 默认加载北京数据
    fetchWeatherData('北京');
    // 按钮点击事件
    searchBtn.addEventListener('click', () => {
      const city = cityInput.value.trim();
      if (!city) {
        showError('请输入有效城市名称');
        return;
      }
      fetchWeatherData(city);
    })
    // 回车快捷查询
    cityInput.addEventListener('keydown', e => {
      if (e.key === 'Enter') searchBtn.click();
    })
  }
  // 页面加载完成初始化
  window.onload = init;
</script>
</body>
</html>
